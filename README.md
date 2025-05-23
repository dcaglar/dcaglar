Dogan-Amsterdam based

# 📦 ecommerce-platform-kotlin
```
# 📦 ecommerce-platform-kotlin

```mermaid
flowchart TD
  %% User placing order
  U[User] --> OrderService[Order Service]
  OrderService --> PC[PaymentController]
  PC --> POS[PaymentOrderService]
  POS --> DB[(PostgreSQL\nPayment + OutboxEvent)]

  %% Outbox → Kafka
  DB --> OD[Outbox Dispatcher\n(@Scheduled)]
  OD --> KP[Kafka Producer]
  KP --> Kafka1[Kafka\ntopic: payment_order_created_queue\nEventEnvelope<PaymentOrderCreated>]

  %% Kafka consumer logic
  Kafka1 --> POE[PaymentOrderExecutor\n(Kafka Consumer)]
  POE --> PSP[PSP Client]
  POE --> KafkaSuccess[Kafka\ntopic: payment_order_success\nEventEnvelope<PaymentOrderSucceeded>]

  %% Retry and Status Check
  POE --> RedisRetry[Redis Retry ZSet\ntransient failures]
  POE --> RedisSchedule[Redis Scheduled ZSet\npending status]

  RedisRetry -->|poll due retry| POE
  RedisSchedule --> Dispatcher[DueStatusCheckDispatcher\n(@Scheduled)]
  Dispatcher --> DB
  Dispatcher --> KafkaDue[Kafka\ntopic: due_payment_status_check_topic\nEventEnvelope<DuePaymentOrderStatusCheck>]

  KafkaDue --> StatusExecutor[ScheduledStatusCheckExecutor\n(Kafka Consumer)]
  StatusExecutor --> PSP

  %% Observability
  subgraph Observability
    Filebeat[Filebeat/Logstash]
    ES[Elasticsearch]
    Kibana[Kibana Dashboard]
  end

  PC --> Filebeat
  POS --> Filebeat
  POE --> Filebeat
  Dispatcher --> Filebeat
  StatusExecutor --> Filebeat
  Filebeat --> ES --> Kibana

  %% Styling
  classDef db fill=#fff2b2,stroke=#b2a700
  classDef kafka fill=#f2e0ff,stroke=#a600ff
  classDef redis fill=#ddffdd,stroke=#008000
  classDef scheduled fill=#ffe0a0,stroke=#cc8800
  classDef observability fill=#eeeeee,stroke=#444444
  class DB db
  class Kafka1,KafkaSuccess,KafkaDue kafka
  class RedisRetry,RedisSchedule redis
  class OD,Dispatcher scheduled
  class Filebeat,ES,Kibana observability
