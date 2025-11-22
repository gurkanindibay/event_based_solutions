# Azure Event Services Reference Guide

This document consolidates the full conversation and provides a structured reference for Azure Event Grid, Event Hubs, and Service Bus, including comparisons, architecture diagrams, decision guides, and real-world examples.

---

## 1. Overview of Event Services

### Event Grid
- **Purpose:** Event routing and notifications.
- **Event Type:** Discrete events (something happened).
- **Delivery Model:** Push-based to subscribers.
- **Best for:** Serverless triggers, resource changes.
- **Key Traits:** Lightweight, integrates with Azure services, supports filtering.

### Event Hubs
- **Purpose:** High-throughput streaming ingestion.
- **Event Type:** Continuous data stream (telemetry/logs).
- **Delivery Model:** Pull-based from partitions.
- **Best for:** Real-time analytics, IoT telemetry.
- **Key Traits:** Partitioned, replayable, scales to millions of events/sec.

### Service Bus
- **Purpose:** Enterprise messaging, commands, and workflows.
- **Event Type:** Messages and tasks.
- **Delivery Model:** Push-based to consumers.
- **Best for:** Ordered processing, reliable delivery, transactional workflows.
- **Key Traits:** Queues and topics, transactions, sessions, dead-letter queues.

---

## 2. Architecture Diagrams

### 2.1 Combined Architecture
```
+----------------------+            +----------------+            +---------------------+
|  External Systems    |            |   IoT Devices  |            |    Azure Services   |
| (Payments, Webhooks) |            | (Sensors, Apps)|            | (Storage, ACR, etc) |
+----------+-----------+            +-------+--------+            +----------+----------+
           |                                |                                |
           v                                v                                v
   +-----------------+             +----------------+               +------------------+
   |  Event Grid     |<------------|  Edge/Cloud    |  (resource   ) |  Resource Events |
   |  (notifications) |  publishes |  Gateway/Proxy |--------------->|  (Blob created,  |
   +-----------------+   alerts    +----------------+               |   VM change...)   |
           |                                                              +-----------+
           | Routes to                                                
+----------+-----------+----------------------+------------------+-----------------------+
|  Subscriptions:      |                      |                  |                       |
|  - Function(s)       |                      |                  |                       |
|  - Logic Apps        |                      |                  |                       |
+----------+-----------+                      |                  |                       |
           |                                  |                  |                       |
           v                                  v                  v                       |
   +------------------+                +----------------+   +----------------+           |
   | Service Bus      |                |   Event Hubs   |   |  StreamMgr /   |           |
   | (Commands, tasks)|<---commands----| (telemetry)    |-->|  Analytics     |           |
   | Queues & Topics  |                | partitions,    |   |  (Databricks,  |           |
   | (ordered, DLQ)   |                | replay, capture|   |   Synapse,     |           |
   +------------------+                +----------------+   |   StreamAnal.) |           |
           |                                   |             +----------------+           |
  Consumers/Workers                                |                   |                 |
  (order worker, retry logic)                      |                   v                 |
           |                                       |             +----------------+      |
           v                                       v             | Cold Storage /  |      |
   +----------------------+                +----------------+    | Data Lake (ADLS)|<-----+
   | Microservices /      |   reads from   | Batch Consumers |    +------------------+
   | Backend APIs         |   partitions   | (ETL jobs)      |
   +----------------------+                +-----------------+
```

### 2.2 Event Grid
```
Event Source (Blob/ARM/App) --> Event Grid Topic --> Filters & Subscriptions --> Subscriber (Function/Logic App/Webhook)
```

### 2.3 Event Hubs
```
Producers (IoT/App) --> Event Hubs (Partitions) --> Consumer Groups --> Stream Processing (Spark/Stream Analytics/Custom)
```

### 2.4 Service Bus
```
Producers --> Service Bus (Queue or Topic+Subscriptions) --> Consumer(s) (workers) --> Ack / DLQ / Retry / Sessions
```

---

## 3. Decision Tree / Heuristics

- Telemetry, streaming → **Event Hubs**
- Simple notifications, resource events → **Event Grid**
- Reliable commands, ordered workflows → **Service Bus**
- Replay required → **Event Hubs**
- Transactional processing → **Service Bus Premium**

---

## 4. Real-World Patterns & Examples

### 4.1 Telemetry & Analytics (Event Hubs)
- IoT Devices → Event Hubs → Stream Analytics → ADLS → Synapse

### 4.2 Reactive Automation (Event Grid)
- Blob Created → Event Grid → Azure Function → Database Update

### 4.3 Enterprise Messaging (Service Bus)
- API → Service Bus Queue → Worker Service → DLQ on failure

