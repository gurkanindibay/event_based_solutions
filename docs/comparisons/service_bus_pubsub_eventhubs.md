# Azure Service Bus Pub/Sub vs Event Hubs  
A Structured Technical Comparison Document

---

## 1. Overview

Azure Service Bus Topics and Azure Event Hubs are both messaging services, but they serve **fundamentally different purposes**.  
This document provides a structured, technical, and practical comparison for architects and engineers.

---

## 2. Purpose / Ideal Use Cases

### 2.1 Service Bus (Topics & Subscriptions)
- Enterprise-grade messaging system  
- Used for reliable inter-service communication  
- Supports **Pub/Sub** natively  
- Designed for workflows, business events, and commands  
- Guarantees message delivery

**Use when:**
- Microservices exchange business events  
- You need guaranteed processing  
- You need filtering and per-subscriber message copies  
- You want dead-lettering, retries, or transactions  

---

### 2.2 Event Hubs
- Distributed streaming platform (Kafka-like)  
- Built for **high-throughput telemetry ingestion**  
- Optimized for millions of events per second  
- For streaming analytics and real-time pipelines  

**Use when:**
- Collecting logs, telemetry, IoT data  
- Integrating with Spark, Databricks, Azure Stream Analytics  
- Needing partitioned, ordered event streams  

---

## 3. Message Delivery Model Comparison

| Feature | Service Bus Topics | Event Hubs |
|--------|--------------------|------------|
| Pub/Sub | **Yes** | **Not traditional** (consumers read from stream) |
| Subscriber model | Each subscription gets a **copy** | Consumers read at **different offsets** |
| Ordering | Guaranteed per subscription | Guaranteed per partition |
| Retention | Until consumed or TTL | Time-based retention (7+ days) |
| Replay messages | No | Yes (by resetting offset) |

### 3.1 Inner Architecture & Mechanism

| Feature | Service Bus | Event Hubs |
|---------|-------------|------------|
| **Broker State** | **Stateful Broker**<br>Tracks state of *each* message (Active, Locked, Completed). | **Stateless Broker**<br>Log-based. Does not track consumer state per message. |
| **Cursor Management** | **Server-Side Cursor**<br>Broker knows what you've read. | **Client-Side Cursor**<br>Consumer (or Checkpoint Store) tracks offset. |
| **Consumption Model** | **Competing Consumers**<br>Multiple consumers on one subscription share the load (1 message = 1 consumer). | **Partitioned Consumers**<br>Consumers take ownership of partitions (1 partition = 1 consumer). |
| **Message Deletion** | **Destructive Read**<br>Message is deleted after `Complete()`. | **Non-Destructive Read**<br>Events remain until retention period expires. |

---

## 4. Throughput & Scale

| Feature | Service Bus | Event Hubs |
|---------|-------------|------------|
| Throughput | Medium | Very high |
| Event rate | Thousands/sec | Millions/sec |
| Consumers | Limited | Massive scale |
| Latency | Low | Very low |
| Typical scenarios | Business events / commands | Streaming ingestion |

---

## 5. Features Comparison

### 5.1 Enterprise Messaging
| Capability | Service Bus | Event Hubs |
|-----------|-------------|------------|
| Dead-Letter Queue | ✔️ | ❌ |
| Transactions | ✔️ | ❌ |
| Sessions | ✔️ | ❌ |
| Peek-lock | ✔️ | ❌ |
| Duplicate detection | ✔️ | ❌ |
| Ordering guarantee | ✔️ (per subscription) | ✔️ (per partition) |

---

### 5.2 Streaming & Analytics
| Capability | Service Bus | Event Hubs |
|-----------|-------------|------------|
| Kafka-like partitions | ❌ | ✔️ |
| Offset-based reading | ❌ | ✔️ |
| Replay events | ❌ | ✔️ |
| Checkpointing | ❌ | ✔️ |
| Integrations with Spark, Databricks | Limited | ✔️ Excellent |

---

## 6. Protocols & Behaviors

| Feature | Service Bus | Event Hubs |
|---------|-------------|------------|
| AMQP | ✔️ | ✔️ |
| HTTP | ✔️ | ❌ |
| Filters / SQL rules | ✔️ | ❌ |
| Batch sending | ✔️ | ✔️ |
| Partitioning | Basic | Advanced |

---

## 7. Scenarios & Use Cases

### 7.1 Distinct Scenarios (Clear Winners)

| Scenario | Best Choice | Why? |
|----------|:-----------:|------|
| **Order Processing** | **Service Bus** | Requires transactions, exactly-once delivery, and dead-lettering for failed orders. |
| **IoT Telemetry** | **Event Hubs** | Millions of small sensor readings/sec; occasional loss is acceptable; need to replay history. |
| **User Activity Tracking** | **Event Hubs** | Clickstreams, page views, and logs are high volume and need stream analytics. |
| **Workflow Orchestration** | **Service Bus** | Long-running processes (Sagas) need state management, deferred messages, and cancellation. |
| **Financial Transactions** | **Service Bus** | Money transfers require ACID-like guarantees and zero data loss. |
| **Log Aggregation** | **Event Hubs** | Centralizing logs from thousands of servers for Splunk/ElasticSearch. |

### 7.2 Common / Overlapping Scenarios

Sometimes both services *could* work, but one is usually better optimized.

#### **Scenario: Microservices Integration**
*   **Service Bus:** Best for **Command/Control** patterns. Service A tells Service B to "Do X". If B is down, the message waits.
*   **Event Hubs:** Best for **Notification/State Change** patterns. Service A says "X happened". Service B, C, and D react to the stream of changes.

#### **Scenario: Data Ingestion for Database**
*   **Service Bus:** Good if every message represents a critical row that must not be lost (e.g., banking ledger).
*   **Event Hubs:** Good if you are buffering massive data to dump into a Data Lake or Data Warehouse (e.g., via Capture feature).

---

## 8. Mental Model

### 7.1 Service Bus
> A **reliable enterprise mailbox system** where every subscriber gets its own inbox and messages remain until processed.

### 7.2 Event Hubs
> A **distributed event log stream**, similar to Kafka, where consumers read using offsets and can replay messages.

---

## 8. When to Choose What

### Choose **Service Bus Topics** when you need:
- Business events with reliable processing  
- Multiple subscribers receiving **individual copies**  
- Filtering rules for subscribers  
- Dead-lettering, retries, transactions  
- Microservices communication  

---

### Choose **Event Hubs** when you need:
- High-volume telemetry ingestion  
- Real-time analytics pipelines  
- Kafka-like event streaming  
- Replay and offset-based processing  
- Millions of events per second  

### 8.1 The "Value of a Message" Heuristic
> **Can we say: "If each message matters → Service Bus. If message loss is acceptable → Event Hubs"?**

**Yes, largely.** This is a great rule of thumb, with a small nuance:

*   **Service Bus (High Value / Zero Loss):**
    *   Every message is critical (e.g., a payment, an order).
    *   The system is designed to **stop and retry** if a single message fails.
    *   *Philosophy:* "I cannot proceed until this specific message is handled."

*   **Event Hubs (High Volume / Tolerable Loss):**
    *   The **stream** or **aggregate** is critical, but individual data points might be disposable.
    *   If one temperature reading is lost, the next one comes in 1 second.
    *   *Philosophy:* "Keep the pipe moving; don't block millions of events for one bad data point."

---

## 9. One-Sentence Summary

**Service Bus Topics = Enterprise Pub/Sub messaging**  
**Event Hubs = High-throughput streaming ingestion (Kafka-style)**

---

## 10. Diagram

            ┌───────────────────────┐
            │  Service Bus Topic     │
            │   (Pub/Sub Model)      │
            └───────┬─────┬─────────┘
                    │     │
         ┌──────────▼──┐ ┌▼───────────┐
         │Subscription A│ │Subscription B│
         └──────────────┘ └────────────┘
   (Each subscriber gets its own copy)


      ┌────────────────────────────┐
      │        Event Hub           │
      │ (Kafka-like Stream Log)    │
      └──────────────┬────────────┘
                     │
      ┌──────────────▼─────────────┐
      │ Multiple Consumers (Offsets)│
      └─────────────────────────────┘
