Dogan-Amsterdam based

# 📦 ecommerce-platform-kotlin

This repository is a work-in-progress **ecommerce platform prototype**, designed with a modular, domain-driven, event-oriented architecture using **Spring Boot + Kotlin**.

> 🔍 **Goal**: Demonstrate architectural design choices and trade-offs for high-throughput systems like Amazon or bol.com. The focus is currently on the **payment-service**.

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
