# Microservices Spring Kotlin

A production-ready distributed event-driven microservices architecture built with **Spring Boot 3+**, **Kotlin 2+**, **Apache Kafka / Azure EventHubs**, **Redis**, and **MongoDB**.

Features **zero-code automatic distributed tracing** across HTTP and Kafka event streams via **Micrometer Tracing (W3C standard)**.

---

## 📦 Microservices & Repositories

| Component | Submodule Directory | Standalone GitHub Repository | Responsibilities |
| :--- | :--- | :--- | :--- |
| **Consumer API Gateway** | `consumer-api-gateway` | [akc276/customer-api-gateway](https://github.com/akc276/customer-api-gateway) | REST API Gateway, HTTP Correlation Filter, Event Publisher, Response Listener, Scheduled Cron Poller |
| **Processor Service** | `processor-service` | [akc276/processor-service](https://github.com/akc276/processor-service) | Background Event Processor, Kafka Consumer, Response Publisher |

---

## 🏗️ Architecture & Technology Stack

- **Core Framework**: Spring Boot 3.4+ with Kotlin 2.1+
- **Event Messaging**: Azure EventHubs Emulator (Kafka API compatible)
- **Distributed Tracing**: Micrometer Tracing with Brave & W3C Trace Context (`traceparent`)
- **Database & Cache**: MongoDB, Redis
- **Containerization**: Docker Compose

---

## 🔍 End-to-End Distributed Tracing

This repository implements automatic distributed trace propagation across HTTP and Kafka event streams:

```text
Client Request (GET /hello)
   │
   ▼
[consumer-api-gateway] HTTP Entry Point (W3C Trace ID generated)
   │
   ▼ (Kafka Producer - auto-injects W3C traceparent header)
[EventHub Topic: gateway-requests]
   │
   ▼ (Kafka Consumer - auto-extracts W3C traceparent header)
[processor-service] Background Processor (Child Span with SAME Trace ID)
   │
   ▼ (Kafka Producer - auto-propagates W3C traceparent header)
[EventHub Topic: service-responses]
   │
   ▼ (Kafka Consumer - auto-extracts W3C traceparent header)
[consumer-api-gateway] Response Event Listener (SAME Trace ID)
```

### Verified Log Output (Trace ID: `6a8a72131eb3e771a29055dcc2f745bf`)

```text
1. Gateway HTTP Entry & Producer:
   2026-08-23T04:07:47.416Z INFO 1 --- [consumer-api-gateway] [main] [6a8a72131eb3e771a29055dcc2f745bf-732c20585fd313da] KafkaProducer : Instantiated an idempotent producer.

2. Processor Service Consumer & Producer:
   2026-08-23T04:07:48.102Z INFO 1 --- [processor-service]   [container#0-0-C-1] [6a8a72131eb3e771a29055dcc2f745bf-810a42b109f5e08c] RequestEventListener : Received request event from consumer-api-gateway

3. Gateway Response Event Listener:
   2026-08-23T04:07:48.620Z INFO 1 --- [consumer-api-gateway] [container#0-0-C-1] [6a8a72131eb3e771a29055dcc2f745bf-732c20585fd313da] ResponseEventListener : Received response message from processor-service
```

---

## ⏱️ Scheduled Polling Service

The Gateway includes a periodic scheduled polling service ([ScheduledPublisherService.kt](consumer-api-gateway/src/main/kotlin/com/example/gateway/scheduler/ScheduledPublisherService.kt)):
- **Immediate Startup Trigger**: Uses `@EventListener(ApplicationReadyEvent::class)` to publish an initial heartbeat upon container startup.
- **Periodic Cron Schedule**: Uses `@Scheduled(cron = "0 */5 * * * *")` to execute every 5 minutes.
- **Trace Context**: Wrapped in `Observation.createNotStarted("scheduled.heartbeat", observationRegistry)` to ensure a root W3C Trace ID is generated from the first line of execution.

---

## 🚀 Quick Start

### 1. Clone Repository (with Submodules)
```bash
git clone --recursive https://github.com/akc276/microservices-spring-kotlin.git
cd microservices-spring-kotlin
```

### 2. Start Infrastructure & Microservices via Docker Compose
```bash
docker compose up -d --build
```

### 3. Test HTTP & Event Stream Flow
```bash
curl -i http://localhost:8080/hello
```

### 4. Run Unit Test Suites
```bash
# Test Consumer API Gateway
cd consumer-api-gateway && ./gradlew test

# Test Processor Service
cd ../processor-service && ./gradlew test
```
