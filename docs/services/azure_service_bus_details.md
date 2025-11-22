# Azure Service Bus Detailed Reference

## 1. Overview
Azure Service Bus is a fully managed enterprise message broker with message queues and publish-subscribe topics. It is used to decouple applications and services.

- **Primary Use Case:** High-value enterprise messaging, order processing, financial transactions.
- **Protocol Support:** AMQP, SBMP, HTTP.

## 2. Core Concepts

### Queues
- **Model:** Point-to-point communication.
- **Behavior:** Sender sends a message; Receiver pulls it. Once processed, the message is removed.
- **Benefit:** Load leveling (producer produces faster than consumer can process).

### Topics and Subscriptions
- **Model:** Publish/Subscribe.
- **Behavior:** Sender sends to a Topic. Multiple Subscriptions can exist. Each subscription gets a copy of the message (potentially filtered).
- **Filters:** SQL Filters or Correlation Filters determine which messages end up in which subscription.

## 2.1. Queue vs Topic Usage Guidelines

### When to Use Queues (Point-to-Point)

#### ✅ Use Queues When:
- **Single Consumer Processing:** Only one consumer should process each message
- **Load Distribution:** Multiple consumers competing for work (competing consumer pattern)
- **Command Processing:** Executing specific actions or commands
- **Work Queue Pattern:** Background job processing
- **Order Processing:** Sequential processing by single consumer type
- **Resource-Intensive Tasks:** Tasks that should not be duplicated

#### Queue Use Cases:
```
📋 Order Processing System
Producer: E-commerce website
Consumer: Order fulfillment service
Reason: Each order should be processed exactly once

📋 Image Processing Queue
Producer: Upload service
Consumer: Image resize workers (multiple instances)
Reason: Only one worker should process each image

📋 Email Queue
Producer: Application events
Consumer: Email service
Reason: Each email should be sent exactly once

📋 Payment Processing
Producer: Checkout service
Consumer: Payment processor
Reason: Critical that payments aren't duplicated
```

### When to Use Topics (Publish-Subscribe)

#### ✅ Use Topics When:
- **Multiple Consumer Types:** Different services need the same message
- **Event Broadcasting:** Notifying multiple systems about events
- **Fan-Out Pattern:** One message triggers multiple workflows
- **Audit/Logging:** Multiple systems need to log the same event
- **Microservices Communication:** Event-driven architecture
- **Business Event Propagation:** Domain events affecting multiple bounded contexts

#### Topic Use Cases:
```
📢 User Registration Event
Publisher: User service
Subscribers: 
  - Email service (welcome email)
  - Analytics service (track signup)
  - CRM service (create lead)
  - Notification service (admin alert)
Reason: Single event triggers multiple workflows

📢 Order Status Changed
Publisher: Order service
Subscribers:
  - Inventory service (update stock)
  - Shipping service (prepare shipment)
  - Customer service (send notification)
  - Analytics service (track metrics)
Reason: Multiple systems react to order changes

📢 Payment Completed
Publisher: Payment service
Subscribers:
  - Order service (update status)
  - Invoice service (generate receipt)
  - Loyalty service (award points)
  - Fraud service (analyze patterns)
Reason: Payment completion affects multiple domains
```

### Decision Matrix

| Scenario | Queue | Topic | Reason |
|----------|-------|-------|--------|
| Process each message exactly once | ✅ | ❌ | Queues ensure single consumption |
| Multiple services need same data | ❌ | ✅ | Topics broadcast to all subscribers |
| Load balancing across workers | ✅ | ❌ | Queue distributes work among consumers |
| Event-driven microservices | ❌ | ✅ | Topics enable loose coupling |
| Critical business transactions | ✅ | ❌ | Queues prevent duplicate processing |
| Audit trail requirements | ❌ | ✅ | Topics allow multiple audit consumers |
| Background job processing | ✅ | ❌ | Queues manage work distribution |
| System integration events | ❌ | ✅ | Topics enable system decoupling |

### Hybrid Patterns

#### Queue → Topic Chain
```
1. Critical processing via Queue (ensures single processing)
2. Success event published to Topic (notifies other systems)

Example:
[Order] → Queue → [Payment Processor] → Topic → [Multiple Services]
```

#### Topic → Queue Fan-Out
```
1. Event published to Topic
2. Each subscriber has its own Queue for reliable processing

Example:
[User Event] → Topic → Multiple Queues → [Dedicated Workers]
```

### Performance Considerations

#### Queues
- **Throughput:** Higher for single consumer scenarios
- **Latency:** Lower overhead, direct message delivery
- **Scaling:** Scale by adding competing consumers
- **Resource Usage:** Lower memory footprint

#### Topics
- **Throughput:** Depends on number of subscriptions
- **Latency:** Slight overhead for message copying
- **Scaling:** Scale by managing subscription filters
- **Resource Usage:** Higher due to message duplication

### Cost Implications

#### Queues
- **Messages:** Pay per message operation
- **Storage:** Pay for message storage duration
- **Connections:** Fewer connections needed

#### Topics
- **Messages:** Pay per message × number of subscriptions
- **Storage:** Higher storage costs due to copies
- **Connections:** More connections for multiple subscribers

### Filter Usage in Topics

#### SQL Filters
```sql
-- Route by message properties
Region = 'US' AND Priority > 5

-- Route by custom properties
EventType = 'OrderCreated' AND Amount > 1000

-- Complex routing logic
CustomerTier IN ('Gold', 'Platinum') AND EventSource = 'WebApp'
```

#### Correlation Filters (Performance Optimized)
```
-- Simple property matching (faster than SQL)
CorrelationId = 'user-events'
Label = 'high-priority'
ContentType = 'application/json'
```

## 3. Advanced Features

### Dead-Letter Queues (DLQ)
A sub-queue for holding messages that cannot be delivered or processed.
- **Reasons:** Max delivery count exceeded, TTL expired, filter evaluation exceptions, or explicit dead-lettering by consumer.

### Message Sessions (FIFO)
- **Function:** Guarantees ordered processing of unbounded sequences of related messages.
- **Mechanism:** Messages with the same `SessionId` are locked by a single receiver.

### Transactions
Supports atomic operations. You can send a message, delete a message, and update state within a single transaction scope.

### Duplicate Detection
Service Bus can automatically remove duplicate messages sent within a specific time window based on `MessageId`.

### Scheduled Delivery
Messages can be sent to a queue/topic but remain invisible to consumers until a specific scheduled time.

## 4. Data Integration Model: Push-Pull (Hybrid)

Service Bus follows a **hybrid push-pull** delivery model:

### Publisher Side (Push)
- **Senders actively push messages** to Service Bus queues or topics.
- Messages are sent via AMQP, SBMP, or HTTP protocols.
- Messages are immediately persisted in the queue/topic.
- Sender receives acknowledgment once the message is stored.

### Consumer Side (Pull with Push Characteristics)
- **Receivers pull messages**, but with push-like behavior through long polling.
- Two receive modes:
  - **Peek-Lock (default):** Message is locked for processing, must be explicitly completed or abandoned.
  - **Receive-and-Delete:** Message is immediately removed upon receipt.
- Service Bus supports **message sessions** for ordered, FIFO processing.
- Consumers can use **event-driven listeners** (SDK abstractions) that continuously poll but appear like push.

### Benefits
- **Guaranteed delivery:** Messages persist until explicitly completed.
- **Order preservation:** Sessions guarantee FIFO within a session.
- **Load leveling:** Queue acts as buffer between fast producers and slow consumers.
- **Transactional support:** Atomic operations across multiple messages.

### Considerations
- Consumers must actively receive messages (even with SDK abstractions).
- Message lock duration limits processing time (renewable).
- Dead-letter queue requires explicit handling and monitoring.
- More complex than Event Grid but offers stronger guarantees.

## 5. Tiers
- **Basic:** Queues only, no topics.
- **Standard:** Queues & Topics, variable latency, shared resources.
- **Premium:** Dedicated resources (Messaging Units), predictable latency, support for large messages (up to 100 MB), VNET integration.

## 6. Best Practices
- **Peek-Lock vs Receive-and-Delete:** Always use Peek-Lock for reliability (default). It allows abandoning the message if processing fails.
- **Prefetch Count:** Increase prefetch count for high throughput scenarios to reduce round-trips.
- **Exception Handling:** Handle `MessageLockLostException` (processing took too long) and `ServiceBusException` (transient errors).
