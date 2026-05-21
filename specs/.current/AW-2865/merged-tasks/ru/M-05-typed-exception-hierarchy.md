# Ввести типизированную иерархию `OhttpException` и обернуть ошибки аутентификации AEAD

**Оценка:** 2d
**Приоритет:** P2: High
**Компонент:** App
**Severity:** HIGH

## Описание задачи

В библиотеке нет типизированной иерархии исключений. `OhttpClient.send()` в `lib/src/ohttp_client.dart:84-87, 120-122` бросает голое `Exception('...')` для обоих не-200 ответов (GET KeyConfig и POST на гейтвей), помещая статус-код только в строку сообщения. Сетевые ошибки (DNS-сбои, сброс соединения) пробрасываются как нетипизированные исключения. Ошибки аутентификации AEAD всплывают как `SecretBoxAuthenticationError` из `package:cryptography` (через `aesGcm.decrypt(...)` в `lib/src/ohttp.dart:217-225`), что делает транзитивную зависимость частью публичной поверхности API.

Два последствия для вызывающего кода кошелька:

1. Осмысленная обработка ошибок — отличие 5xx от 4xx, таймаута от ошибки парсинга, ошибки аутентификации AEAD от транспортной ошибки — невозможна без парсинга строк сообщений или широких `catch (e)`. Это мешает разумным UI-сценариям типа «повторить при временной ошибке, прервать при misconfiguration».
2. Любое major-обновление `package:cryptography` становится breaking change для потребителей, которые ловят `SecretBoxAuthenticationError`, чтобы показывать UI «ответ испорчен».

## Технические детали

**1. Описать типизированную иерархию**

Ввести общий базовый класс `OhttpException` (наследник `Exception`) и конкретные подклассы:

- `OhttpHttpException(statusCode, body?)` — не-2xx ответ от релея или POST на гейтвей. Либо разделить дальше, либо выставить достаточно информации, чтобы вызывающий код мог различать `statusCode >= 400 && < 500` и `>= 500`.
- `OhttpTimeoutException` — таймаут запроса (согласуется с работой по таймаутам в задаче по network reliability).
- `OhttpParseException(cause)` — некорректный BHTTP/OHTTP/KeyConfig-ответ.
- `OhttpKeyConfigException` — ошибки, специфичные для парсинга KeyConfig.
- `OhttpDecryptionException(cause)` — оборачивает `SecretBoxAuthenticationError`, бросаемый `aesGcm.decrypt(...)` в `lib/src/ohttp.dart:217-225`.
- `OhttpUnsupportedSuiteException` — неподдерживаемый KEM/KDF/AEAD (если соответствующее исключение из параллельной задачи приходит отдельно — встроить его в эту иерархию под общий базовый класс).

Реэкспортировать каждый тип из `lib/ohttp_dart.dart`.

**2. Заменить голые throw** (`lib/src/ohttp_client.dart:84-87, 120-122` и `lib/src/ohttp.dart:217-225`)

- Строки 84–87: `if (configResponse.statusCode != 200) { throw Exception('...'); }` → `throw OhttpHttpException(statusCode: configResponse.statusCode, ...)`.
- Строки 120–122: тот же паттерн для POST на гейтвей.
- Строки 217–225 в `ohttp.dart`: внутри `ohttpDecapsulate` ловить `SecretBoxAuthenticationError` и бросать `OhttpDecryptionException`, оборачивающее исходный cause, без утечки имени внутреннего типа cause в публичный контракт.

Обновить doc-комментарии у `send()`, `sendDirect()` и `ohttpDecapsulate` так, чтобы они перечисляли исключения, которые могут быть выброшены.

**3. Тесты** (`test/ohttp_test.dart`, `test/ohttp_client_test.dart`)

- Мок `http.Client` возвращает 4xx → проверить, что выброшено `OhttpHttpException` с `statusCode` в 4xx-диапазоне.
- Мок `http.Client` возвращает 5xx → проверить, что выброшено `OhttpHttpException` с `statusCode` в 5xx-диапазоне.
- Сформировать корректный OHTTP encap, вывести секрет ответа, зашифровать ответ, перевернуть один байт шифротекста, вызвать `ohttpDecapsulate`, проверить, что выброшено `OhttpDecryptionException` (а не `SecretBoxAuthenticationError`).

| **Платформа** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## AC

- Библиотечный базовый класс `OhttpException` экспортируется из `lib/ohttp_dart.dart` как минимум с подтипами, перечисленными выше.
- Все места `throw Exception(...)` в `lib/src/ohttp_client.dart` заменены на типизированные подклассы, несущие релевантный контекст (статус-код, cause).
- `ohttpDecapsulate` больше не пробрасывает `SecretBoxAuthenticationError` напрямую вызывающему коду; вместо этого пробрасывается `OhttpDecryptionException`.
- Doc-комментарии у `send()`, `sendDirect()` и `ohttpDecapsulate` перечисляют исключения, которые могут быть выброшены.
- Юнит-тесты покрывают: 4xx от гейтвея → типизированное исключение с `statusCode == 4xx`; 5xx от гейтвея → типизированное исключение с `statusCode == 5xx`; ошибка аутентификации AEAD → `OhttpDecryptionException` (а не транзитивный тип из cryptography).

## Дополнительно

- Тесты: юнит-тесты на каждый типизированный путь исключения.
- Migration note: любой потребитель, который раньше ловил `Exception` от `OhttpClient.send` или `SecretBoxAuthenticationError` от decap, должен мигрировать на новую типизированную поверхность.
