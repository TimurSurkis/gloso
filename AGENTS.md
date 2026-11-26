# AGENTS.md — Правила разработки backend Gloso

Этот файл задаёт обязательные правила, границы и инварианты для AI-агентов и разработчиков, работающих с backend-частью проекта **Gloso** (директория [`backend/`](backend/)).

## 1. Источники истины и что строго игнорировать

### Источники истины (читать перед изменениями)

1. [`docs/index-backend.md`](docs/index-backend.md) — карта документации и статусы.
2. [`docs/concepts/decision-backend.md`](docs/concepts/decision-backend.md) — реестр согласованных решений владельца (`D-001`–`D-075`). **При любом конфликте приоритет у этого файла.**
3. [`docs/concepts/concept-backend-overview.md`](docs/concepts/concept-backend-overview.md) — краткий обзор объёма первого релиза.
4. Спецификации в [`docs/specifications/`](docs/specifications/):
   * [`docs/specifications/spec-backend-architecture.md`](docs/specifications/spec-backend-architecture.md) — модули, порты и границы зависимостей;
   * [`docs/specifications/spec-scheduling-and-booking.md`](docs/specifications/spec-scheduling-and-booking.md) — расписание, слоты и протокол бронирования;
   * [`docs/specifications/spec-data-and-retention.md`](docs/specifications/spec-data-and-retention.md) — категории данных, таблицы и ретенция;
   * [`docs/specifications/spec-calls-pilot-notifications.md`](docs/specifications/spec-calls-pilot-notifications.md) — допуск звонков, пилот, SMS и Telegram;
   * [`docs/specifications/spec-http-api.md`](docs/specifications/spec-http-api.md) — нормализованные HTTP-контракты;
   * [`docs/specifications/spec-booking-recovery.md`](docs/specifications/spec-booking-recovery.md) — порядок ручной и автоматической сверки;
   * [`docs/specifications/spec-security-and-operations.md`](docs/specifications/spec-security-and-operations.md) — безопасность, логирование, окружения и восстановление.
5. [`docs/trackers/plan-backend-implementation.md`](docs/trackers/plan-backend-implementation.md) — этапы реализации и матрица приёмки (`A-01`–`A-35`).
6. [`docs/trackers/track-backend-validation.md`](docs/trackers/track-backend-validation.md) — блокеры внешних провайдеров.
7. [`docs/trackers/track-backend-discovery.md`](docs/trackers/track-backend-discovery.md) — история проработки и открытые пункты.

### Строго игнорировать и не изменять

* **Исторические файлы** ([`docs/spec-legacy-architecture.md`](docs/spec-legacy-architecture.md), [`docs/spec-legacy-architecture-scheme.md`](docs/spec-legacy-architecture-scheme.md), [`docs/spec-legacy-database.md`](docs/spec-legacy-database.md), [`docs/spec-legacy-project.md`](docs/spec-legacy-project.md)) — устаревшие документы (старые Zadarma/SMSAPI, автоподмена на диагностику, старая схема БД). **Не переносить их требования и код автоматически.**
* **Юридические HTML-файлы** ([`regulamin.html`](regulamin.html), [`polityka-prywatnosci.html`](polityka-prywatnosci.html)) — не редактировать в рамках разработки backend.
* **Промпты голосового агента** — вне текущего объёма (`D-017`). Не писать и не менять тексты диалогов.

## 2. Текущий этап и границы первого релиза

Работа идёт по этапам (`D-065`): сначала **локальное ядро и mock-адаптеры с автотестами** (Этапы 1–2 в [`docs/trackers/plan-backend-implementation.md`](docs/trackers/plan-backend-implementation.md)), затем реальные интеграции (Этап 3).

### Входит в первый релиз (`D-001`, `D-007`, `D-070`)

* Режим **Calendar**: расчёт свободных слотов и создание новых записей в одном Google Calendar автосервиса ([`docs/specifications/spec-scheduling-and-booking.md`](docs/specifications/spec-scheduling-and-booking.md)).
* Режим **Relay**: передача новой заявки сотруднику по SMS без создания и подтверждения визита ([`docs/specifications/spec-calls-pilot-notifications.md`](docs/specifications/spec-calls-pilot-notifications.md)).
* Автоматический переход на **Relay** для текущего обращения (без смены основного режима мастерской), если:
  * Google Calendar недоступен на этапе чтения (`D-002`);
  * услуга или комплекс не найдены в каталоге (`D-015`, `D-059`);
  * нет подходящих свободных слотов (`D-054`);
  * мастерская находится в статусе сверки бронирования `RECOVERY_REQUIRED` (`D-064`);
  * звонящий просит **перенести или отменить** существующий визит (`D-070` — без поиска в календаре и без OTP).
* Административный HTTP API без UI ([`docs/specifications/spec-http-api.md`](docs/specifications/spec-http-api.md), `D-026`), пилот 14 дней / 40 минут (`D-033`), технические алерты администратору в Telegram (`D-032`).

### ЗАПРЕЩЕНО реализовывать сейчас (отложено на будущие фазы)

* Автоматический поиск, перенос и отмена существующих визитов (`D-004`, `D-005`, `D-007`).
* OTP / SMS-коды и любые SMS звонящим (`D-018`, `D-029`).
* Платные периоды, биллинг, продление пилота, пополнение/сброс/возврат минут (`D-037`, `D-038`, `D-075`). **Оставлять чистые точки расширения в модуле `access`, но не добавлять endpoints.**
* Клиентское самообслуживание, OAuth Google Calendar, админ-панель (`D-022`, `D-025`, `D-026`).
* Несколько одновременных AI-разговоров или визитов на один автосервис (`D-009`, `D-012`).
* Микросервисы, Redis, брокеры сообщений, долговечные очереди SMS (`D-027`, `D-031`).

## 3. Технологический стек и архитектура

Детали описаны в [`docs/specifications/spec-backend-architecture.md`](docs/specifications/spec-backend-architecture.md).

* **Стек (`D-023`):** Node.js 22 LTS, TypeScript (`strict: true`), Express, PostgreSQL 16+, Drizzle ORM, Zod.
* **Рекомендуемые утилиты:** `libphonenumber-js` (нормализация E.164 с дефолтной страной `PL`), `luxon` или `date-fns-tz` (работа с `Europe/Warsaw` и DST), `vitest` (тесты).
* **Структура (`D-027`):** модульный монолит в [`backend/`](backend/) со строгим разделением модулей:
  `tenants`, `calls`, `access`, `scheduling`, `bookings`, `relay`, `notifications`, `audit`.
* **Направление зависимостей:**
  `HTTP / controllers -> application use cases -> domain rules & ports`.
  * Доменный слой **не импортирует** Express, Drizzle, SDK провайдеров и `process.env`.
  * Внешние провайдеры (Google Calendar, Twilio Voice/Messaging, ElevenLabs, Telegram) подключаются строго через порты (`CalendarPort`, `SmsPort`, `TechnicalAlertPort`, `CallContextResolver`, `ProviderEventVerifier`, `Clock`, `HolidayCalendar`).
  * Циклические зависимости между модулями запрещены.

## 4. Жёсткие инварианты (нарушать запрещено)

### 4.1. Приватность и Zero-PII (`D-006`, `D-040`–`D-043`, `D-072`, `D-073`)

Детали схемы и сроков хранения — в [`docs/specifications/spec-data-and-retention.md`](docs/specifications/spec-data-and-retention.md).

* **Строго запрещено** сохранять в PostgreSQL, логах, трассировках, ошибках SDK, Telegram или долговечных очередях:
  имя и телефон звонящего (и их фрагменты/хеши), марку/модель авто, описание проблемы, время визита (`start_at`/`end_at`), строку `slot_id`, текст SMS/заявки, полные HTTP-ответы, сырые тела webhook, аудио и транскрипции.
