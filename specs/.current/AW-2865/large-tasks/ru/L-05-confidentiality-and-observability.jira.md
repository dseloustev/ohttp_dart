# [describing] Обнуление ключевого материала, observer и нормализация заголовков

h2. Описание задачи:
 # Чувствительный ключевой материал HPKE и OHTTP должен обнуляться после использования.
 # Добавить типизированный observer для {{OhttpSession}} — поверхность observability у пакета сейчас отсутствует.
 # Привести имена заголовков ответа к нижнему регистру при материализации {{OhttpResponseData}}.
 # Запретить логирование чувствительных данных в событиях observer.

RFC:
 * RFC 9110 §5.1 (case-insensitive имена HTTP-заголовков)
 * RFC 9180 (HPKE — ключевой материал контекста sender)
 * RFC 9458 §3 (OHTTP — выведенные секреты ответа)

h2. Технические детали:

Чувствительные буферы (поля {{HpkeSenderContext}}, поля {{OhttpEncapsulateResult}}, выведенные ключи AEAD ответа) перезаписываются нулями после использования. Обнуление best-effort: Dart GC и AOT-оптимизации не дают гарантии — указывается в doc-комментариях.

Новый интерфейс {{OhttpObserver}} с реэкспортом из публичного API пакета. Опциональный параметр {{OhttpSession}} (и {{OhttpSession.withTransport}}); при {{null}} события не испускаются. События:
 * {{onKeyConfigFetch({required int statusCode, required Duration elapsed})}}
 * {{onKeyConfigCacheHit()}}
 * {{onGatewayPost({required int statusCode, required Duration elapsed})}}
 * {{onDecryptionFailure()}} — срабатывает до того, как пробрасывается {{OhttpDecryptionException}} (из AW-2937).

Запрещено логировать в событиях observer:
 * ключевой материал, значение {{enc}};
 * URL, путь, метод, заголовки, тело внутреннего запроса;
 * {{authority}} (target host) из {{OhttpRequestData}};
 * тело и заголовки ответа;
 * gateway URL на уровне DEBUG и ниже в проде.

Разрешено логировать: код статуса {{fetchKeyConfig}}, код статуса {{postToGateway}}, попадание/промах кэша, сигнал сбоя расшифровки.

Имена заголовков ответа в {{OhttpSession.send}} приводятся к нижнему регистру при сборке {{OhttpResponseData}} (порядок и дубликаты сохраняются; RFC 9110 §5.1).

|*Платформа*|All (default)|
|*URLs*|[https://github.com/AdguardTeam/ohttp_dart]|
|*Figma*| |
|*Notion*| |

h2. AC:

Обнуление:
 * {{HpkeSenderContext}} и {{OhttpEncapsulateResult}} явно обнуляются после использования; {{OhttpSession.send}} вызывает обнуление в try/finally вокруг decap (на путях успеха и сбоя).
 * Doc-комментарии документируют best-effort ограничение Dart.

Observer:
 * {{OhttpObserver}} определён в публичном API пакета с четырьмя событиями из «Технических деталей».
 * {{OhttpSession}} (и {{OhttpSession.withTransport}}) принимает опциональный {{OhttpObserver}}; при {{null}} события не испускаются.
 * Публичный doc-комментарий интерфейса воспроизводит ограничения логирования дословно.
 * Ни одно событие в полезной нагрузке не содержит запрещённого поля.

Заголовки:
 * {{OhttpResponseData.headers}} содержит имена в нижнем регистре; doc-комментарий ссылается на RFC 9110 §5.1.

Тесты:
 * Unit-тесты покрывают: обнуление на happy-path и путях сбоя; захват событий observer на успешном и падающем потоках с утверждением отсутствия запрещённых полей; поток с попаданием в кэш испускает {{onKeyConfigCacheHit}} и пропускает {{onKeyConfigFetch}}; имена заголовков в смешанном регистре нормализуются к нижнему.

h2. Дополнительно:
 - Инструкции для тестирования (если именно тут надо что-то дописать, а не в AC).
