# Реструктуризация пакета до архитектуры, агностичной к HTTP-клиенту (ядро + опциональный `http`-адаптер)

**Оценка:** 5 дней
**Приоритет:** P1: Critical
**Компонент:** App
**Серьёзность:** BLOCKER

## Описание задачи

Сегодня `ohttp_dart` поставляет единственный высокоуровневый `OhttpClient`, который владеет собственной обвязкой `package:http` и возвращает кастомную форму `OhttpResponse`/`OhttpHeader`. Ветке `ohttp_integration` кошелька не нужно было ни то, ни другое: она полностью обходила `OhttpClient` и переоркестрировала примитивы (`serializeRequest` → `ohttpEncapsulate` → POST → `ohttpDecapsulate` → `parseResponse`) внутри ручного наследника `http.BaseClient`, плюс ручной кэш KeyConfig (~155 строк склейки). Причина: форма ответа `OhttpClient` не подходит под контракт `http.StreamedResponse` метода `http.BaseClient.send`, и он владеет своей обвязкой HTTP к шлюзу вместо того, чтобы дать потребителю подключить свою.

Конкретные трения сегодня:

1. `OhttpClient` фактически мёртвый груз для любой реальной интеграции — каждый потребитель его обходит.
2. `ohttp_dart` жёстко зависит от `package:http` даже для потребителей, которые будут использовать Dio или другой клиент.
3. Пакет не поставляет кэш KeyConfig, поэтому каждый потребитель переизобретает его (а в ручном кэше кошелька есть race в single-flight после истечения TTL).
4. Кастомные типы `OhttpResponse` / `OhttpHeader` добавляют слой трансляции на каждой границе потребителя.

Эта задача реструктурирует пакет в тонкое ядро без зависимостей от HTTP-клиента плюс опциональный адаптер `package:http`, так что кошелёк (или любой другой потребитель) интегрируется через ~6 строк проводки в конструкторе вместо ~155 строк кастомного wrapper-кода, и так чтобы будущий Dio-адаптер можно было добавить, не трогая ядро.

Это BLOCKER для каждого другого follow-up: раскладка файлов, публичный API и поверхности exception/cache/observability, которые модифицируют L-02…L-07, существуют только после выполнения этой задачи.

## Технические детали

### Новая архитектура (две библиотеки в одном пакете)

```
package:ohttp_dart/ohttp_dart.dart   ← ядро (без зависимости от HTTP-клиента)
package:ohttp_dart/http.dart         ← опциональный адаптер package:http
```

Будущий Dio-адаптер жил бы в `package:ohttp_dart/dio.dart` поверх того же ядра, без изменений в типах ядра.

### Новая раскладка файлов

```
lib/
├── ohttp_dart.dart          (точка входа core-библиотеки)
├── http.dart                (точка входа http-адаптера)
└── src/
    ├── bhttp.dart                  (без изменений — RFC 9292)
    ├── hpke.dart                   (без изменений — RFC 9180 sender)
    ├── ohttp.dart                  (без изменений — RFC 9458 encap/decap)
    ├── transport.dart              (новый)
    ├── key_config_cache.dart       (новый)
    ├── ohttp_session.dart          (новый)
    └── adapters/
        └── http/
            ├── http_transport.dart      (новый)
            └── ohttp_http_client.dart   (новый)
```

**Удалено:** `lib/src/ohttp_client.dart` полностью. Типы, которые он определял (`OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig`), удаляются без цикла deprecation — у них нет внешних потребителей, и они вытеснены новой поверхностью ниже. Никаких shim'ов обратной совместимости.

### Новая поверхность публичного API

**Core (`package:ohttp_dart/ohttp_dart.dart`):**

