# Типизированная иерархия `OhttpException` и согласование multi-suite в KeyConfig

**Оценка:** 3 дня
**Приоритет:** P2: High
**Компонент:** App
**Серьёзность:** HIGH

## Описание задачи

После реструктуризации L-01 в пакете появился один новый тип исключения (`OhttpGatewayException`, определённый в `lib/src/transport.dart`), а во всём остальном условия ошибок по-прежнему всплывают через голые `Exception`, `FormatException`, `UnsupportedError` и `SecretBoxAuthenticationError` из `package:cryptography`. Эта задача приземляет типизированную иерархию `OhttpException`, которая поглощает существующий `OhttpGatewayException`, исправляет сломанный парсер KeyConfig multi-suite в `ohttp.dart` и унифицирует два несвязанных типа исключений, которые текущий парсер бросает для одного и того же концептуального условия:

1. **Нет типизированной иерархии исключений.** `http`-адаптер `HttpClientTransport.fetchKeyConfig` / `postToGateway` (в `lib/src/adapters/http/http_transport.dart`) бросает `OhttpGatewayException` с кодом статуса на non-200 ответы — хорошо — но каждый другой путь ошибок не типизирован:
   - Сетевые ошибки (отказ DNS, сброс соединения) всплывают как исключения `package:http` / `dart:io`.
   - Сбои аутентификации AEAD пробрасываются как `SecretBoxAuthenticationError` из `package:cryptography` (через `aesGcm.decrypt(...)` внутри `ohttpDecapsulate` в `lib/src/ohttp.dart`), делая транзитивную зависимость частью публичного контракта.
   - Сбои парсинга BHTTP/OHTTP бросают `FormatException` (хорошо, но не под библиотечной базой).
   - `OhttpGatewayException` (уже в ядре) не наследует общую библиотечную базу, поэтому потребители не могут написать единое `on OhttpException` для покрытия всех ошибок, брошенных библиотекой.

2. **Согласование multi-suite в KeyConfig сломано.** RFC 9458 §4.1 позволяет шлюзу анонсировать несколько пар KDF+AEAD в секции симметричных алгоритмов (`symLen > 4`). `OhttpKeyConfig.parse()` читает `symLen`, делает bounds-check, затем потребляет ровно одну 4-байтовую пару (`kdfId`, `aeadId`) и возвращается, молча отбрасывая любые завершающие пары. Шлюз, который сначала анонсирует неподдерживаемый suite, а затем поддерживаемый, заставит парсер залочиться на первой паре и провалить валидацию — или молча выбрать suite, который библиотека не реализует.

3. **Сопутствующее рассогласование в ошибках о неподдерживаемом suite.** `parse()` бросает `FormatException('Unsupported KEM: ...')`, а `validate()` бросает `UnsupportedError` для того же концептуального условия. Две несвязанные иерархии исключений для одного и того же класса ошибок заставляют вызывающую сторону писать два `catch` блока или скатываться к широкому `catch (e)`.

Выполнение их одновременно избегает повторной перетряски поверхности исключений: исправление multi-suite вводит новые места `throw`, которые должны уже использовать типизированную иерархию.

## Технические детали

### Часть A — Типизированная иерархия исключений

**1. Определить базу + подклассы**

В `lib/src/exceptions.dart` (новый файл) определить:

- `abstract class OhttpException implements Exception` — общая база, раскрывает `message`.
- `class OhttpHttpException extends OhttpException` — non-2xx ответ от relay или шлюза. Несёт `statusCode` и опциональный фрагмент `body`. **Это тип, который бросает `HttpClientTransport`** (заменяет существующий `OhttpGatewayException` — см. §2 ниже).
- `class OhttpTimeoutException extends OhttpException` — таймаут запроса. Используется работой по таймаутам в L-04.
- `class OhttpParseException extends OhttpException` — оборачивает сбои парсинга BHTTP/OHTTP. Адаптер для `FormatException` на границах библиотеки там, где это уместно; сырой `FormatException` из `bhttp.dart` всё ещё может всплывать из `serializeRequest` / `parseResponse` (они документируют `FormatException` напрямую).
- `class OhttpKeyConfigException extends OhttpException` — специфичные для KeyConfig сбои парсинга (некорректный `symLen`, неизвестная структура).
- `class OhttpDecryptionException extends OhttpException` — оборачивает `SecretBoxAuthenticationError`, брошенный `aesGcm.decrypt(...)` внутри `ohttpDecapsulate`. Хранит исходную причину без утечки имени типа из пакета cryptography в публичный API.
- `class OhttpUnsupportedSuiteException extends OhttpException` — неподдерживаемый KEM/KDF/AEAD. Заменяет `FormatException('Unsupported KEM: ...')` из `parse()` и `UnsupportedError` из `validate()`.
- `class OhttpConfigException extends OhttpException` — сбои валидации входов из L-02 (HTTPS-схема, форма authority). Если L-02 приземлится первой с placeholder-типом, свернуть его в эту иерархию здесь.

