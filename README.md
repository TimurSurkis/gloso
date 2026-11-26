# Gloso

Голосовой AI-ассистент для автосервисов в Польше. Обрабатывает пропущенные и занятые входящие звонки (в рабочее и нерабочее время), записывает клиентов в Google Calendar либо передаёт структурированные заявки сотруднику по SMS.

## Текущий статус

Проект находится на этапе **document-first проектирования (v0.2)** и переходит к реализации локального ядра backend с mock-адаптерами (Этапы 1–2 по [плану реализации](docs/trackers/plan-backend-implementation.md)):

* Продуктовые требования и ключевые развилки зафиксированы в решениях `D-001`–`D-075` ([реестр решений](docs/concepts/decision-backend.md)).
* Технические спецификации в [`docs/specifications/`](docs/specifications/) описывают целевой проект первого релиза.
* Живой пилот заблокирован до прохождения интеграционных и эксплуатационных проверок из [`docs/trackers/track-backend-validation.md`](docs/trackers/track-backend-validation.md) (маршрут `Twilio → ElevenLabs` с допуском до старта AI, польские номера/отправители SMS, гарантии удаления и бэкапов в Neon).
* Юридические файлы [`regulamin.html`](regulamin.html) и [`polityka-prywatnosci.html`](polityka-prywatnosci.html) ещё содержат старые упоминания Zadarma/SMSAPI и будут обновлены после финализации технической модели.

## Возможности первого релиза

* **Режим Calendar:** проверка расписания мастерской (`Europe/Warsaw`, перерывы, исключения, польские госпраздники), расчёт слотов на полную длительность услуги и создание записи в одном Google Calendar автосервиса (подключение через Service Account).
* **Режим Relay:** сбор данных обращения и передача заявки одному назначенному сотруднику по SMS без подтверждения визита. Автоматически используется как fallback при недоступности календаря, неизвестной услуге/комплексе, отсутствии слотов, нахождении календаря на сверке (`RECOVERY_REQUIRED`) и при просьбах перенести/отменить существующий визит.
* **Пилот и допуск:** 14 календарных дней и 40 минут AI на автосервис (ручная активация, учёт `ceil(ai_seconds / 60)`), максимум 1 активный AI-разговор на мастерскую (потолок 180 сек), жёсткий **fail-closed** допуск *до* подключения AI.
* **Zero-PII в БД и логах:** персональные данные звонящих, детали авто/проблемы, время визита и тексты SMS не сохраняются в PostgreSQL, логах или очередях — они обрабатываются только в оперативной памяти. В БД хранятся конфигурация, краткоживущее операционное состояние (до 7 дней) и зашифрованный технический аудит (до 30 дней с точностью до часового интервала).
* **Управление и эксплуатация:** защищённый административный HTTP API без UI и технические оповещения администратору в Telegram (строго без PII).

Переносы/отмены с OTP, клиентские SMS, биллинг, OAuth и клиентский кабинет осознанно отложены за пределы первого релиза.

## Технологический стек

* **Backend:** Node.js 22 LTS, TypeScript (`strict`), Express, PostgreSQL 16+, Drizzle ORM, Zod — модульный монолит с портами и адаптерами.
* **Внешние сервисы:** Twilio (Programmable Voice и Programmable Messaging), ElevenLabs Conversational AI, Google Calendar API, Telegram (технические алерты).
* **Целевой хостинг:** Northflank (приложение) + Neon в ЕЭЗ (PostgreSQL).

## Структура репозитория

| Путь | Назначение |
| --- | --- |
| [`backend/`](backend/) | Исходный код серверной части (модульный монолит, миграции, тесты) |
| [`docs/`](docs/) | Техническая документация backend (см. навигацию ниже) |
| [`AGENTS.md`](AGENTS.md) | Обязательные правила, инварианты и границы для AI-агентов и разработчиков |
| [`landing_page_v2.html`](landing_page_v2.html) | Статический лендинг проекта (стили в [`src/input.css`](src/input.css)) |
| [`regulamin.html`](regulamin.html) | Регламент оказания услуги и DPA (требует синхронизации перед пилотом) |
| [`polityka-prywatnosci.html`](polityka-prywatnosci.html) | Политика приватности (требует синхронизации перед пилотом) |
| [`audio/`](audio/), [`design/`](design/), [`img/`](img/) | Статические медиа- и дизайн-материалы |

## Документация

Точка входа — [`docs/index-backend.md`](docs/index-backend.md). Рекомендуемый порядок чтения:

1. [`docs/concepts/concept-backend-overview.md`](docs/concepts/concept-backend-overview.md) — обзор объёма и ограничений.
2. [`docs/concepts/decision-backend.md`](docs/concepts/decision-backend.md) — реестр согласованных решений (`D-001`–`D-075`).
3. [`docs/specifications/spec-backend-architecture.md`](docs/specifications/spec-backend-architecture.md) — модули, порты и конкурентность.
4. [`docs/specifications/spec-scheduling-and-booking.md`](docs/specifications/spec-scheduling-and-booking.md) — расписание, слоты и протокол бронирования.
5. [`docs/specifications/spec-data-and-retention.md`](docs/specifications/spec-data-and-retention.md) — схема данных, аудит и ретенция.
6. [`docs/specifications/spec-calls-pilot-notifications.md`](docs/specifications/spec-calls-pilot-notifications.md) — звонки, пилот, SMS и Telegram.
7. [`docs/specifications/spec-http-api.md`](docs/specifications/spec-http-api.md) — нормализованные HTTP-контракты.
8. [`docs/specifications/spec-booking-recovery.md`](docs/specifications/spec-booking-recovery.md) — ручная и автоматическая сверка бронирований.
9. [`docs/specifications/spec-security-and-operations.md`](docs/specifications/spec-security-and-operations.md) — безопасность, секреты и эксплуатация.
10. [`docs/trackers/plan-backend-implementation.md`](docs/trackers/plan-backend-implementation.md) и [`docs/trackers/track-backend-validation.md`](docs/trackers/track-backend-validation.md) — план этапов, матрица приёмки (`A-01`–`A-35`) и блокеры провайдеров.

> **Важно:** файлы [`docs/spec-legacy-architecture.md`](docs/spec-legacy-architecture.md), [`docs/spec-legacy-architecture-scheme.md`](docs/spec-legacy-architecture-scheme.md), [`docs/spec-legacy-database.md`](docs/spec-legacy-database.md) и [`docs/spec-legacy-project.md`](docs/spec-legacy-project.md) являются историческими материалами и не отражают актуальную архитектуру.

## Разработка и безопасность

* Правила работы над кодом и архитектурные инварианты собраны в [`AGENTS.md`](AGENTS.md).
* Инструкции по локальному запуску, миграциям и тестам будут поддерживаться в `backend/README.md` после инициализации локального ядра (Этап 1).
* Ключи API, токены, учётные данные Google/Twilio/Telegram и реальные данные обращений никогда не коммитятся в репозиторий.