```


This repository is a work-in-progress **ecommerce platform prototype**, designed with a modular, domain-driven, event-oriented architecture using **Spring Boot + Kotlin**.

> 🔍 **Goal**: Demonstrate architectural design choices and trade-offs for high-throughput systems like Amazon or bol.com. The focus is currently on the **payment-service**.



                           ┌────────────────────────────┐
                           │     PaymentController       │
                           │  (Receives REST request)    │
                           └────────────┬───────────────┘
                                        │
                                        ▼
                           ┌────────────────────────────┐
                           │   PaymentOrderService       │
                           │  - Create PaymentOrder      │
                           │  - Persist to DB            │
                           │  - Save OutboxEvent<T>      │
                           └────────────┬───────────────┘
                                        │
                 ┌─────────────────────▼─────────────────────┐
                 │           Database                         │
                 │  - payment_order                           │
                 │  - outbox_event<EventEnvelope<T>>          │
                 └─────────────────────┬─────────────────────┘
                                       │
                    (OutboxDispatcherScheduler @Scheduled)
                                       │
                                       ▼
                  ┌────────────────────────────────────────┐
                  │ Kafka Publisher (from OutboxEvent<T>)  │
                  │ Topic: `payment_order_created_queue`   │
                  │ Payload: EventEnvelope<PaymentOrderCreated> │
                  └────────────────────────────────────────┘
                                       │
                                       ▼
            ┌────────────────────────────────────────────────────────────┐
            │ PaymentOrderExecutor (Kafka Consumer)                      │
            │  - Deserializes envelope<EventEnvelope<PaymentOrderCreated>>│
            │  - Calls PSP (mock/real)                                   │
            │  - Delegates to domain: PaymentOrder                       │
            │  - Based on result:                                        │
            │      - Retry? → use RetryQueuePort<PaymentOrderRetryRequested>│
            │      - Pending? → use RetryQueuePort<PaymentOrderStatusScheduled>│
            │      - Success? → publish success                          │
            └────────────────────────────────────────────────────────────┘
                  │                                   │
                  ▼                                   ▼
   ┌────────────────────────────┐       ┌────────────────────────────────────────────┐
   │ Redis ZSet: Short Retry    │       │ Redis ZSet: Long-Term PSP Status Scheduling│
   │ Key: payment:retry         │       │ Key: payment:status:schedule               │
   │ Purpose: retry transient   │       │ Purpose: decouple DB I/O, delay scheduling │
   └────────────┬───────────────┘       └──────────────┬──────────────────────────────┘
                │                                       │
    pollDueRetries() (short-term)              DueStatusCheckRequestDispatcherJob
                                                  (runs every N seconds):
                                                  - poll ZSet (score <= now)
                                                  - persist due to DB
                                                  - publish EventEnvelope<DuePaymentOrderStatusCheck>
                                                        │
                                                        ▼
                          Kafka Topic: `due_payment_status_check_topic`
                                                        │
                                                        ▼
           ┌────────────────────────────────────────────────────────────┐
           │ ScheduledPaymentStatusCheckExecutor (Kafka Consumer)       │
           │  - Consumes envelope<EventEnvelope<DuePaymentOrderStatusCheck>> │
           │  - Calls PSP again                                         │
           │  - Updates order status                                    │
           │  - Publishes success/failure event                         │
           └────────────────────────────────────────────────────────────┘


──────────────────────────────────────────────────────────────────────────────
📊 Observability & Logging: Filebeat + ELK
──────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────┐
│ MDC Context (eventId, traceId, aggregateId)                │
│ Structured logs using logstash-logback-encoder             │
└────────────────────────────┬───────────────────────────────┘
                             ▼
                   ┌────────────────────────┐
                   │      Filebeat          │
                   └────────────┬───────────┘
                                ▼
                   ┌────────────────────────┐
                   │     Elasticsearch      │
                   └────────────┬───────────┘
                                ▼
                   ┌────────────────────────┐
                   │        Kibana          │
                   │   - Trace logs         │
                   │   - Visualize flow     │
                   └────────────────────────┘

---

## ✅ Highlights

- **Modular architecture** with decoupled domains (starting with `payment-service`)
- Designed using **Domain-Driven Design (DDD)** and **Hexagonal Architecture**
- Implements **event-driven communication** using **Kafka**
- Production-level patterns for:
  - Fault tolerance (retry, backoff, DLQ)
  - Observability (MDC, traceId, structured logging)
  - Testability and decoupling

---

## 💳 Payment Service

The payment service handles the core payment flow for an ecommerce platform with many sellers:

- Supports **multi-seller split payments** per order
- Integrates with a mock **PSP (Payment Service Provider)**
- Kafka events model the full flow: order creation → payment processing → status updates
- Uses **Redis** and **PostgreSQL** for retry queues and delayed processing

### 🔁 Retry & Scheduling

- **Short-term retries** for transient PSP failures are managed via **Redis ZSet**.
- **Long-term PSP status checks** are persisted to DB and dispatched periodically by a scheduled job.

---

### 🔬 Production Simulation & Fault Injection

To test the resilience of this system, I’ve implemented a **mock PSP (Payment Service Provider)** that simulates real-world failure scenarios:

- 🐌 **Slow responses**
- 💥 **Timeouts**
- ❌ **Hard PSP failures**
- 🔁 **Retryable errors**
- ❓ **Ambiguous “pending” states**

This allows me to **analyze system behavior under pressure**, **identify bottlenecks**, and **fine-tune retry, scheduling, and observability strategies**.

---

### 📈 Observability & Resilience

- All events are wrapped in a custom `EventEnvelope` structure with trace IDs and parent-child causality for full traceability.
- Logs are structured in **JSON** and shipped via **Filebeat → Elasticsearch → Kibana**.
- Kafka-based flows are **fully decoupled**, with backoff, retries, and dead-letter logic modeled around real-world patterns.

---

## 🛠️ Tech Stack

- Kotlin + Spring Boot 3.x
- Kafka (event backbone)
- PostgreSQL (JPA)
- Redis (retry queues)
- Keycloak (OAuth2 authentication)
- ELK stack (logging and observability)
- Maven multi-module layout

---

## 🧩 Modules (WIP)

- `payment-service` ✅
- `order-service` (planned)
- `wallet-service` (planned)
- `shipment-service` (planned)

---

## 📌 Running Locally

This project uses Docker Compose for infra (Postgres, Redis, Kafka, Keycloak, Elasticsearch, Kibana):

```bash
docker-compose up -d
