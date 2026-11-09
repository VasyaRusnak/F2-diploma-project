# Software Requirements Specification (SRS)

Стандарт: ISO/IEC/IEEE 29148. Проєкт: «Цукровий Компаньйон» / GlucoBuddy. Версія: 1.0 (Approved для Розділу 2). Дата: 2026-11-09.

Похідність: вимоги мігровано з розділу «Аналіз предметної області» (ЛР№4, Завд.1–2). Початкові 8 Core Features розбито на 21 атомарну вимогу; матрицю NFR перенесено та уточнено за результатами аудиту.

Пов'язані документи: [Розділ 2. Проєктування архітектури](./2_design.md) · [ADR-001](./adr/0001-choice-of-technology.md)

# 1. System Context (Контекст системи)

## 1.1 Межі системи (Scope)

Система GlucoBuddy є розподіленою подієво-керованою мікросервісною системою з розділенням моделей запису й читання (CQRS). Система приймає телеметричні події пацієнта (глюкоза, хлібні одиниці, доза інсуліну), зберігає їх із гарантією доставки at-least-once, формує read-моделі для аналітики та ігрового контуру, надсилає нагадування й надає рольовий доступ до даних.

У межах системи: прийом і валідація подій, outbox-публікація, проєкції, гейміфікація (XP, streak, Achievements), BFF, нагадування, автентифікація, авторизація, аудит, адаптер CGM, видалення даних.

Поза межами системи: рекомендації щодо дозування інсуліну; кваліфікація за MDR 2017/745 та IEC 62304; розпізнавання голосу; клієнти для Wear OS/watchOS; експорт у HL7 FHIR; експорт даних за GDPR ст. 20; багаторегіональне розгортання.

## 1.2 Взаємодія із зовнішнім середовищем

| Зовнішня сутність | Тип | Взаємодія | Протокол |
| --- | --- | --- | --- |
| Дитина (Patient) | Користувач | Реєструє показники, взаємодіє з компаньйоном | REST / HTTPS (через Mobile App) |
| Опікун (Caregiver) | Користувач | Переглядає метрики, отримує ескалації, надає згоду | REST / HTTPS (через Mobile App) |
| Лікар-ендокринолог (Clinician) | Користувач | Переглядає агреговані метрики за активним AccessGrant | REST / HTTPS (через Mobile App) |
| Dexcom API | Зовнішня система | Система отримує дані CGM за pull-моделлю (polling), лише читання | REST / HTTPS, OAuth 2.0 |
| FCM / APNs | Зовнішня система | Система передає push-повідомлення | HTTPS (FCM HTTP v1), HTTP/2 (APNs) |

## 1.3 Обмеження та припущення

- Система не розраховує та не рекомендує дози інсуліну (висновок щодо MDR зафіксовано в ЛР№4, Лог №1).
- Розгортання: Docker / Docker Compose на локальному стенді. Конфігурація стенду для навантажувальних тестів: 4 vCPU, 8 ГБ RAM (уточнюється перед тестуванням і фіксується в протоколі тесту).
- Юрисдикція експлуатації: Україна та ЄС. HIPAA не є обов'язковою вимогою; при інтеграції з Dexcom API застосовуються умови провайдера.

# 2. Functional Requirements (Функціональні вимоги)

Кожна вимога містить модальне дієслово «повинна/повинен/має». Якір формату `srs-fun-NN` використовується для трасування (`srs.md#srs-fun-NN`).