- `abstract interface class OhttpTransport` — шов bytes-in / bytes-out между оркестрацией OHTTP и любым HTTP-клиентом. Два метода: `Future<Uint8List> fetchKeyConfig()` и `Future<Uint8List> postToGateway(Uint8List body)`. Имплементации ОБЯЗАНЫ бросать `OhttpGatewayException` на non-2xx, чтобы сессия могла инвалидировать кэш.
- `class OhttpGatewayException implements Exception` — шлюз вернул non-2xx ответ. Несёт `statusCode` и сообщение. Живёт в ядре (а не в адаптере), потому что политика инвалидации, реагирующая на него, живёт в `OhttpSession`.
- `class KeyConfigCache` — TTL-кэш распарсенного `OhttpKeyConfig` шлюза с single-flight дедупликацией одновременных fetch'ей и ручным `invalidate()`. Инжектируемые часы для детерминированных TTL-тестов. TTL по умолчанию: 1 час.
- `class OhttpSession` — оркестратор, заменяющий сегодняшний `OhttpClient`. Владеет `(transport, cache)`. На каждый вызов: cache.get() → BHTTP-сериализация внутреннего запроса → OHTTP-encapsulate → transport.postToGateway → OHTTP-decapsulate → BHTTP-parse. При `OhttpGatewayException` от транспорта инвалидирует кэш перед re-throw. Удобный `OhttpSession.withTransport(...)` строит кэш за вас.
- `class OhttpRequestData` — форма запроса, нейтральная к HTTP-клиенту (`method`, `scheme`, `authority`, `path`, `headers`, `body`). `authority` — это **внутренний целевой хост**, на который шлюз форвардит — НЕ сам шлюз; URL шлюза — это забота транспорта.
- `class OhttpResponseData` — форма ответа, нейтральная к HTTP-клиенту. Заголовки как `List<(String, String)>`, чтобы сохранить порядок и дубликаты (например, `Set-Cookie`); схлопывание в `Map` происходит только в адаптерах, которые этого требуют.
- Существующие примитивы (модули `bhttp`, `hpke`, `ohttp`) реэкспортируются как сегодня.

**`http`-адаптер (`package:ohttp_dart/http.dart`):**

- `class HttpClientTransport implements OhttpTransport` — оборачивает `http.Client` плюс `Uri keysUrl` и `Uri gatewayUrl`. Ставит `Content-Type: message/ohttp-req` на POST. Не владеет жизненным циклом клиента.
- `class OhttpHttpClient extends http.BaseClient` — drop-in замена `http.Client`, которая маршрутизирует через `OhttpSession`. Извлекает method/scheme/authority/path/body из `request.url` и инжектит заголовок `host` (который `dart:io` обычно добавил бы на транспортном слое, но внутренний BHTTP-запрос никогда не достигает этого слоя, потому что он зашифрован). Опциональный параметр `closeWith` управляет тем, пропагирует ли `close()` на сырой клиент (по умолчанию выключено — предполагается, что сырым клиентом владеет DI).

### Удалённый API (без shim'ов)

- `lib/src/ohttp_client.dart` (весь файл).
- `OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig`.
- `OhttpClient.sendDirect()` и его getter `effectiveDirectBaseUrl` — обход, который поднимала L-01 (старая), не имеет смысла в новой архитектуре: потребитель, желающий plaintext HTTP, просто использует сырой `http.Client` и пропускает конструирование `OhttpHttpClient`. Библиотека больше не выставляет метод, чьё имя предполагает «use OHTTP», но молча отправляет plaintext.
- Строковый callback `onLog` из `OhttpClient` — заменён типизированной поверхностью observer, вводимой в L-05.

### Pubspec

- `cryptography ^2.9.0` — без изменений.
- `http ^1.6.0` — **остаётся в `dependencies`**. Core-библиотека не делает `import 'package:http/...'` (верифицируется раскладкой выше); только `lib/http.dart` и файлы под `lib/src/adapters/http/` делают. Потребители, которым нужно только ядро, платят одной неиспользуемой транзитивной зависимостью. Разделение на два пакета или перевод `http` в opt-in workflow (`dependency_overrides`) отложено — прагматически цена мала.
- Никаких новых зависимостей.

### Миграция кошелька (иллюстративно — не часть работы по этому пакету)

Ветка `ohttp_integration` кошелька заменяет ~155 строк под `lib/data/api/ohttp/` проводкой в конструкторе в своём DI-модуле:

```dart
final raw = http.Client();
final transport = HttpClientTransport(
  client: raw,
  keysUrl: Uri.parse(keysUrl),
  gatewayUrl: Uri.parse(gatewayUrl),
);
final session = OhttpSession.withTransport(transport: transport);
final ohttpClient = OhttpHttpClient(session: session, closeWith: raw);
```

Downstream-код кошелька продолжает использовать интерфейсы `http.Client` без изменений.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Критерии приёмки

**Структура библиотеки:**

- Существуют две точки входа библиотек: `lib/ohttp_dart.dart` (ядро) и `lib/http.dart` (адаптер `http`).
- Ни один файл под `lib/src/` вне `lib/src/adapters/http/` не импортирует `package:http`. Верифицируется инспекцией `import`-строк после реструктуризации.
- `lib/src/ohttp_client.dart` больше не существует; `OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig` не экспортируются ни из одной библиотеки.

**Публичный API:**

- `OhttpTransport`, `OhttpGatewayException`, `KeyConfigCache`, `OhttpSession`, `OhttpRequestData`, `OhttpResponseData` экспортируются из `lib/ohttp_dart.dart`.
- `HttpClientTransport` и `OhttpHttpClient` экспортируются из `lib/http.dart`.
- Удобный конструктор `OhttpSession.withTransport(...)` строит дефолтный `KeyConfigCache` автоматически.
- `KeyConfigCache.get()` разделяет один in-flight fetch между параллельными stale-вызовами (single-flight дедупликация).
- `KeyConfigCache.invalidate()` форсирует re-fetch на следующем `get()`.
- `OhttpSession.send` инвалидирует кэш, когда транспорт бросает `OhttpGatewayException`, и **не** инвалидирует ни на каком другом классе исключений (сетевые ошибки, сбои decap и т. п.).

**Поведение сохранено:**

- Полный round trip RFC 9180 / 9292 / 9458 продолжает проходить существующие тестовые файлы `test/hpke_test.dart`, `test/bhttp_test.dart` и `test/ohttp_test.dart` без модификации (эти тестовые файлы не связаны с поверхностью публичного клиента).
- `OhttpHttpClient` соблюдает инъекцию заголовка `host`: материализует заголовок из `request.url`, если вызывающий не предоставил его, и сохраняет предоставленный вызывающим заголовок `host` дословно. Дефолтные порты (80/443) исключаются из сконструированного `authority`.

**Пример + документация:**

- `example/ohttp_dart_example.dart` демонстрирует два пути в этом порядке: (1) quick-start `OhttpHttpClient` (то, что скопирует большинство потребителей), (2) более низкоуровневый путь `OhttpSession.send` (то, во что вызывал бы Dio/кастомный адаптер).
- Секции «Project Structure» и «Usage» в `README.md` отражают новую раскладку и quick-start `OhttpHttpClient`.
- Секция «Layering» в `CLAUDE.md` перечисляет новые файлы (`transport.dart`, `key_config_cache.dart`, `ohttp_session.dart`, два файла адаптера) и отмечает, что у ядра нет зависимости от HTTP-клиента.
- Секция «Cross-cutting gotchas» в `CLAUDE.md` получает буллет, утверждающий, что инъекция заголовка `host` живёт в `http`-адаптере (а не в ядре), потому что внутренний BHTTP-запрос зашифрован и никогда не достигает транспортного слоя `dart:io`, так что другие адаптеры (Dio, кастомные) должны делать то же самое.

**Тесты:**