**2. Поглотить `OhttpGatewayException`**

Реструктуризация L-01 определяет `OhttpGatewayException` в `lib/src/transport.dart` с `statusCode` и `message`. Переименовать его в `OhttpHttpException` и переместить в иерархию как подкласс `OhttpException`. Контракт интерфейса транспорта в `OhttpTransport.postToGateway` обновляется соответственно — реализации ДОЛЖНЫ бросать `OhttpHttpException` на non-2xx, чтобы `OhttpSession` мог инвалидировать кэш.

Обновить `catch`-блок в `OhttpSession.send`: `on OhttpHttpException` (вместо `on OhttpGatewayException`). Проверить через `ast-index usages "OhttpGatewayException"`, что не осталось устаревших ссылок.

**3. Перенацелить места `throw`**

- `lib/src/adapters/http/http_transport.dart`: `OhttpGatewayException(...)` → `OhttpHttpException(...)` как в `fetchKeyConfig` (non-200 на GET KeyConfig), так и в `postToGateway` (non-200 на POST шлюза).
- Место AEAD-расшифровки в `lib/src/ohttp.dart` внутри `ohttpDecapsulate`: оборачивать `SecretBoxAuthenticationError` и перебрасывать как `OhttpDecryptionException`, сохраняя причину.
- `lib/src/ohttp.dart` `OhttpKeyConfig.parse` и `OhttpKeyConfig.validate` (пути неподдерживаемого KEM/KDF/AEAD): из обоих бросать `OhttpUnsupportedSuiteException` — один и тот же тип, независимо от того, какой code path обнаруживает неподдерживаемое значение.
- `lib/src/ohttp.dart` `OhttpKeyConfig.parse` (некорректный wire-формат): бросать `OhttpKeyConfigException`.

**4. Экспортировать иерархию**

Добавить `export 'src/exceptions.dart';` в `lib/ohttp_dart.dart`. Удалить теперь неиспользуемый экспорт `OhttpGatewayException`, если он сохранился под старым именем.

Обновить doc-комментарии на `OhttpSession.send`, `OhttpTransport.postToGateway`, `OhttpTransport.fetchKeyConfig`, `ohttpDecapsulate`, `OhttpKeyConfig.parse` и `OhttpKeyConfig.validate`, перечисляя исключения, которые каждый из них может бросить.

### Часть B — Парсинг multi-suite в KeyConfig

**5. Переработать обработку нескольких пар в `OhttpKeyConfig.parse`** (`lib/src/ohttp.dart`)

Инспектировать каждую пару KDF+AEAD в секции симметричных алгоритмов. Выбрать одну из двух политик и задокументировать её в doc-комментарии функции со ссылкой на RFC 9458 §4.1:

(a) **Iterate-and-select.** Пройти по всем парам `(kdfId, aeadId)` в порядке `symLen / 4`; выбрать первую пару, совпадающую с единственным поддерживаемым suite библиотеки (`kdfId == 0x0001`, `aeadId == 0x0001`). Если ни одна не совпадает, бросить `OhttpUnsupportedSuiteException`, перечислив анонсированные пары.

(b) **Strict-single-pair.** Требовать `symLen == 4` и ровно одну поддерживаемую пару. Бросать `OhttpUnsupportedSuiteException`, когда wire-формат подразумевает `symLen > 4`.

Выбрать (a), если нет конкретной причины отвергать multi-pair конфиги — (a) дружелюбнее к шлюзам, которые анонсируют legacy suites рядом с поддерживаемым.

**6. Отвергать некорректный `symLen`** (`lib/src/ohttp.dart`)

`symLen`, не кратный 4, указывает на некорректный wire-формат. Отвергнуть с `OhttpKeyConfigException` (типизированный), несущим оскорбительную длину.

### Часть C — Тесты

**7. Тесты иерархии** (`test/adapters/http_adapter_test.dart`, `test/ohttp_test.dart`)

- `MockClient` возвращает 4xx для `keysUrl` → `HttpClientTransport.fetchKeyConfig` бросает `OhttpHttpException` с `statusCode` в диапазоне 4xx.
- `MockClient` возвращает 5xx для `gatewayUrl` → `HttpClientTransport.postToGateway` бросает `OhttpHttpException` с `statusCode` в диапазоне 5xx.
- То же исключение ловится как `on OhttpException`.
- Произвести корректный OHTTP-encap, вывести response-секрет в тесте, зашифровать ответ, перевернуть один байт в шифротексте, вызвать `ohttpDecapsulate`, утверждать, что брошен `OhttpDecryptionException` (а не базовый `SecretBoxAuthenticationError`). Утверждать, что поле причины несёт исходную ошибку.
- `OhttpSession.send` инвалидирует кэш, когда транспорт бросает `OhttpHttpException` (по-прежнему работает — `catch` теперь на `OhttpHttpException`, а не на `OhttpGatewayException`).