| ID | Вимога | Модуль | Походження (ЛР№4) |
| --- | --- | --- | --- |
| <a id="srs-fun-01"></a>SRS-FUN-01 | Ingestion API повинен приймати подію показника (глюкоза, ХО, інсулін) методом `POST /v1/events` у форматі JSON через REST / HTTPS. | Ingestion | Ingestion #1 |
| <a id="srs-fun-02"></a>SRS-FUN-02 | Ingestion API повинен відхиляти подію, що не відповідає JSON-схемі, відповіддю HTTP 422 із переліком порушених полів. | Ingestion | Ingestion #1 |
| <a id="srs-fun-03"></a>SRS-FUN-03 | Ingestion API повинен відхиляти подію зі значенням поза конфігурованими границями (за замовчуванням: глюкоза 0,6–33,3 ммоль/л; ХО 0–30 на подію; інсулін 0,05–30 од.) відповіддю HTTP 422. | Ingestion | Ingestion #1 |
| <a id="srs-fun-04"></a>SRS-FUN-04 | Ingestion API повинен при повторній відправці події з тим самим заголовком `Idempotency-Key` повертати збережену відповідь (HTTP 200) без повторного запису. | Ingestion | Ingestion #1 |
| <a id="srs-fun-05"></a>SRS-FUN-05 | Ingestion Service повинен зберігати подію та запис outbox в одній транзакції PostgreSQL. | Ingestion | Ingestion #2 |
| <a id="srs-fun-06"></a>SRS-FUN-06 | Outbox Relay повинен публікувати записи outbox у Kafka з ключем партиціонування `patientId` і позначати їх як опубліковані лише після підтвердження брокера (acks=all). | Ingestion | Ingestion #2 |
| <a id="srs-fun-07"></a>SRS-FUN-07 | Projection Service повинен формувати часові ряди показників у TimescaleDB із continuous aggregates. | Projection | Projection #1 |
| <a id="srs-fun-08"></a>SRS-FUN-08 | Gamification Service повинен нараховувати XP не більше одного разу для кожного `eventId`. | Gamification | Gamification #1 |
| <a id="srs-fun-09"></a>SRS-FUN-09 | Gamification Service повинен обчислювати streak як кількість послідовних календарних діб (за часовим поясом IANA пацієнта), у кожній з яких зареєстровано щонайменше 4 події глюкози (значення конфігурується). | Gamification | Gamification #1 |
| <a id="srs-fun-10"></a>SRS-FUN-10 | Gamification Service повинен присвоювати Achievement при виконанні умови з каталогу досягнень не більше одного разу для пари (`patientId`, `achievementId`). | Gamification | Gamification #1 |
| <a id="srs-fun-11"></a>SRS-FUN-11 | Gamification Service повинен оновлювати `CompanionState` у Redis після кожної обробки події. | Gamification | Projection #1 |
| <a id="srs-fun-12"></a>SRS-FUN-12 | BFF повинен повертати клієнту дитини `CompanionState`, XP, streak та рівень. | BFF | BFF #1 |
| <a id="srs-fun-13"></a>SRS-FUN-13 | BFF повинен повертати опікуну та лікарю Time in Range, середнє значення, коефіцієнт варіації та тренди за період від 1 до 90 діб. | BFF | BFF #2 |
| <a id="srs-fun-14"></a>SRS-FUN-14 | Reminder Service повинен планувати та доставляти нагадування через FCM / APNs. | Reminder | Reminder #1 |
| <a id="srs-fun-15"></a>SRS-FUN-15 | Reminder Service повинен надсилати сповіщення-ескалацію опікуну, якщо протягом 15 хв після нагадування подію не зареєстровано. | Reminder | Reminder #1 |
| <a id="srs-fun-16"></a>SRS-FUN-16 | Auth Service повинен автентифікувати користувачів за OAuth 2.0 / OIDC (Authorization Code + PKCE) і видавати JWT. | Auth & Audit | Auth #1 |
| <a id="srs-fun-17"></a>SRS-FUN-17 | Auth Service повинен надавати доступ опікуну або лікарю до даних пацієнта лише за активним `AccessGrant` (RBAC + ABAC). | Auth & Audit | Auth #1 |
| <a id="srs-fun-18"></a>SRS-FUN-18 | Audit Service повинен реєструвати кожне звернення до медичних даних із полями `userId`, `patientId`, `action`, `timestamp`, `requestId`. | Auth & Audit | Auth #1 |
| <a id="srs-fun-19"></a>SRS-FUN-19 | CGM Adapter повинен опитувати Dexcom API з інтервалом 15 хв і передавати нові показники в Ingestion API. | CGM Adapter | Додано за аудитом |
| <a id="srs-fun-20"></a>SRS-FUN-20 | Auth Service повинен вимагати підтверджену згоду опікуна перед створенням профілю дитини (GDPR, ст. 8). | Auth & Audit | Додано за аудитом |
| <a id="srs-fun-21"></a>SRS-FUN-21 | Система повинна видаляти персональні та медичні дані пацієнта протягом 30 діб після запиту опікуна (GDPR, ст. 17). | Auth & Audit | Додано за аудитом |

# 3. Non-Functional Requirements (матриця NFR)

| ID | Категорія | Метрика | Цільове значення та умови вимірювання |
| --- | --- | --- | --- |
| <a id="srs-nfr-perf-01"></a>SRS-NFR-PERF-01 | Продуктивність | Latency Ingestion API, p95 | ≤ 200 мс при 500 RPS; вимірювання на межі API Gateway; тривалість ≥ 10 хв; ≥ 300 автентифікованих віртуальних користувачів; payload ≤ 1 КБ |
| <a id="srs-nfr-perf-02"></a>SRS-NFR-PERF-02 | Продуктивність | Latency Ingestion API, p99 | ≤ 500 мс за умов SRS-NFR-PERF-01 |
| <a id="srs-nfr-perf-03"></a>SRS-NFR-PERF-03 | Продуктивність | Затримка «HTTP 2xx Ingestion → запис у read-модель», p95 | ≤ 2 с |
| <a id="srs-nfr-perf-04"></a>SRS-NFR-PERF-04 | Продуктивність | Latency BFF (read-запити), p95 | ≤ 150 мс при 100 RPS |
| <a id="srs-nfr-perf-05"></a>SRS-NFR-PERF-05 | Продуктивність | Redis cache hit ratio (ключі `CompanionState`) | ≥ 90 % |
| <a id="srs-nfr-scal-01"></a>SRS-NFR-SCAL-01 | Масштабованість | Kafka consumer lag | менше 1000 повідомлень і ≤ 5 с при 500 RPS |
| <a id="srs-nfr-scal-02"></a>SRS-NFR-SCAL-02 | Масштабованість | Відмова одного з двох інстансів Ingestion Service | частка HTTP 5xx ≤ 0,1 % протягом failover; відновлення пропускної здатності ≤ 30 с |
| <a id="srs-nfr-rel-01"></a>SRS-NFR-REL-01 | Надійність | SLO доступності | ≥ 99,9 % за 30 діб (≤ 43,2 хв недоступності); SLI = частка відповідей Gateway зі статусом, відмінним від 5xx; заплановані роботи виключаються |
| <a id="srs-nfr-rel-02"></a>SRS-NFR-REL-02 | Надійність | Семантика доставки | at-least-once + ідемпотентні споживачі; повідомлення спрямовується в DLQ після 5 невдалих спроб з експоненційною затримкою |
| <a id="srs-nfr-rel-03"></a>SRS-NFR-REL-03 | Надійність | RPO / RTO | ≤ 5 хв / ≤ 30 хв |
| <a id="srs-nfr-rel-04"></a>SRS-NFR-REL-04 | Надійність | Резервне копіювання та реплікація PostgreSQL | щоденна повна копія зі зберіганням 7 діб; безперервна архівація WAL (`archive_timeout` ≤ 60 с); асинхронна streaming-реплікація |
| <a id="srs-nfr-sec-01"></a>SRS-NFR-SEC-01 | Безпека | Автентифікація | OAuth 2.0 / OIDC; JWT RS256; TTL access-токена 15 хв; refresh-токен 7 діб із ротацією; алгоритми `none` та HS* відхиляються |
| <a id="srs-nfr-sec-02"></a>SRS-NFR-SEC-02 | Безпека | Шифрування | TLS ≥ 1.2 (канал); AES-256-GCM (поля PHI у стані спокою) |
| <a id="srs-nfr-sec-03"></a>SRS-NFR-SEC-03 | Безпека | Rate limiting | ≤ 100 запитів/хв на користувача; відповідь HTTP 429 із заголовком `Retry-After`; алгоритм sliding window на API Gateway |
| <a id="srs-nfr-sec-04"></a>SRS-NFR-SEC-04 | Безпека | Аудит доступу | 100 % звернень до PHI журналюються (SRS-FUN-18) |
| <a id="srs-nfr-sec-05"></a>SRS-NFR-SEC-05 | Безпека | Комплаєнс | OWASP API Security Top 10 (0 знахідок рівня High у ZAP API scan); GDPR: мінімізація даних, видалення ≤ 30 діб |
| <a id="srs-nfr-sec-06"></a>SRS-NFR-SEC-06 | Безпека | Безпека ланцюга постачання | 0 CRITICAL/HIGH із наявним виправленням (Trivy); 0 витоків секретів (gitleaks); SonarQube Quality Gate = Passed |
| <a id="srs-nfr-mnt-01"></a>SRS-NFR-MNT-01 | Супровід | Покриття коду | ≥ 80 % для пакетів `domain` та `application` |
| <a id="srs-nfr-mnt-02"></a>SRS-NFR-MNT-02 | Супровід | Спостережуваність | OpenTelemetry-трасування (наскрізний `traceId` Ingestion → Kafka → споживачі), метрики Prometheus, дашборди Grafana |
| <a id="srs-nfr-mnt-03"></a>SRS-NFR-MNT-03 | Супровід | Розгортання | Docker; CI/CD у GitHub Actions; `docker compose up` приводить усі контейнери у стан healthy |

