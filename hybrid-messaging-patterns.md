# Hybrid Messaging Patterns

Combining Azure Service Bus, Event Hubs, and Event Grid to build resilient, observable, and scalable messaging platforms.

## Table of Contents
1. [CQRS with Event Sourcing Pattern](#1-cqrs-with-event-sourcing-pattern)
2. [Saga Pattern for Distributed Transactions](#2-saga-pattern-for-distributed-transactions)
3. [Event Streaming with Command Processing](#3-event-streaming-with-command-processing)
4. [IoT Event Processing Pipeline](#4-iot-event-processing-pipeline)
5. [Multi-Channel Event Distribution](#5-multi-channel-event-distribution)
6. [Event-Driven Data Synchronization](#6-event-driven-data-synchronization)
7. [Real-time Analytics with Historical Processing](#7-real-time-analytics-with-historical-processing)
8. [Microservices Event Mesh](#8-microservices-event-mesh)

---

## 1. CQRS with Event Sourcing Pattern

**Use Case**: Separate the write model from the read model while keeping a complete audit trail of every change.

**Services Used**:
- **Service Bus**: queue commands and orchestrate retries.
- **Event Hubs**: append-only store for emitted domain events.
- **Event Grid**: fan-out events to projection handlers and notifications.
- **Cosmos DB**: materialized read models.

**Event Flow**:
1. Command lands in a Service Bus queue.
2. Command handler runs domain logic and produces events.
3. Events persist to Event Hubs and simultaneously publish to Event Grid.
4. Projection handlers update Cosmos DB; queries use the read model.

**Architecture Diagram**:
```text
                                      +------------------+
                                      |     Client       |
                                      +--------+---------+
                                               |
                                               v
                                      +--------+---------+
                                      |   Service Bus    |
                                      |     (Queue)      |
                                      +--------+---------+
                                               |
                                               v
+------------------+                  +--------+---------+
|   Event Hubs     | <----------------+ Command Handler  |
|  (Event Store)   |                  +--------+---------+
+--------+---------+                           |
         |                                     v
         |                            +--------+---------+
         |                            |   Domain Model   |
         |                            +------------------+
         |
         v
+--------+---------+                  +------------------+
|    Event Grid    +----------------->| Projection Func  |
+--------+---------+                  +--------+---------+
         |                                     |
         v                                     v
+--------+---------+                  +--------+---------+
|   Notification   |                  |    Cosmos DB     |
|     Handler      |                  |   (Read Model)   |
+------------------+                  +------------------+
```
**Implementation Highlights**:
- Stamp each event with aggregate id/version to support deterministic replay from Event Hubs.
- Projection functions triggered by Event Grid keep read models eventually consistent.
- Dead-letter queues capture failed commands for later inspection.

**Benefits**:
- Clear read/write separation, replayable audit trail, and scalable projections.

---

## 2. Saga Pattern for Distributed Transactions

**Use Case**: Coordinate long-running, cross-service workflows without locking or two-phase commits.

**Services Used**:
- **Service Bus**: commands, compensations, and reliable message routing.
- **Event Grid**: signals state transitions to external observers.
- **Durable Functions** or an orchestration service: maintain saga state and issue new commands.
- **Event Hubs**: log saga progress for off-line troubleshooting.

**Event Flow**:
- A saga orchestrator publishes a start event to Event Grid.
- Each service reacts by processing the command via Service Bus and replies with success/failure events.
- Failures trigger compensation commands that roll back previously completed steps.
- Event Hubs captures every saga step for replay, and Event Grid broadcasts the final outcome.

**Architecture Diagram**:
```text
                                  +-------------------+
                                  | Saga Orchestrator |
                                  +---------+---------+
                                            |
                                            v
                                  +---------+---------+
                                  |    Event Grid     |
                                  +----+---------+----+
                                       |         |
                      +----------------+         +----------------+
                      |                                           |
                      v                                           v
             +--------+--------+                         +--------+--------+
             |   Service Bus   |                         |   Service Bus   |
             |    (Topic A)    |                         |    (Topic B)    |
             +--------+--------+                         +--------+--------+
                      |                                           |
                      v                                           v
             +--------+--------+                         +--------+--------+
             | Inventory Svc   |                         |  Payment Svc    |
             +--------+--------+                         +--------+--------+
                      |                                           |
            +---------+---------+                       +---------+---------+
            | Success / Failure |                       | Success / Failure |
            +---------+---------+                       +---------+---------+
                      |                                           |
                      v                                           v
             +--------+--------+                         +--------+--------+
             |  Compensation   |------------------------>|   Service Bus   |
             |    Command      |                         | (Reply Channel) |
             +-----------------+                         +--------+--------+
                                                                  |
                                                                  v
                                                         +--------+--------+
                                                         | Saga Orchestrator|
                                                         +------------------+
```
**Implementation Highlights**:
```csharp
[Function("ReserveInventory")]
public Task Run([ActivityTrigger] Order order) => _inventoryService.Reserve(order);
```
- Status events include correlation and sequence numbers to keep compensating in order.
- Service Bus sessions enforce the sequence per saga instance.

**Benefits**:
- Observable state transitions, deterministic compensations, and audit logs for compliance.

---

## 3. Event Streaming with Command Processing

**Use Case**: Pair high-throughput event ingestion with imperative commands to act on detected conditions.

**Services Used**:
- **Event Hubs**: telemetry/event ingestion and partitioned consumption.
- **Service Bus**: command/control plane for reactive actions.
- **Event Grid**: fan-out of significant events to dashboards or automation logic.

**Event Flow**:
1. Event Hubs receives telemetry streams from upstream systems.
2. Stream processors evaluate data and post commands to Service Bus when thresholds are breached.
3. Command handlers execute business actions (e.g., scale service, send notification) and emit follow-up events via Event Grid.

**Architecture Diagram**:
```text
+----------------+       +------------------+       +------------------+
| Telemetry Src  +------>|    Event Hubs    +------>| Stream Processor |
+----------------+       +--------+---------+       +--------+---------+
                                  |                          |
                                  | (Capture)                | (Threshold Breached)
                                  v                          v
                         +--------+---------+       +--------+---------+
                         |    Data Lake     |       |   Service Bus    |
                         +------------------+       |     (Queue)      |
                                                    +--------+---------+
                                                             |
                                                             v
+----------------+       +------------------+       +--------+---------+
|   Dashboard    |<------+    Event Grid    |<------+  Action Handler  |
+----------------+       +--------+---------+       +------------------+
                                  |
                                  v
                         +--------+---------+
                         |    Automation    |
                         +------------------+
```
**Implementation Highlights**:
- Keep event metadata (tenant, device, priority) consistent between Event Hubs and Service Bus messages to correlate workflows.
- Use Event Grid filters to send only critical alerts to downstream systems.
- Service Bus sessions guard command ordering for a given target.

**Benefits**:
- Real-time monitoring married to reliable command execution, enabling actionable insights without losing throughput.

---

## 4. IoT Event Processing Pipeline

**Use Case**: Ingest, analyze, and act on IoT telemetry in near real time while keeping command paths reliable.

**Services Used**:
- **IoT Hub → Event Hubs**: ingest device telemetry.
- **Service Bus**: device command queue and telemetry-backed control messages.
- **Event Grid**: alerting layer that notifies workflows, dashboards, or partners.
- **Azure Stream Analytics** / Functions: evaluate telemetry and trigger commands.

**Event Flow**:
- Devices push telemetry to IoT Hub; routed into Event Hubs for stream processing.
- Stream jobs detect anomalies and enqueue commands into Service Bus (adjust settings, reboot devices).
- Event Grid broadcasts alerts to support teams or automation runbooks.
- Commands return status events that trace back to the originating telemetry.

**Architecture Diagram**:
```text
                                     +-------------+
                                     | IoT Devices |
                                     +------+------+
                                            |
                                            v
                                     +------+------+
                                     |   IoT Hub   |
                                     +------+------+
                                            |
                       +--------------------+--------------------+
                       |                                         |
                       v                                         v
                +------+------+                           +------+------+
                |  Event Hubs |                           | Service Bus |
                +------+------+                           |  (Commands) |
                       |                                  +------+------+
                       v                                         ^
                +------+------+                                  |
                | Stream Job  +----------------------------------+
                +------+------+
                       |
                       v
                +------+------+
                |  Event Grid |
                +------+------+
                       |
          +------------+------------+
          |                         |
          v                         v
   +------+------+           +------+------+
   |  Logic App  |           |  Ops Team   |
   +-------------+           +-------------+
```
**Implementation Highlights**:
- Enable Event Hubs capture to archive raw telemetry for ML training.
- Configure Service Bus sessions to keep per-device commands in order.
- Event Grid subscriptions deliver alerts to Logic Apps, Teams, or third-party connectors.

**Benefits**:
- Unified telemetry ingestion, reliable command delivery, and out-of-the-box alert routing for IoT fleets.

---

## 5. Multi-Channel Event Distribution

**Use Case**: Broadcast domain events to multiple consumer types—analytics, partner APIs, downstream services—without duplicates.

**Services Used**:
- **Event Grid**: central event fabric with filtering and fan-out.
- **Service Bus topics**: durable pipelines for mission-critical systems.
- **Event Hubs**: ingest events for analytics, replay, and storage.
- **WebHooks/Logic Apps**: integrate with partner systems and automation.

**Event Flow**:
- Domain services publish canonical events to Event Grid.
- Service Bus subscriptions attached via filters consume events with reliable delivery.
- Event Hubs receives the same stream for archival, analytics, and ML.
- Event Grid also routes partner notifications through webhooks or Logic Apps.

**Architecture Diagram**:
```text
                                  +----------------+
                                  | Domain Service |
                                  +-------+--------+
                                          |
                                          v
                                  +-------+--------+
                                  |   Event Grid   |
                                  +-------+--------+
                                          |
             +----------------------------+-----------------------------+
             |                            |                             |
             v                            v                             v
    +--------+--------+          +--------+--------+           +--------+--------+
    |   Service Bus   |          |   Event Hubs    |           |    WebHooks     |
    |     (Topic)     |          |    (Capture)    |           |  (Logic Apps)   |
    +--------+--------+          +--------+--------+           +--------+--------+
             |                            |                             |
             v                            v                             v
    +--------+--------+          +--------+--------+           +--------+--------+
    | Critical System |          |    Analytics    |           | Partner System  |
    +-----------------+          |    (PowerBI)    |           +-----------------+
                                 +-----------------+
```
**Implementation Highlights**:
- Use Event Grid filters to deliver tailored payloads to different consumer groups.
- Attach retry-enabled Service Bus queues to Event Grid for prioritized consumers.
- Event Hubs capture ensures raw data is always available for reprocessing.

**Benefits**:
- Flexible fan-out, durable consumption for critical workloads, and simultaneous analytics ingestion.

---

## 6. Event-Driven Data Synchronization

**Use Case**: Keep polyglot datastores (SQL, NoSQL, search indexes) synchronized without synchronous coupling.

**Services Used**:
- **Event Grid**: change-data notifications from primary systems.
- **Service Bus**: drives datastore-specific commands within transactions.
- **Event Hubs**: records every change for reconciliation and auditing.
- **Data Factory/Synapse**: orchestrate batch refreshes triggered by Event Grid.

**Event Flow**:
- Source systems publish change events to Event Grid whenever a record mutates.
- Change handlers enqueue targeted Service Bus commands to update each downstream store.
- A reconciliation job reads Event Hubs to compare expected vs actual states and reissues commands if needed.

**Architecture Diagram**:
```text
                                  +---------------+
                                  | Source System |
                                  +-------+-------+
                                          |
                                          v
                                  +-------+-------+
                                  |   Event Grid  |
                                  +-------+-------+
                                          |
                     +--------------------+--------------------+
                     |                                         |
                     v                                         v
            +--------+--------+                       +--------+--------+
            |   Service Bus   |                       |   Event Hubs    |
            |     (Queue)     |                       | (Change Stream) |
            +--------+--------+                       +--------+--------+
                     |                                         |
                     v                                         v
            +--------+--------+                       +--------+--------+
            | Change Handler  |                       | Reconciliation  |
            +--------+--------+                       |      Job        |
                     |                                +--------+--------+
                     v                                         |
            +--------+--------+                                |
            | Target Datastore| <------------------------------+
            +-----------------+
```
**Implementation Highlights**:
- Event Grid filters route only the subset of events relevant to each store (e.g., customer vs product updates).
- Service Bus transactions ensure each store sees either the complete update or none at all.
- Event Hubs streams persist the authoritative change feed for drift analysis.

**Benefits**:
- Eventually consistent syncing with built-in audit and reconciliation paths.

---

## 7. Real-time Analytics with Historical Processing

**Use Case**: Pair low-latency alerting with historical trend analysis and model training.

**Services Used**:
- **Event Hubs**: ingest streaming telemetry.
- **Event Grid**: triggers analytics jobs, notebooks, and workflows when anomalies appear.
- **Service Bus**: throttles ingestion/backpressure commands to prevent downstream overload.
- **Synapse/Data Lake**: store historical data for ML and reporting.

**Event Flow**:
- Event Hubs feeds stream processors; anomalies publish events to Event Grid.
- Event Grid triggers notebooks, Logic Apps, or downstream dashboards for immediate action.
- Service Bus commands adjust sampling, batching, or throttling based on system health.
- Event Hubs capture mirrors the stream to Data Lake for long-term storage.

**Architecture Diagram**:
```text
                                  +----------------+
                                  | Telemetry Src  |
                                  +-------+--------+
                                          |
                                          v
                                  +-------+--------+
                                  |   Event Hubs   |
                                  +-------+--------+
                                          |
                     +--------------------+--------------------+
                     |                                         |
                     v                                         v
            +--------+--------+                       +--------+--------+
            | Stream Processor|                       |   Event Hubs    |
            +--------+--------+                       |    (Capture)    |
                     |                                +--------+--------+
                     v                                         |
            +--------+--------+                                v
            |   Event Grid    |                       +--------+--------+
            +--------+--------+                       |    Data Lake    |
                     |                                +--------+--------+
        +------------+------------+                            |
        |            |            |                            v
        v            v            v                   +--------+--------+
 +------+-----+ +----+-----+ +----+-----+             | Synapse / ML    |
 |  Notebook  | |Logic App | |Dashboard |             +-----------------+
 +------------+ +----------+ +----------+
        |
        v
 +------+-----+
 | Service Bus|
 | (Control)  |
 +------+-----+
        |
        v
 +------+-----+
 | Telemetry  |
 | Source     |
 +------------+
```
**Implementation Highlights**:
- Event Grid orchestration lets you respond without polling.
- Service Bus handles backpressure by issuing control commands that adjust upstream producers.
- Historical Event Hubs capture is replayable for retraining ML models.

**Benefits**:
- Balanced real-time insights and durable archives for compliance and learning.

---

## 8. Microservices Event Mesh

**Use Case**: Establish a domain event mesh where each microservice can publish, subscribe, and replay without tight coupling.

**Services Used**:
- **Event Grid**: domain-level event fabric with filtering and routing.
- **Service Bus topics/subscriptions**: durable endpoints for each microservice.
- **Event Hubs**: telemetry and observability stream for tracing event paths.

**Event Flow**:
- Microservices publish domain events to Event Grid.
- Event Grid delivers filtered events to Service Bus subscriptions owned by interested services.
- Services consume messages, apply idempotent updates, and optionally publish new events back into the mesh.
- Event Hubs captures the stream for observability and debugging.

**Architecture Diagram**:
```text
                                  +----------------+
                                  | Microservice A |
                                  +-------+--------+
                                          |
                                          v
                                  +-------+--------+
                                  |   Event Grid   |
                                  +-------+--------+
                                          |
                     +--------------------+--------------------+
                     |                                         |
                     v                                         v
            +--------+--------+                       +--------+--------+
            |   Service Bus   |                       |   Event Hubs    |
            |     (Topic)     |                       | (Observability) |
            +--------+--------+                       +--------+--------+
                     |                                         |
                     v                                         v
            +--------+--------+                       +--------+--------+
            | Microservice B  |                       |   Log Analytics |
            +--------+--------+                       +-----------------+
                     |
                     v
            +--------+--------+
            |   Event Grid    |
            +-----------------+
```
**Implementation Highlights**:
- Service Bus dead-lettering captures events that cannot be processed immediately.
- Event Hubs stores correlation ids so you can reconstruct flow paths.
- Auto-forwarding and sessions in Service Bus keep complex workflows ordered.

**Benefits**:
- Observable, decoupled microservices with reliable delivery and replay capabilities.