**8. Тесты multi-suite** (`test/ohttp_test.dart`)

Сконструировать синтетические байтовые буферы KeyConfig в тестовых fixture (задокументированные байтовые массивы, чтобы wire-формат был явным). Покрыть:

- Одна поддерживаемая пара (текущий happy-path).
- Две пары, где поддерживаемая на индексе 0 — `parse` успешно возвращает с выбранной поддерживаемой парой.
- Две пары, где поддерживаемая на индексе 1 — `parse` успешно возвращает (сейчас это падает).
- Две или три пары, где нет поддерживаемого suite — бросается `OhttpUnsupportedSuiteException` с перечислением анонсированных пар.
- Некорректный `symLen`, не кратный 4 — `OhttpKeyConfigException`.
- Неизвестный KEM через `parse()` — `OhttpUnsupportedSuiteException`.
- Неизвестный KEM/KDF/AEAD через `validate()` — `OhttpUnsupportedSuiteException` (тот же тип, что и в пути `parse()`).

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Критерии приёмки

**Иерархия:**
- `lib/src/exceptions.dart` определяет `OhttpException` плюс как минимум подклассы, перечисленные в §1. Все реэкспортируются из `lib/ohttp_dart.dart`.
- `OhttpGatewayException` из L-01 переименован в `OhttpHttpException` и приведён под `OhttpException`. Doc-комментарий `OhttpTransport.postToGateway` обновлён, чтобы отражать новый контракт.
- `HttpClientTransport.fetchKeyConfig` и `HttpClientTransport.postToGateway` бросают `OhttpHttpException` (несущий `statusCode`) на non-200.
- `ohttpDecapsulate` больше не отдаёт `SecretBoxAuthenticationError`; отдаёт `OhttpDecryptionException`, оборачивающий исходную причину.
- `OhttpKeyConfig.parse` и `OhttpKeyConfig.validate` оба бросают `OhttpUnsupportedSuiteException` для условия неподдерживаемого suite.
- `OhttpKeyConfig.parse` бросает `OhttpKeyConfigException` для некорректного `symLen`.
- Doc-комментарии на `OhttpSession.send`, `OhttpTransport.fetchKeyConfig` / `postToGateway`, `ohttpDecapsulate`, `OhttpKeyConfig.parse` / `validate` перечисляют исключения, которые они могут бросать.

**Multi-suite:**
- `OhttpKeyConfig.parse` больше не игнорирует завершающие байты при `symLen > 4`; итерирует по всем парам `(kdfId, aeadId)` и выбирает поддерживаемую либо бросает `OhttpUnsupportedSuiteException` согласно задокументированной политике.
- Doc-комментарий на `parse()` ссылается на RFC 9458 §4.1 и документирует выбранную политику согласования.
- Некорректный `symLen` (не кратный 4) отвергается с `OhttpKeyConfigException`.

**Тесты:**
- Иерархия: 4xx → `OhttpHttpException`(4xx); 5xx → `OhttpHttpException`(5xx); оба ловятся как `on OhttpException`; сбой аутентификации AEAD → `OhttpDecryptionException`; инвалидация кэша в `OhttpSession.send` по-прежнему срабатывает на `OhttpHttpException`.
- Multi-suite: одна пара, поддерживаемая на индексе 0, поддерживаемая на индексе 1, нет поддерживаемой пары (типизированное исключение), некорректный `symLen` (типизированное исключение), неподдерживаемый KEM через оба `parse()` и `validate()`.

## Дополнительно

- Тесты: unit-тесты для каждого пути типизированного исключения и каждого случая wire-формата multi-suite из перечисленных выше.
- Зависит от L-01 (типы `OhttpTransport`, `OhttpGatewayException`, `OhttpSession`, `HttpClientTransport` должны существовать до того, как их можно будет переселить под новую иерархию).
- Миграция: внешние потребители, ловящие `OhttpGatewayException` из L-01, должны обновиться на `OhttpHttpException`. Поскольку у этой реструктуризации пока нет выпущенных потребителей (единственный `publish_to` пакета — `none`), переименование — это free move.
- Эта задача должна приземлиться второй (после L-01), чтобы последующие задачи (L-04, L-05, L-07) с первого дня эмитили свои новые `throw` на типизированную поверхность.
- Исходные merged-задачи (в `../merged-tasks/`): M-05 (типизированная иерархия исключений), M-04 (multi-suite KeyConfig + унификация ошибок о неподдерживаемом suite). Местоположения мест `throw` из M-05 (изначально `ohttp_client.dart:84-87, 120-122`) перенацелены на `HttpClientTransport`, как задокументировано выше.