### 4.4 Combined Scenario
- Event Hubs detects anomaly → Event Grid publishes alert → Function writes command into Service Bus → Maintenance workflow executes.

---

## 5. Best Practices

### Security
- Managed Identities for service-to-service authentication
- Private Endpoints for sensitive workloads
- RBAC, short-lived SAS tokens if used

### Scaling
- Event Hubs: plan partitions upfront
- Service Bus: Premium tier for predictable latency
- Event Grid: auto-scaling; monitor subscriber endpoints for throttling

### Reliability
- Service Bus: Dead-letter queues, sessions
- Event Hubs: Checkpointing for consumers
- Event Grid: Implement idempotent event handlers

---

## 6. Architecture Template

Components:
- **Ingress Layer:** App Gateway / Front Door
- **Notification Layer:** Event Grid topics and subscriptions
- **Streaming Layer:** Event Hubs with partitions
- **Messaging & Workflow Layer:** Service Bus queues/topics
- **Processing Layer:** Azure Functions, containerized microservices
- **Storage & Analytics:** ADLS Gen2 / Blob Storage, Synapse / Databricks
- **Operations:** Azure Monitor, Log Analytics, Alerts
- **Security:** Key Vault + Managed Identities

Flow Example:
User/API → Service Bus (commands) → Worker → Event Hubs (telemetry) → Stream Processing → Event Grid (alerts) → Functions/Notifications → Service Bus (maintenance commands)

---

## 7. Summary Comparison Table

| Feature | Event Grid | Event Hubs | Service Bus |
|---------|------------|------------|-------------|
| Primary Purpose | Event routing / notifications | High-throughput streaming ingestion | Enterprise messaging (commands, workflows) |
| Event Type | Discrete events | Continuous data stream | Commands/messages/tasks |
| Delivery Model | Push | Pull | Push |
| Throughput | Low–Medium | Extremely high | Medium |
| Message Size | Small (<1MB) | Large/frequent | Up to 256 KB / 1 MB (premium) |
| Retention | 24 hours | Days–years | Until processed |
| Replay | No | Yes | No (except DLQ) |
| Advanced Messaging | No | No | Yes: sessions, FIFO, transactions |
| Filtering | Yes (per subscriber) | No | Yes (topic subscriptions) |
| Best For | Integrations, serverless triggers | Telemetry, logs, IoT ingestion | Ordered workflows, microservices commands |

---

## 8. Data Integration Models Comparison

| Aspect | Event Grid | Event Hubs | Service Bus |
|--------|------------|------------|-------------|
| **Model** | Push-Push | Push-Pull | Push-Pull (Hybrid) |
| **Publisher** | Pushes events to Event Grid | Pushes events to partitions | Pushes messages to queue/topic |
| **Consumer** | Receives pushed events via HTTP | Pulls events from partitions | Pulls messages (with lock/session) |
| **Delivery Control** | Event Grid controls delivery | Consumer controls read pace | Consumer controls receive pace |
| **Endpoint Requirement** | HTTP endpoint required | No endpoint (consumer connects) | No endpoint (receiver connects) |
| **Replay Capability** | No | Yes (consumer resets offset) | Limited (DLQ only) |
| **Ordering** | No guarantees | Per-partition ordering | Session-based FIFO |
| **Backpressure** | Subscriber must handle | Consumer controls rate | Consumer controls rate + lock renewal |
| **State Management** | Stateless (Event Grid manages) | Consumer manages offsets | Service Bus manages message state |
| **Best Use Case** | Reactive automation, webhooks | High-volume streaming, analytics | Reliable command processing, workflows |

### Push-Push (Event Grid)
**How it works:** Event sources push events → Event Grid receives and routes → Event Grid pushes to subscribers.

**Advantages:**
- Minimal consumer complexity (just expose an endpoint)
- Low latency for event delivery
- No polling overhead
- Built-in retry and dead-lettering

**Disadvantages:**
- Requires publicly accessible or private endpoints
- No replay capability
- Subscriber must handle bursts

### Push-Pull (Event Hubs)
**How it works:** Producers push events → Event Hubs persists to partitions → Consumers pull at their pace.

**Advantages:**
- High throughput and scalability
- Consumers control processing rate
- Full replay capability
- Multiple independent consumer groups

**Disadvantages:**
- More complex consumer implementation
- Requires checkpoint management
- Continuous polling overhead

### Push-Pull Hybrid (Service Bus)
**How it works:** Senders push messages → Service Bus queues → Receivers pull with locks and sessions.

**Advantages:**
- Guaranteed delivery and ordering (sessions)
- Built-in dead-letter handling
- Transactional support
- Load leveling between producers/consumers