* Содержимое обращения живёт **только в оперативной памяти** на время обработки запроса и ограниченных in-memory повторов SMS.
* **Логирование:** только явный allowlist из [`docs/specifications/spec-security-and-operations.md`](docs/specifications/spec-security-and-operations.md) (модуль, маршрут без PII, безопасный код ошибки, метрики). Никогда не логировать `req.body`, `res.body`, заголовки авторизации и SQL-параметры с PII.
* **Аудит (`audit_logs`):**
  * Шифруется `AES-256-GCM` со случайным nonce и AAD.
  * Результат строго один из 3 (`BOOKING_CREATED`, `RELAY_SENT`, `ERROR`) либо `null` (если не было операции или исход неизвестен — **не добавлять 4-й статус** и не писать `ERROR` при неизвестном исходе).
  * При успешной записи и сбое SMS результат аудита — `BOOKING_CREATED`.
  * Время хранится с точностью до **часового интервала завершения разговора** (`D-073`); граница удаления `expires_at` = начало UTC-часа завершения + 30×24 часа (если завершение потеряно — временно от часа начала AI-разговора).
  * Длительность — только списанные целые минуты (`ceil(ai_seconds / 60)`), без точных секунд. Не добавлять на `audit_logs` точные `created_at`/`updated_at`.
  * Любое чтение аудита обязано фильтровать `expires_at > now()`.
* **Операционное состояние:** верхний срок хранения — 7 дней (`D-062`), без продления повторами.
* **Оффбординг (`D-072`):** деактивация мастерской + затирание контакта сотрудника и отвязка ресурсов (номер, агент, календарь) в течение 14 дней; пустой UUID-якорь остаётся до естественного истечения 30-дневного аудита. Не делать безусловный каскадный `DELETE` мастерской.

### 4.2. Доверенный контекст и безопасность (`D-051`, `D-055`)

* Раздельная аутентификация для Admin API (`Authorization: Bearer`, сравнение через `crypto.timingSafeEqual`), Tools API и Provider Events ([`docs/specifications/spec-security-and-operations.md`](docs/specifications/spec-security-and-operations.md)).
* **Никогда не доверять** `tenant_id`, `agent_id` или `call_id` из JSON-аргументов LLM. Контекст извлекается только через `CallContextResolver`.
* Тестовые заголовки и mock-адаптеры должны быть гарантированно заблокированы при `NODE_ENV === 'production'` (сценарий `A-32` в [`docs/trackers/plan-backend-implementation.md`](docs/trackers/plan-backend-implementation.md)).
* Секреты хранятся только в env/secret store и никогда не попадают в git, миграции, seed, БД или ответы API.

### 4.3. Звонки, допуск и пилот (`D-009`, `D-033`–`D-039`, `D-053`, `D-056`, `D-060`, `D-061`, `D-063`)

Детали — в [`docs/specifications/spec-calls-pilot-notifications.md`](docs/specifications/spec-calls-pilot-notifications.md).

* Допуск проверяется **до старта AI** в режиме **fail-closed** (`D-056`).
* Лимиты: 1 активный AI-разговор на автосервис, максимум 180 секунд (настраиваемо), пилот 14 календарных дней в `Europe/Warsaw` (до того же местного времени) и 40 минут AI.
* Ручная однократная активация пилота (`D-034`); повтор активации не сбрасывает расход и не запускает новый пилот.
* Расход: `ceil(ai_seconds / 60)` на каждый фактически подключённый разговор (0 сек = 0 мин), включая прерванные и спам; списание идемпотентно.
* **Активный разговор не обрывается** при истечении срока/минут пилота или при ручной паузе — блокируются только новые допуски.
* Ручная пауза/возобновление (`D-061`) не меняет дату окончания пилота и расход.

### 4.4. Расписание, слоты и контакты (`D-013`–`D-015`, `D-019`, `D-044`–`D-050`, `D-059`, `D-067`, `D-074`)

Детали — в [`docs/specifications/spec-scheduling-and-booking.md`](docs/specifications/spec-scheduling-and-booking.md).

* Часовой пояс: `Europe/Warsaw`, сравнение моментов в UTC, без фиксированного смещения.
* Приоритет расписания: исключение на дату -> польский государственный праздник (закрыт по умолчанию, настраиваемо) -> недельное расписание.
* Параметры по умолчанию (настраиваются на автосервис): минимальный срок до визита `60 мин`, горизонт `14 дней` (до конца местного дня `today + horizon_days` включительно), сетка слотов от `00:00` с шагом `30 мин`, буфер `0 мин`, тестовая длительность для фикстур `60 мин`.
* Ненулевой буфер применяется **только между визитами** (включая ручные записи) и **не отрезает** последний слот в конце рабочего дня, если услуга успевает закончиться до закрытия.
* Полная длительность выбранной услуги не округляется до шага сетки; автосуммирование нескольких услуг запрещено.
* При активном пилоте разрешено создавать визит на дату после окончания пилота в пределах горизонта (`D-067`).
* **Контактный телефон (`D-019`):** принимается любой валидный международный номер E.164 (для Calendar и Relay); локальный 9-значный номер нормализуется с дефолтным кодом `+48`. Список разрешённых стран выносится в конфиг. Явно переданный контакт не подменяется молча Caller ID.