# 4. Verification Criteria (Критерії верифікації)

| Вимога | Метод верифікації | Інструмент | Критерій прийняття |
| --- | --- | --- | --- |
| SRS-FUN-01…04 | Контрактні та інтеграційні тести API | Postman / Newman, Go test (`httptest`), Testcontainers (PostgreSQL) | 100 % сценаріїв пройдено; дублікат `Idempotency-Key` не створює другого запису |
| SRS-FUN-05 | Fault injection: переривання транзакції | Go test + Testcontainers | Подія без outbox-запису (і навпаки) відсутня |
| SRS-FUN-06 | Інтеграційний тест брокера | Testcontainers (Kafka), Go test | Ключ повідомлення = `patientId`; порядок подій одного пацієнта збережено |
| SRS-FUN-07 | Інтеграційний тест проєкції | Go test, SQL-перевірка continuous aggregate | Агрегат збігається з еталонним розрахунком |
| SRS-FUN-08…11 | Модульні тести та повторна доставка подій | Jest, Testcontainers (Kafka, Redis) | Повторна доставка 100 однакових подій не змінює XP, streak та Achievements |
| SRS-FUN-12…13 | API-тести та порівняння з еталоном | Jest + Supertest, Newman | Значення TIR, середнього та коефіцієнта варіації збігаються з еталоном для тестового набору |
| SRS-FUN-14…15 | Тест із керованим часом та заглушкою провайдера | Jest/Go test, WireMock (FCM/APNs) | Ескалація відправлена на 15-й хв; без пропуску ескалація відсутня |
| SRS-FUN-16…17, 20 | E2E-тести автентифікації та авторизації | Jest e2e, Keycloak у Testcontainers, OWASP ZAP | Без grant — HTTP 403; прострочений токен — HTTP 401; профіль без згоди не створюється |
| SRS-FUN-18, SRS-NFR-SEC-04 | Звірка кількості запитів із кількістю записів аудиту | E2E-тест, SQL | Кількість записів аудиту = кількості звернень до PHI |
| SRS-FUN-19 | Тест із заглушкою Dexcom API | WireMock, Go test | Нові показники передано рівно один раз |
| SRS-FUN-21, SRS-NFR-SEC-05 | Тест процедури видалення; сканування API | E2E-тест, OWASP ZAP API scan, Schemathesis | Дані відсутні в PostgreSQL, TimescaleDB, Redis; 0 High-знахідок |
| SRS-NFR-PERF-01, 02 | Навантажувальний тест (сценарії Ramp-up та Spike) | k6 (`ramping-arrival-rate`) | Пороги `http_req_duration: p(95)<200, p(99)<500` при 500 RPS |
| SRS-NFR-PERF-03 | Вимірювання затримки до read-моделі | k6 + Prometheus `histogram_quantile` | p95 ≤ 2 с |
| SRS-NFR-PERF-04 | Навантажувальний тест BFF | k6 | p95 ≤ 150 мс при 100 RPS |
| SRS-NFR-PERF-05 | Метрики Redis | redis_exporter (`keyspace_hits / (hits + misses)`) | ≥ 90 % |
| SRS-NFR-SCAL-01 | Моніторинг відставання споживачів | kafka-exporter, Prometheus, Grafana | `kafka_consumergroup_lag` менше 1000 |
| SRS-NFR-SCAL-02 | Тест відмови інстансу | `docker compose --scale`, k6 | Частка 5xx ≤ 0,1 %; відновлення ≤ 30 с |
| SRS-NFR-REL-01 | Синтетичне опитування | blackbox_exporter, Prometheus, Grafana (SLO-панель) | SLI ≥ 99,9 % за період тестування |
| SRS-NFR-REL-02 | Chaos-тест: зупинка споживача, poison message | Docker CLI, Jest/Go test | XP незмінний; після 5 спроб повідомлення в DLQ |
| SRS-NFR-REL-03, 04 | Тест відновлення (restore drill) | WAL-G або pgBackRest, bash-скрипт із вимірюванням часу | Відновлення на довільну мітку часу; RPO ≤ 5 хв; RTO ≤ 30 хв |
| SRS-NFR-SEC-01 | Негативні тести токенів | Jest e2e | Прострочений токен та алгоритми `none`/HS256 відхиляються |
| SRS-NFR-SEC-02 | Сканування TLS; перевірка шифртексту | testssl.sh, SQL-перевірка | Протоколи нижче TLS 1.2 вимкнено; значення PHI-полів у БД не є відкритим текстом |
| SRS-NFR-SEC-03 | Тест перевищення ліміту | k6 | 101-й запит за хвилину — HTTP 429 з `Retry-After` |
| SRS-NFR-SEC-06 | Статичний аналіз та сканування | SonarQube, Trivy, gitleaks у GitHub Actions | Quality Gate = Passed; 0 CRITICAL/HIGH; 0 секретів |
| SRS-NFR-MNT-01 | Звіт покриття | `go test -cover`, Jest `--coverage`, SonarQube | ≥ 80 % для `domain` та `application` |
| SRS-NFR-MNT-02 | Перевірка трасування та дашбордів | Grafana, Jaeger/Tempo, E2E-тест | Один `traceId` у логах усіх сервісів ланцюга |
| SRS-NFR-MNT-03 | Перевірка CI/CD | GitHub Actions | Pipeline = success; усі контейнери healthy |

# 5. Glossary (Глосарій)

| Термін | Визначення |
| --- | --- |
| CGM (Continuous Glucose Monitoring) | Безперервний моніторинг глюкози: сенсор формує значення приблизно кожні 5 хв |
| Time in Range (TIR) | Частка часу, протягом якої глюкоза перебуває в цільовому діапазоні, у відсотках від періоду вимірювання |
| CQRS | Патерн розділення моделей запису (команди) та читання (запити) з окремими сховищами |
| Transactional Outbox | Патерн, за якого подія й запис про публікацію зберігаються в одній транзакції БД, а окремий процес публікує їх у брокер |
| Idempotency Key | Унікальний ключ запиту, за яким сервер розпізнає повторну відправку та не виконує дію вдруге |
| Consumer Lag | Різниця між останнім зсувом (offset) у партиції та зсувом, підтвердженим групою споживачів |
| DLQ (Dead Letter Queue) | Окремий топік або черга для повідомлень, які не вдалося обробити після заданої кількості спроб |