**Disadvantages:**
- More complex than Event Grid
- Lock timeout management required
- No replay (except DLQ)
- Higher latency than Event Hubs

---

## 9. Event Grid Advanced Features

### Message Schemas
Event Grid supports multiple schemas for event data:
- **Event Grid Schema:** The default schema with properties like `subject`, `eventType`, `eventTime`, `id`, and `data`.
- **CloudEvents Schema:** An open standard (CNCF) for describing event data, enabling interoperability across different cloud providers and platforms.
- **Custom Input Schema:** Allows mapping custom JSON fields to Event Grid requirements, useful when you cannot change the event publisher's format.

### Retry & Retry Policies
When Event Grid fails to deliver an event to an endpoint, it retries based on a schedule:
- **Schedule:** It uses an exponential backoff policy (e.g., 10s, 30s, 1m, 5m, 10m, 30m, 1h) up to 24 hours.
- **Randomization:** A small randomization factor is added to avoid thundering herd issues.
- **Configurable Policies:**
    - **Max Delivery Attempts:** Configurable between 1 and 30.
    - **Event Time-to-Live (TTL):** Configurable duration (e.g., 1 minute to 1440 minutes) after which the event is dropped if not delivered.

### Dead Letter Events
If an event cannot be delivered after all retry attempts or the TTL expires:
- **Dead Lettering:** You can configure a storage account (blob container) to store these undelivered events.
- **Purpose:** Allows for later analysis, debugging, and manual reconciliation of missed events.
- **Content:** The dead-lettered blob contains the original event payload along with the error reason for the failure.

### Access Control & Permissions
Event Grid divides access between resource (management) operations and data-plane delivery:
- **System/operations level:** Roles such as `Owner`, `Contributor`, `EventGrid Contributor`, and `EventGrid Event Subscription Contributor` control who can create or update topics, event subscriptions, filters, and delivery settings (`Microsoft.EventGrid/*` operations).
- **User/data level:** Publishers use `EventGrid Data Sender` to push events (SAS tokens, managed identity, or keys), and subscribers rely on delivery endpoints that may also require `EventGrid Data Receiver` or custom authentication to consume events safely.
- **Scope:** Assign roles at subscription, resource group, or individual topic level to restrict who can configure routes versus who can send or receive payloads.

### Certificates & TLS
Event Grid requires validated HTTPS endpoints for data delivery and endpoint validation must complete over TLS:
- **Trusted CA:** Delivery endpoints must present certificates issued by public/trusted root CAs (including Azure-managed certificates); self-signed certs are not accepted unless you establish a private endpoint with a custom root capable of being trusted by Event Grid.
- **Hostname match:** The certificate's subject must match the DNS name used during subscription creation, and wildcard certificates are supported when the wildcard covers the specified endpoint domain.
- **Automatic renewal:** Use Azure App Service-managed certificates or Azure Front Door to manage renewal, preventing delivery breaks when certificates expire.
- **Mutual TLS:** Not required for standard Event Grid delivery; if you implement client cert authentication at the endpoint you must ensure Event Grid's requests are allowed through your network controls.

### Endpoint Validation
When an event subscription targets a webhook or HTTP endpoint, Event Grid performs validation before sending business events:
- **Validation event:** Event Grid sends a `Microsoft.EventGrid.SubscriptionValidationEvent` payload that includes a `validationCode` and `validationUrl`.
- **Expected response:** The endpoint must respond within 30 seconds with an HTTP 200 and either echo the `validationCode` in the body or follow the `validationUrl` to confirm ownership.
- **Failed validation:** If the endpoint never acknowledges the validation event or responds with an error, the subscription stays in a `PendingValidation` state and delivery never starts; retry attempts are made but eventually the subscription is disabled.
- **Automation tip:** Functions/Logic Apps listening for Event Grid events should explicitly handle the validation event (check `eventType` and return the code) before processing normal events.

### Delivery Response Handling
When Event Grid receives a `400 (Bad Request)` or `413 (Request Entity Too Large)` during delivery:
- **No retries:** These status codes are treated as permanent failures. Event Grid still makes that single delivery attempt, records the failure, and will not retry that event again even though the subscription stays active.
- **Failure tracking:** The failed delivery is recorded in metrics/logs and, if dead-lettering is enabled, the event is written there for inspection; otherwise the payload is eventually dropped once the TTL expires.
- **Subscription footprint:** The subscription stays enabled so future deliveries continue, but repeated 400/413 responses should trigger troubleshooting of payload size limits and validation logic.
- **Payload guidance:** Split oversized payloads, trim unnecessary properties, and ensure subscribers parse the schema properly to avoid 400 responses.

---

**End of Reference Guide**
"

canmore.create_textdoc(content)