### 4.5. Конкурентность, шлюз бронирования и восстановление (`D-057`, `D-058`, `D-064`, `D-071`)

Порядок сверки — в [`docs/specifications/spec-booking-recovery.md`](docs/specifications/spec-booking-recovery.md).

* Максимум **одна** созданная запись Calendar **либо одна** переданная Relay-заявка за AI-разговор (`D-058`).
* **Короткие SQL-транзакции:** внешние HTTP-запросы (Calendar, SMS, Telegram) **никогда** не выполняются внутри открытой транзакции PostgreSQL.
* Межпроцессная сериализация: таблица `booking_gates` + короткий `pg_advisory_xact_lock(hashtext(tenant_id))`.
* Двухтранзакционный протокол бронирования:
  1. Транзакция 1: проверка идемпотентности и `booking_gates`, запись `PENDING` в `operations`, генерация стабильного совместимого с Google `external_ref` и фиксация исходного `calendar_ref` — **коммит до внешнего вызова**.
  2. Вне транзакции: повторная проверка доступности и вызов `CalendarPort.createEvent` с тем же `external_ref`.
  3. Транзакция 2: фиксация `COMPLETED` / `FAILED` (и освобождение gate с проверкой `owner_token`/`owner_version`) либо `UNCERTAIN` (gate остаётся заблокирован).
* `404` при сверке, таймаут или истечение lease **не доказывают**, что предыдущий `insert` завершился, и **не освобождают** gate автоматически. При истечении 7-дневного окна ссылки операции очищаются, а gate переводится в минимальный `RECOVERY_REQUIRED`.
* Свободная смена режима `CALENDAR <-> RELAY` через Admin API разрешена в любой момент, но **не сбрасывает** `RECOVERY_REQUIRED` (`D-071`).
* Эндпоинт `POST /admin/tenants/:id/recovery/resolve` обязан проверять `owner_version` и явный подтверждённый исход (`CONFIRMED_CREATED` / `CONFIRMED_NOT_CREATED`), без слепого сброса в `IDLE`.

### 4.6. Уведомления (`D-028`–`D-032`)

* SMS отправляется только одному назначенному сотруднику. Ошибка SMS **не отменяет** созданный визит в Google Calendar.
* Ограниченные повторы SMS (проектно: до 3 попыток, задержки 1 с и 3 с) выполняются **только в оперативной памяти** и **только при подтверждённом временном отказе без принятия**. Постоянные ошибки не повторяются; неизвестный исход (таймаут) **не повторяется вслепую**.
* В Telegram администратору уходят только безопасные технические коды (сбой SMS, `RECOVERY_REQUIRED`, переход пилота к окончанию/исчерпанию — один раз на переход), строго без PII и сырых стектрейсов.

## 5. Правила разработки и тестирования

1. **Изоляция тестовой БД:** тесты обязаны использовать отдельную временную БД PostgreSQL и **никогда** не запускаться поверх рабочего `DATABASE_URL`.
2. **Миграции:** применяются явно через Drizzle (`db:migrate`), не запускаются автоматически при старте HTTP-сервера.
3. **Готовность (Readiness):** `/health/ready` возвращает `false` до завершения стартовой очистки просроченных данных и проверки соединения с БД.
4. **Обязательное покрытие:** реализация проверяется по матрице `A-01`–`A-35` из [`docs/trackers/plan-backend-implementation.md`](docs/trackers/plan-backend-implementation.md), включая конкурентные тесты с двумя процессами backend и внешним mock-провайдером, переживающим рестарт backend.
5. **Запрет на молчаливые изменения требований:** если при реализации обнаружено противоречие в спецификациях или невозможность выполнить требование провайдера, не придумывать обходное продуктовое решение молча — остановиться и задать владельцу один конкретный вопрос.