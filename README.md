1. Регистрация пользователя и кошелька

Client App - Auth Service: логин/регистрация, получение JWT (TLS)  
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
- outbox TopupCredited, коммит

Outbox - Kafka - History Service: события применяются, история обновляется.  
Payment Orchestrator: фиксирует итог (SAGA=COMPLETED) и делает результат доступным через публичное REST-API за Gateway  

Отказы:
- если Accounts & Ledger Service временно недоступен - настроить повторы операций с управляемой политикой: сколько раз пробовать, какие исключения считать «временными», что делать после исчерпания попыток (Spring Retry);  
- если деньги в PSP списались, но Accounts & Ledger Service не зачислил - Payment Orchestrator инициирует refund  

Перевод между пользователями:

Client App - API Gateway: POST /transfer (from=A, to=B, amount) + Idempotency-Key.  
API Gateway: проверка/фиксация идемпотентности в Redis Idempotency Store - проксирование.  
API Gateway - Accounts & Ledger Service: команда «выполнить перевод» с requestId (тот же, что в идемпотентности).  
Accounts & Ledger Service (@Transactional, SERIALIZABLE):
- блокирует счета A и B (SELECT ... FOR UPDATE) в строгом порядке;
- проверяет доступный баланс и лимиты;
- вызывает Fee Service для расчёта;
- пишет двойные проводки: дебет A (с учётом комиссии), кредит B, кредит «счёта комиссий»;
- фиксирует requestId в таблице дедупликации (UNIQUE);
- пишет событие в outbox, коммит

Outbox - Kafka - History Service: событие "перевод" применено, история обновлена.  
Accounts & Ledger Service - API Gateway - Client App: итоговый статус и новые балансы.  

3. История транзакций

Client App - API Gateway: GET /history?walletId=... (JWT).  
API Gateway - History Service: проксирует запрос  
History Service: читает из History DB (материализованное представление из событий Kafka)  
Ответ: список транзакций.  

4. Простая система комиссий

Расчёт синхронный, применение атомарно в Accounts & Ledger Service  
Вызов места расчёта:  
- при переводе между пользователями: Accounts & Ledger Service перед записью проводок вызывает Fee Service,  
- при пополнении: Payment Orchestrator может запросить комиссию провайдера и передать её в Accounts & Ledger Service.

Accounts & Ledger Service (внутри одной транзакции): включает комиссию в проводки  
Надёжность: если Fee Service временно недоступен — Retry  
Учёт комиссий: отдельный счёт для дохода в Accounts & Ledger Service, все комиссии проходят через него.  

5. Поддержка распределённых транзакций

Паттерн: локальные доменные транзакции + межсервисная SAGA (оркестровка).  

Общая схема (на примере пополнения):  
Шаг 1 (Payment Orchestrator): создать SAGA (INITIATED) - outbox событие.  
Шаг 2 (Provider Adapter): создать платёж в PSP → AWAITING_CONFIRMATION.  
Шаг 3 (Provider Adapter webhook): получить финальный статус.  
Шаг 4 (Accounts & Ledger Service): при успехе — локальная доменная транзакция с зачислением и комиссией (ACID, SERIALIZABLE).  
Каждый шаг пишет в outbox - Kafka - History.  
Если Шаг 4 не удался после успеха в PSP — выполнить refund  

Идемпотентность:  
входная — Idempotency-Key в API Gateway/Redis Idempotency Store;  
внутри шагов — requestId/eventId + UNIQUE в БД;  
потребители Kafka — таблица применённых eventId.  
Повторы:  
Spring Retry с паузой между повторными попытками выполнить операцию(backoff) на внешних вызовах;  
DLQ в Kafka для «ядовитых» сообщений; алёрты.  

Безопасность:  
секреты/ключи через Vault;  
JWT;
