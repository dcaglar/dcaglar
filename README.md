# 🛒 ecommerce-platform-kotlin

A **modular**, **event-driven**, and **resilient** eCommerce backend prototype built with **Kotlin** and **Spring Boot**, demonstrating how to design a high-throughput system (like Amazon or bol.com) using **Domain-Driven Design (DDD)** and **Hexagonal Architecture**.

> 🚧 Currently focused on the `payment-service` module. Other modules (like order, wallet, and shipment) are planned for future development.

---

## 📌 Overview




This project simulates a real-world multi-seller eCommerce platform where:

- A single order may contain products from multiple sellers.
- Each seller must be paid independently.
- Payment flow must handle failures, retries, and PSP timeouts robustly.
- All communication is decoupled using Kafka events.
- Observability and fault tolerance are built-in from day one.

---

## 🔍 Why This Project Exists

- Showcase scalable architecture choices in high-volume systems.
- Demonstrate mastery of **DDD**, **modularity**, **event choreography**, and **resilience patterns**.
- Enable others to contribute and learn by building well-structured components.

---

## ✅ Current Focus: `payment-service`

Handles the full lifecycle of payment processing for multi-seller orders:

### 🌐 Responsibilities

- Generate and persist `Payment` and multiple `PaymentOrder`s (one per seller).
- Create outbox events for Kafka: `payment_order_created`.
- Consume `payment_order_created` events and process via a mock PSP.
- Retry failed payments with backoff (via Redis).
- Schedule delayed status checks.
- Emit follow-up events: `payment_order_succeeded`, `retry_requested`, `status_check_scheduled`, etc.
- Gracefully recover Redis ID state on startup.

---

## 🧱 Architecture Principles

### ✅ Domain-Driven Design (DDD)
- Clear separation of `domain`, `application`, `adapter`, and `config` layers.
- Domain logic is isolated and testable, with all IO abstracted via ports.

### ✅ Hexagonal Architecture
- Adapters implement ports and isolate external dependencies (e.g., database, Redis, Kafka).
- Prevents domain leakage and encourages modular evolution.

### ✅ Event-Driven Communication
- Kafka events drive all workflows.
- Events are wrapped in a custom `EventEnvelope` with traceability built-in (`traceId`, `parentEventId`).

### ✅ Observability
- Structured JSON logs via `logstash-logback-encoder`.
- `MDC` context propagation.
- Metrics with Prometheus (Micrometer).
- Events include traceability metadata for end-to-end correlation.

### ✅ Resilience Patterns
- Redis ZSet for short-term retry queue (transient PSP failures).
- PostgreSQL + scheduler for long-term status checks.
- Retry, backoff, DLQ support.
- Simulated PSP supports slow responses, timeouts, hard failures, pending states.

---

## 🔩 Tech Stack

| Component | Technology |
|----------|------------|
| Language | Kotlin (JDK 21) |
| Framework | Spring Boot 3.x |
| Messaging | Kafka |
| DB | PostgreSQL + JPA |
| Caching/Retry | Redis |
| Auth | Keycloak (OAuth2) |
| Logging | Logback + JSON + MDC |
| Observability | Prometheus + Micrometer |
| Infra Simulation | Testcontainers (Redis, Kafka) |

---

## 📦 Modules (Maven Multi-Module Layout)

| Module | Status | Description |
|--------|--------|-------------|
| `payment-service` | ✅ Active | Multi-seller payment orchestration |
| `common` | ✅ Active | Shared contracts, envelope, logging |
| `order-service` | 🕒 Planned | Will emit order created events |
| `wallet-service` | 🕒 Planned | Tracks balances per seller |
| `shipment-service` | 🕒 Planned | Handles delivery coordination |

---

## 🚧 Roadmap

> See [`docs/user-stories.md`](./docs/user-stories.md) for a breakdown of backlog stories and priorities.

Key items:
- ⏱ Improve retry observability
- 🧪 Write integration tests for `PaymentOrderExecutor`
- 🧾 Add Elasticsearch read model for querying payment status
- 🛡 Harden failure flows & DLQ
- 🔐 Enforce OAuth2 in all endpoints
- 🔁 Wire up status check job for PSP polling
- 📤 Build wallet & shipment dummy services to simulate downstream effects

---

## 🧪 Testing Strategy

- Unit tests at domain & mapper level.
- Integration tests with real Redis & Kafka via Testcontainers.
- Outbox dispatching & retry scheduler are testable via event assertions.

---

## ✍️ Contributing

This repo is currently in active design and implementation.

If you want to contribute:
1. Read the `docs/architecture.md` (coming soon)
2. Check open issues or user stories
3. Fork + PR + include test coverage

---

## ✉️ Contact

This project is maintained by [@dogancaglar](https://github.com/dogancaglar).

---

## 📜 License

MIT License (c) Doğan Çağlar
