# Event Grid Messaging Patterns

Azure Event Grid is a fully managed event routing service that enables event-driven architectures using a publish-subscribe model.

## Table of Contents
1. [Simple Event Publishing Pattern](#1-simple-event-publishing-pattern)
2. [Fan-Out Pattern](#2-fan-out-pattern)
3. [Event Routing Pattern](#3-event-routing-pattern)
4. [Reactive Programming Pattern](#4-reactive-programming-pattern)
5. [Event Aggregation Pattern](#5-event-aggregation-pattern)
6. [Webhook Integration Pattern](#6-webhook-integration-pattern)
7. [Cross-Region Event Replication](#7-cross-region-event-replication)
8. [Event Filtering and Transformation](#8-event-filtering-and-transformation)
9. [Domain Events Pattern](#9-domain-events-pattern)

---

## 1. Simple Event Publishing Pattern

**Use Case**: Single publisher broadcasting events to multiple subscribers.

```
┌─────────────┐
│   Event     │
│  Publisher  │
│  (Source)   │
└──────┬──────┘
       │ Publishes Event
       ▼
┌─────────────────────────────┐
│     Azure Event Grid        │
│    (Event Distribution)     │
└──┬─────────┬──────────┬────┘
   │         │          │
   │         │          │
   ▼         ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│Subscriber│Subscriber│Subscriber│
│    1    │ │    2    │ │    3    │
└────────┘ └────────┘ └────────┘
```

**Characteristics**:
- One-to-many event distribution
- Loose coupling between publisher and subscribers
- Automatic retry and dead-lettering
- At-least-once delivery guarantee

**Implementation Example**:
```csharp
// Publisher
var client = new EventGridPublisherClient(
    new Uri(endpoint),
    new AzureKeyCredential(key));

var evt = new EventGridEvent(
    subject: "orders/new",
    eventType: "OrderCreated",
    dataVersion: "1.0",
    data: new { OrderId = 123, Amount = 99.99 });

await client.SendEventAsync(evt);
```

**Best Practices**:
- Use consistent event schemas
- Include correlation IDs
- Implement idempotent subscribers
- Monitor dead-letter queues

---

## 2. Fan-Out Pattern

**Use Case**: Distribute a single event to multiple independent processing pipelines.

```
                    ┌──────────────────┐
                    │  Event Source    │
                    │  (Blob Storage)  │
                    └────────┬─────────┘
                             │
                             │ Blob Created Event
                             ▼
                    ┌──────────────────┐
                    │  Event Grid      │
                    │  Topic           │
                    └─┬──────┬───────┬─┘
                      │      │       │
           ┌──────────┘      │       └──────────┐
           │                 │                  │
           ▼                 ▼                  ▼
    ┌────────────┐    ┌────────────┐    ┌────────────┐
    │ Azure      │    │ Azure      │    │ Logic      │
    │ Function   │    │ Function   │    │ App        │
    │ (Thumbnail)│    │ (Metadata) │    │ (Notify)   │
    └────────────┘    └────────────┘    └────────────┘
         │                  │                  │
         ▼                  ▼                  ▼
    ┌────────┐        ┌────────┐        ┌────────┐
    │ CDN    │        │Cosmos  │        │ Email  │
    │        │        │  DB    │        │Service │
    └────────┘        └────────┘        └────────┘
```

**Characteristics**:
- Parallel processing of same event
- Each subscriber processes independently
- No coordination required between subscribers
- High throughput and scalability

**Best Practices**:
- Design for idempotency
- Use appropriate filtering
- Implement independent error handling
- Monitor each pipeline separately

---

## 3. Event Routing Pattern

**Use Case**: Route events to different subscribers based on event properties.

```
┌─────────────────┐
│  Application    │
│  (Multi-Event)  │
└────────┬────────┘
         │
         │ Multiple Event Types
         ▼
┌────────────────────────────┐
│     Event Grid Topic       │
│    (Intelligent Router)    │
└─┬────────┬────────┬───────┘
  │        │        │
  │ Filter │ Filter │ Filter
  │ Type=A │ Type=B │ Type=C
  │        │        │
  ▼        ▼        ▼
┌────┐  ┌────┐  ┌────┐
│Sub │  │Sub │  │Sub │
│ A  │  │ B  │  │ C  │
└────┘  └────┘  └────┘
  │        │        │
  ▼        ▼        ▼
┌────┐  ┌────┐  ┌────┐
│Proc│  │Proc│  │Proc│
│ A  │  │ B  │  │ C  │
└────┘  └────┘  └────┘

Advanced Filtering:
┌──────────────────────────────┐
│ Filter Options:              │
│ • Event Type                 │
│ • Subject (prefix/suffix)    │
│ • Data Field Values          │
│ • Advanced Operators         │
└──────────────────────────────┘
```

**Characteristics**:
- Content-based routing
- Multiple filtering criteria
- No coding required for routing logic
- Dynamic subscription management

**Implementation Example**:
```csharp
// Advanced Filter Example
{
    "filter": {
        "includedEventTypes": ["OrderCreated", "OrderUpdated"],
        "advancedFilters": [
            {
                "operatorType": "NumberGreaterThan",
                "key": "data.amount",
                "value": 1000
            },
            {
                "operatorType": "StringIn",
                "key": "data.region",
                "values": ["US-East", "US-West"]
            }
        ],
        "subjectBeginsWith": "/orders/priority"
    }
}
```

**Best Practices**:
- Define clear routing rules
- Use appropriate filter types
- Document routing logic
- Test filters thoroughly

---

## 4. Reactive Programming Pattern

**Use Case**: Chain multiple services in response to events automatically.

```
Event Chain Flow:

┌─────────┐      Event 1       ┌─────────┐      Event 2       ┌─────────┐
│Service A│ ─────────────────> │Service B│ ─────────────────> │Service C│
└─────────┘  OrderCreated      └─────────┘  PaymentProcessed  └─────────┘
     │                              │                              │
     │                              │                              │
     ▼                              ▼                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        Event Grid Topic                              │
└──────────────────────────────────────────────────────────────────────┘
     │                              │                              │
     ▼                              ▼                              ▼
┌─────────┐                    ┌─────────┐                    ┌─────────┐
│Inventory│                    │ Billing │                    │Shipping │
│ Service │                    │ Service │                    │ Service │
└─────────┘                    └─────────┘                    └─────────┘


Example: Order Processing Workflow
┌──────────────────────────────────────────────────────────────┐
│ Step 1: Order Created                                        │
│   ↓                                                          │
│ Step 2: Inventory Reserved (triggers InventoryReserved)     │
│   ↓                                                          │
│ Step 3: Payment Processed (triggers PaymentCompleted)       │
│   ↓                                                          │
│ Step 4: Shipment Created (triggers ShipmentInitiated)       │
│   ↓                                                          │
│ Step 5: Notification Sent (triggers NotificationSent)       │
└──────────────────────────────────────────────────────────────┘
```

**Best Practices**:
- Design for eventual consistency
- Implement compensation logic
- Use correlation IDs throughout chain
- Monitor the entire workflow

---

## 5. Event Aggregation Pattern

**Use Case**: Collect and aggregate events from multiple sources.

```
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│Source 1 │  │Source 2 │  │Source 3 │  │Source N │
└────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
     │            │            │            │
     │ Events     │ Events     │ Events     │ Events
     ▼            ▼            ▼            ▼
┌─────────────────────────────────────────────────┐
│          Event Grid Domain/Topic                │
│         (Central Event Collection)              │
└────────────────────┬────────────────────────────┘
                     │
                     │ Aggregated Events
                     ▼
          ┌──────────────────────┐
          │  Event Processor     │
          │  (Aggregation Logic) │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Analytics/Storage   │
          └──────────────────────┘
```

**Implementation Example**:
```csharp
// Aggregator Function
[Function("EventAggregator")]
public async Task AggregateEvents(
    [EventGridTrigger] EventGridEvent[] events)
{
    var groupedEvents = events
        .GroupBy(e => e.Subject.Split('/')[0])
        .ToDictionary(g => g.Key, g => g.ToList());
    
    foreach (var group in groupedEvents)
    {
        await _analyticsService.ProcessBatchAsync(
            group.Key, 
            group.Value);
    }
}
```

---

## 6. Webhook Integration Pattern

**Use Case**: Integrate external systems via webhooks.

```
┌────────────────┐
│ Azure Service  │
│ (Event Source) │
└───────┬────────┘
        │
        │ Event
        ▼
┌────────────────────┐
│   Event Grid       │
└─────────┬──────────┘
          │
          │ Webhook POST
          ▼
┌────────────────────┐
│  External System   │
│  (Webhook Handler) │
└─────────┬──────────┘
          │
          │ 200 OK / Retry
          ▼
┌────────────────────┐
│  Business Logic    │
└────────────────────┘

Webhook Validation Flow:
┌────────────┐                    ┌────────────────┐
│Event Grid  │ ── Validation ──> │ Webhook        │
│            │    Request         │ Endpoint       │
│            │ <── Validation ── │                │
│            │     Response       │                │
│            │                    │                │
│            │ ── Event Data ──> │                │
│            │ <── 200 OK ─────  │                │
└────────────┘                    └────────────────┘
```

**Implementation Example**:
```csharp
[ApiController]
[Route("api/webhooks")]
public class WebhookController : ControllerBase
{
    [HttpPost("eventgrid")]
    public async Task<IActionResult> HandleEventGridWebhook(
        [FromBody] EventGridEvent[] events)
    {
        // Handle validation
        foreach (var evt in events)
        {
            if (evt.EventType == "Microsoft.EventGrid.SubscriptionValidationEvent")
            {
                var data = evt.Data.ToObjectFromJson<SubscriptionValidationEventData>();
                return Ok(new { validationResponse = data.ValidationCode });
            }
            
            // Process actual events
            await ProcessEventAsync(evt);
        }
        
        return Ok();
    }
}
```

---

## 7. Cross-Region Event Replication

**Use Case**: Replicate events across regions for disaster recovery.

```
Region 1 (Primary):                    Region 2 (Secondary):
┌─────────────┐                        ┌─────────────┐
│   Primary   │                        │  Secondary  │
│   Service   │                        │   Service   │
└──────┬──────┘                        └──────┬──────┘
       │                                      │
       │ Publishes                            │ Receives
       ▼                                      ▼
┌─────────────┐       Replication      ┌─────────────┐
│ Event Grid  │ ──────────────────────>│ Event Grid  │
│  (Region 1) │                        │  (Region 2) │
└──────┬──────┘                        └──────┬──────┘
       │                                      │
       │                                      │
       ▼                                      ▼
┌─────────────┐                        ┌─────────────┐
│  Local      │                        │  Local      │
│ Subscribers │                        │ Subscribers │
└─────────────┘                        └─────────────┘
```

---

## 8. Event Filtering and Transformation

**Use Case**: Filter and transform events before delivery.

```
┌──────────────┐
│Event Publisher│
└──────┬───────┘
       │
       │ Raw Events
       ▼
┌─────────────────────────────────┐
│      Event Grid Topic           │
│                                 │
│  ┌───────────────────────────┐ │
│  │  Filtering Layer          │ │
│  │  • Subject Filters        │ │
│  │  • Event Type Filters     │ │
│  │  • Advanced Filters       │ │
│  └───────────────────────────┘ │
└────┬──────────┬─────────┬──────┘
     │          │         │
     │ Filtered │ Filtered│ Filtered
     ▼          ▼         ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│  Sub 1  │ │  Sub 2  │ │  Sub 3  │
│ (High)  │ │ (Medium)│ │  (Low)  │
└─────────┘ └─────────┘ └─────────┘
```

---

## 9. Domain Events Pattern

**Use Case**: Implement Domain-Driven Design with Event Grid.

```
┌────────────────────────────────────────────────┐
│           Bounded Context (Domain)             │
│                                                │
│  ┌──────────────┐        ┌─────────────────┐  │
│  │   Aggregate  │ ──────>│  Domain Event   │  │
│  │   Root       │ Raises │  (OrderPlaced)  │  │
│  └──────────────┘        └────────┬────────┘  │
│                                   │            │
└───────────────────────────────────┼────────────┘
                                    │
                                    │ Publishes
                                    ▼
                    ┌───────────────────────────┐
                    │     Event Grid Topic      │
                    │  (Domain Event Bus)       │
                    └──┬──────────┬──────────┬──┘
                       │          │          │
          ┌────────────┘          │          └────────────┐
          │                       │                       │
          ▼                       ▼                       ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ Bounded Context  │   │ Bounded Context  │   │ Bounded Context  │
│   (Inventory)    │   │   (Shipping)     │   │  (Notification)  │
└──────────────────┘   └──────────────────┘   └──────────────────┘
```

**Implementation Example**:
```csharp
// Domain Event
public class OrderPlacedEvent : DomainEvent
{
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public DateTime PlacedAt { get; set; }
}

// Aggregate Root
public class Order : AggregateRoot
{
    public void PlaceOrder()
    {
        // Business logic
        ValidateOrder();
        
        // Raise domain event
        AddDomainEvent(new OrderPlacedEvent
        {
            OrderId = this.Id,
            CustomerId = this.CustomerId,
            Items = this.Items,
            PlacedAt = DateTime.UtcNow
        });
    }
}

// Event Publisher
public class DomainEventPublisher
{
    private readonly EventGridPublisherClient _client;
    
    public async Task PublishAsync(DomainEvent domainEvent)
    {
        var evt = new EventGridEvent(
            subject: $"{domainEvent.GetType().Name}/{domainEvent.AggregateId}",
            eventType: domainEvent.GetType().Name,
            dataVersion: "1.0",
            data: domainEvent);
            
        await _client.SendEventAsync(evt);
    }
}
```

**Best Practices**:
- Use ubiquitous language in event names
- Keep domain events focused and granular
- Implement event versioning strategy
- Use event sourcing when appropriate
- Maintain bounded context boundaries

---

## Summary

Event Grid patterns enable:
- **Loose Coupling**: Publishers and subscribers are decoupled
- **Scalability**: Automatic scaling and high throughput
- **Flexibility**: Easy to add/remove subscribers
- **Reliability**: Built-in retry and dead-lettering
- **Simplicity**: No infrastructure to manage

## When to Use Event Grid

✅ **Good For**:
- Event-driven architectures
- System integration
- Real-time notifications
- Serverless applications
- Microservices communication

❌ **Not Ideal For**:
- Large message payloads (>1MB)
- Guaranteed order processing
- Complex message routing logic in code
- Stream processing requirements

## Related Patterns
- See [Service Bus Patterns](service-bus-patterns.md) for guaranteed messaging
- See [Event Hubs Patterns](event-hubs-patterns.md) for high-throughput streaming
- See [Hybrid Patterns](hybrid-messaging-patterns.md) for combining multiple Azure messaging services