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
Пополнение:
Client App - API Gateway: POST /topups (пополнение) + Idempotency-Key
API Gateway: проверка ключа в Redis Idempotency Store - проксирование
API Gateway - Payment Orchestrator: создать процесс пополнения.
Payment Orchestrator (@Transactional): фиксирует topupId + дополнительно записываем доменное событие в таблицу outbox той же БД и в той же транзакции
Payment Orchestrator - Provider Adapter: POST /psp/payments
Provider Adapter - PSP: создаёт платёж
Provider Adapter - Payment Orchestrator: сохранение paymentId, состояние AWAITING_CONFIRMATION + дополнительно записываем доменное событие в таблицу outbox той же БД и в той же транзакции
PSP - Provider Adapter (webhook): финальный статус SUCCEEDED/FAILED + подпись webhook-запроса
Provider Adapter: верифицирует подпись - сообщает Оркестратору.
Payment Orchestrator (логика SAGA):
- если SUCCEEDED - команда в Accounts & Ledger Service «зачислить пополнение» (requestId=topupId).
- если FAILED → состояние FAILED (+ в таблицу outbox), уведомление клиенту.
Accounts & Ledger Service (@Transactional, SERIALIZABLE):
- дедупликация по requestId;
- двойные проводки: кредит «кошелёк пользователя», дебет «внешний счёт»;
- вызов Fee Service (если есть комиссия PSP);
- outbox TopupCredited, коммит.
Outbox - Kafka - History Service: события применяются, история обновляется.
Payment Orchestrator: фиксирует итог (SAGA=COMPLETED) и делает результат доступным через публичное REST-API за Gateway

Отказы:
если Accounts & Ledger Service временно недоступен - настроить повторы операций с управляемой политикой: сколько раз пробовать, какие исключения считать «временными», что делать после исчерпания попыток (Spring Retry);
если деньги в PSP списались, но Accounts & Ledger Service не зачислил - Payment Orchestrator инициирует refund

Перевод между пользователями:
