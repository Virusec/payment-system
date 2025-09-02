1. Регистрация пользователя и кошелька
Client App - Auth Service: логин/регистрация, получение JWT (TLS).
Client App - API Gateway: POST /users + Authorization: Bearer <JWT> + Idempotency-Key: <uuid>.
API Gateway (фильтры): валидирует JWT, проверяет Idempotency-Key в Redis Idempotency Store: если найден пометить FINAL - вернуть сохранённый ответ, иначе пометить PENDING и пропустить запрос дальше.
API Gateway - User Service - User Service (@Transactional): пишет пользователя в Users DB (JPA) отдаёт userId.
User Service - Accounts & Ledger Service: POST /wallets (создать кошелёк под userId).
Accounts & Ledger Service (@Transactional, SERIALIZABLE): создаёт запись кошелька (balance=0), пишет в outbox и коммит.
Outbox (CDC) - Kafka: через Debezium CDC.
History Service (Kafka consumer): идемпотентно применяет событие - обновляет History DB.
User Service - API Gateway - Client: 200 OK с userId, walletId
API Gateway: сохраняет итоговый ответ в Redis Idempotency Store для ключа идемпотентности.


2. Пополнение счёта и переводы между пользователями

Пополнение через внешнего провайдера (PSP) — SAGA:
Client - Gateway: POST /topups (пополнение) + Idempotency-Key.
Gateway: проверка ключа в Redis - проксирование.
Gateway - Payment Orchestrator: создать процесс пополнения.
Orchestrator (@Transactional): фиксирует topupId
Orchestrator - Provider Adapter: POST /psp/payments
Adapter - PSP: создаёт платёж
Adapter - Orchestrator: сохранение paymentId, состояние AWAITING_CONFIRMATION (+ outbox).