- Новый `test/key_config_cache_test.dart` покрывает: cold get, hot get в пределах TTL, истечение TTL с инжектированными часами, `invalidate()` форсирует re-fetch, параллельные stale-get'ы разделяют один fetch, ошибка fetch пропагируется без отравления кэша.
- Новый `test/ohttp_session_test.dart` верифицирует оркестрацию (переиспользование кэша, инвалидация-кэша-на-gateway-error, сохранение-кэша-на-non-gateway-error, байты, передаваемые транспорту). Замечание: полный crypto round-trip через `OhttpSession.send` нельзя протестировать в этом пакете, потому что здесь нет имплементации HPKE-приёмника — тесты `OhttpSession` верифицируют оркестрацию; crypto round trip в изоляции уже покрыт `test/ohttp_test.dart`.
- Новый `test/adapters/http_adapter_test.dart` покрывает `HttpClientTransport` (GET keysUrl, POST на шлюз с content type `message/ohttp-req`, `OhttpGatewayException` на non-200) и `OhttpHttpClient` (извлечение URL → request-data, политика host-заголовка, обработка дефолтного порта, поведение `closeWith`). Использует `MockClient` из `package:http/testing.dart`.
- Полный набор (`dart test`) зелёный. `dart analyze` сообщает `No issues found!`. Форматтер (`dart format --line-length=120 --set-exit-if-changed`) сообщает об отсутствии изменений.

## Дополнительно

- Руководство по имплементации: детальный пошаговый план существует в `docs/superpowers/plans/2026-05-25-http-client-agnostic-integration.md` (TDD-упорядоченный, 8 задач, заканчивающихся примером + документацией). Дизайн-спецификация — в `docs/superpowers/specs/2026-05-25-http-client-agnostic-integration-design.md`. С обоими следует свериться перед разработкой.
- Эта задача **должна выполняться первой**. Каждый другой follow-up (L-02..L-07) нацелен на пути файлов, имена типов и заботы, которые существуют только после реструктуризации:
  - L-02 валидация входов выполняет проверки HTTPS-схемы на `HttpClientTransport` и валидацию authority на `OhttpRequestData`.
  - L-03 иерархия исключений приводит `OhttpGatewayException` (вводимый здесь) под новую базу `OhttpException` и добавляет `OhttpHttpException`, `OhttpTimeoutException`, `OhttpDecryptionException`, `OhttpParseException`, `OhttpKeyConfigException`, `OhttpUnsupportedSuiteException`.
  - L-04 hardening добавляет таймауты в `HttpClientTransport`, ограничение размера ответа на слое OHTTP в `OhttpSession.send`, защиту парсера BHTTP в `bhttp.dart`. Заботы о кэше KeyConfig и нормализации URL из старой L-03 удовлетворены **этой** задачей и больше не являются follow-up'ами.
  - L-05 обнуление в `hpke.dart`/`ohttp.dart` независимо; observability привязывается к `OhttpSession`.
  - L-06 тесты негативных путей перенацеливают ссылки тестов с `OhttpClient` на `OhttpSession` / `HttpClientTransport`.
  - L-07 интеграционный тест прогоняет `OhttpHttpClient` через `MockClient` end-to-end.
- Вне scope для этой задачи: имплементация Dio-адаптера (отложено — дизайн поддерживает его без изменений в ядре), персистентность кэшированных KeyConfig между перезапусками приложения, политики stale-while-revalidate или background refresh, соблюдение метаданных `not_before` / `not_after` (KeyConfig RFC 9458 их не несёт), streaming тел ответов, поддержка cipher suites, отличных от текущей фиксированной тройки `(0x0020, 0x0001, 0x0001)` (multi-suite **парсинг** адресован L-03; поддержка дополнительных suites end-to-end — отдельная работа).
- Источник: дизайн-спецификация + план имплементации в `docs/superpowers/{specs,plans}/2026-05-25-http-client-agnostic-integration*.md`.
