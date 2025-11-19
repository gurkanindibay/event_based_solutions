# Azure Service Bus - Messaging Patterns with ASCII Diagrams

## Table of Contents
1. [Point-to-Point Pattern (Queue)](#1-point-to-point-pattern-queue)
2. [Competing Consumers Pattern](#2-competing-consumers-pattern)
3. [Publish-Subscribe Pattern (Topic)](#3-publish-subscribe-pattern-topic)
4. [Request-Reply Pattern](#4-request-reply-pattern)
5. [Message Sessions (FIFO) Pattern](#5-message-sessions-fifo-pattern)
6. [Priority Queue Pattern](#6-priority-queue-pattern)
7. [Dead Letter Queue Pattern](#7-dead-letter-queue-pattern)
8. [Saga Pattern with Service Bus](#8-saga-pattern-with-service-bus)
9. [Claim Check Pattern](#9-claim-check-pattern)

---

## 1. Point-to-Point Pattern (Queue)

### Description
Single producer sends messages to a queue, and a single consumer receives and processes each message exactly once.

### Use Cases
- Order processing
- Payment transactions
- Email notifications
- Single-step workflows

### ASCII Diagram
```
┌──────────┐         ┌─────────────────┐         ┌──────────┐
│ Producer │ ──────> │  Service Bus    │ ──────> │ Consumer │
│          │  Send   │     Queue       │ Receive │          │
└──────────┘         └─────────────────┘         └──────────┘
                            │
                            │ Message stored
                            │ until consumed
                            ▼
                     [Msg1][Msg2][Msg3]
```

### Code Pattern
```csharp
// Producer
await sender.SendMessageAsync(new ServiceBusMessage("Order #123"));

// Consumer
ServiceBusReceivedMessage message = await receiver.ReceiveMessageAsync();
await receiver.CompleteMessageAsync(message);
```

### Characteristics
- ✅ Guaranteed delivery
- ✅ Exactly-once processing
- ✅ FIFO within partition
- ✅ Load leveling

---

## 2. Competing Consumers Pattern

### Description
Multiple consumers compete for messages from the same queue, enabling parallel processing and load distribution.

### Use Cases
- Image processing
- Video encoding
- Batch data processing
- High-throughput scenarios

### ASCII Diagram
```
                         ┌─────────────────┐
                         │  Service Bus    │
┌──────────┐            │     Queue       │
│ Producer │ ────────>  │                 │
└──────────┘   Send     │ [M1][M2][M3]    │
                         │ [M4][M5][M6]    │
                         └────┬─────┬──────┘
                              │     │
                    Compete   │     │     Compete
                         ┌────┴──┐  └────┬────┐
                         │       │       │    │
                         ▼       ▼       ▼    ▼
                    ┌────────┐ ┌────────┐ ┌────────┐
                    │Consumer│ │Consumer│ │Consumer│
                    │   #1   │ │   #2   │ │   #3   │
                    └────────┘ └────────┘ └────────┘
                         │         │         │
                         └─────────┴─────────┘
                                   │
                              Parallel Processing
```

### Code Pattern
```csharp
// Multiple consumer instances
var processor = client.CreateProcessor(queueName, new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 5,
    AutoCompleteMessages = false
});

processor.ProcessMessageAsync += async args =>
{
    await ProcessMessageAsync(args.Message);
    await args.CompleteMessageAsync(args.Message);
};
```

### Characteristics
- ✅ Horizontal scaling
- ✅ Load distribution
- ✅ Fault tolerance
- ⚠️ No message ordering across consumers

---

## 3. Publish-Subscribe Pattern (Topic)

### Description
Single publisher sends messages to a topic, and multiple subscribers independently receive copies of the message.

### Use Cases
- Event broadcasting
- User registration events
- Order status updates
- Audit logging

### ASCII Diagram
```
                               ┌─────────────────┐
                               │  Service Bus    │
┌──────────┐                  │     Topic       │
│Publisher │ ──────────────>  │                 │
└──────────┘   Publish        │  [Event Data]   │
                               └────┬────┬───┬───┘
                                    │    │   │
                       Subscription │    │   │ Subscription
                                    │    │   │
                          ┌─────────┘    │   └─────────┐
                          │              │             │
                          ▼              ▼             ▼
                   ┌────────────┐ ┌────────────┐ ┌────────────┐
                   │Subscriber  │ │Subscriber  │ │Subscriber  │
                   │   Email    │ │ Analytics  │ │   CRM      │
                   │  Service   │ │  Service   │ │  Service   │
                   └────────────┘ └────────────┘ └────────────┘
```

### Code Pattern
```csharp
// Publisher
await sender.SendMessageAsync(new ServiceBusMessage("User registered"));

// Subscriber 1 (Email Service)
var emailProcessor = client.CreateProcessor(topicName, "email-subscription");

// Subscriber 2 (Analytics Service)
var analyticsProcessor = client.CreateProcessor(topicName, "analytics-subscription");

// Subscriber 3 (CRM Service)
var crmProcessor = client.CreateProcessor(topicName, "crm-subscription");
```

### Characteristics
- ✅ Message fan-out
- ✅ Independent subscriptions
- ✅ Filtered subscriptions
- ✅ Multiple consumption models

---

## 4. Request-Reply Pattern

### Description
Request-reply pattern using two queues for bidirectional communication between services.

### Use Cases
- Synchronous-like operations in async systems
- RPC-style communication
- Service orchestration

### ASCII Diagram
```
┌──────────┐         ┌─────────────┐         ┌──────────┐
│ Client   │ ──────> │ Request     │ ──────> │ Service  │
│ (Sender) │ Request │   Queue     │ Request │(Processor│
└────┬─────┘         └─────────────┘         └────┬─────┘
     │                                             │
     │               ┌─────────────┐               │
     │ <──────────── │ Reply       │ <──────────── │
     │    Response   │   Queue     │   Response    │
     │               └─────────────┘               │
     │                                             │
     │  ReplyTo = reply-queue                     │
     │  CorrelationId = request-id                │
     └─────────────────────────────────────────────┘
```

### Code Pattern
```csharp
// Client (Request)
var request = new ServiceBusMessage("Process order #123")
{
    ReplyTo = "reply-queue",
    CorrelationId = Guid.NewGuid().ToString()
};
await requestSender.SendMessageAsync(request);

// Service (Reply)
var response = new ServiceBusMessage("Order processed")
{
    CorrelationId = request.CorrelationId
};
await replySender.SendMessageAsync(response);

// Client (Receive Reply)
var reply = await replyReceiver.ReceiveMessageAsync();
if (reply.CorrelationId == originalRequest.CorrelationId)
{
    // Handle response
}
```

### Characteristics
- ✅ Correlation tracking
- ✅ Async request-response
- ⚠️ Requires correlation logic
- ⚠️ Timeout handling needed

---

## 5. Message Sessions (FIFO) Pattern

### Description
Guarantees ordered processing of related messages using session IDs.

### Use Cases
- Shopping cart processing
- Multi-step workflows
- Transaction processing
- Conversation management

### ASCII Diagram
```
┌──────────┐         ┌─────────────────────────┐
│ Producer │ ──────> │  Service Bus Queue      │
└──────────┘         │  (Session Enabled)      │
                     │                         │
     Send with       │ ┌─────────────────────┐ │
     SessionId       │ │ Session: User-123   │ │
                     │ │ [M1][M2][M3]        │ │
                     │ └─────────────────────┘ │
                     │                         │
                     │ ┌─────────────────────┐ │
                     │ │ Session: User-456   │ │
                     │ │ [M4][M5][M6]        │ │
                     │ └─────────────────────┘ │
                     └────┬──────────────┬─────┘
                          │              │
                          ▼              ▼
                   ┌────────────┐  ┌────────────┐
                   │ Consumer   │  │ Consumer   │
                   │  #1        │  │  #2        │
                   │ (Session   │  │ (Session   │
                   │ User-123)  │  │ User-456)  │
                   └────────────┘  └────────────┘
                          │              │
                    FIFO Order      FIFO Order
                    Within Session  Within Session
```

### Code Pattern
```csharp
// Producer with Session
var message = new ServiceBusMessage("Add item to cart")
{
    SessionId = "user-123"  // All messages with same SessionId are ordered
};
await sender.SendMessageAsync(message);

// Consumer with Session
var sessionProcessor = client.CreateSessionProcessor(queueName, 
    new ServiceBusSessionProcessorOptions
    {
        MaxConcurrentSessions = 5
    });

sessionProcessor.ProcessMessageAsync += async args =>
{
    Console.WriteLine($"Session: {args.SessionId}");
    // Process messages in FIFO order within session
};
```

### Characteristics
- ✅ FIFO guarantee within session
- ✅ Session state management
- ✅ Parallel sessions processing
- ⚠️ Session lock timeout

---

## 6. Priority Queue Pattern

### Description
Process high-priority messages before low-priority ones using message properties.

### Use Cases
- Emergency notifications
- VIP customer orders
- System alerts

### ASCII Diagram
```
┌──────────┐         ┌─────────────────────────┐
│ Producer │ ──────> │  Service Bus Topic      │
└──────────┘         └────────┬────────────────┘
                              │
      Set Priority            │
      Property         ┌──────┴──────┐
                       │             │
                       ▼             ▼
             ┌──────────────┐  ┌──────────────┐
             │Subscription  │  │Subscription  │
             │ HIGH Priority│  │ LOW Priority │
             │              │  │              │
             │Filter:       │  │Filter:       │
             │Priority >= 8 │  │Priority < 8  │
             └──────┬───────┘  └──────┬───────┘
                    │                 │
                    ▼                 ▼
             ┌──────────────┐  ┌──────────────┐
             │  Consumer    │  │  Consumer    │
             │(High Prio    │  │(Low Prio     │
             │ Worker Pool) │  │ Worker Pool) │
             └──────────────┘  └──────────────┘
```

### Code Pattern
```csharp
// Send with priority
var message = new ServiceBusMessage("VIP Order")
{
    ApplicationProperties = { ["Priority"] = 10 }
};
await sender.SendMessageAsync(message);

// Subscription filter (High Priority)
var rule = new CreateRuleOptions
{
    Filter = new SqlRuleFilter("Priority >= 8"),
    Name = "HighPriorityFilter"
};
```

### Characteristics
- ✅ Priority-based routing
- ✅ Separate processing pools
- ✅ Flexible filtering
- ⚠️ Not true priority queue (uses filtering)

---

## 7. Dead Letter Queue Pattern

### Description
Handles messages that cannot be processed after maximum delivery attempts.

### Use Cases
- Failed message analysis
- Poison message handling
- Retry logic
- Monitoring and alerts

### ASCII Diagram
```
┌──────────┐         ┌─────────────────┐         ┌──────────┐
│ Producer │ ──────> │  Service Bus    │ ──────> │ Consumer │
│          │         │     Queue       │         │          │
└──────────┘         └────────┬────────┘         └────┬─────┘
                              │                       │
                              │                       │ Processing
                              │                       │ Fails
                              │                       │
                              │                       ▼
                              │              ┌─────────────────┐
                              │              │ Max Delivery    │
                              │              │ Count Exceeded  │
                              │              └────────┬────────┘
                              │                       │
                              ▼                       ▼
                     ┌──────────────────────────────────┐
                     │    Dead Letter Queue (DLQ)       │
                     │                                  │
                     │  [Failed Msg 1][Failed Msg 2]   │
                     └──────────────┬───────────────────┘
                                    │
                                    │
                                    ▼
                           ┌─────────────────┐
                           │ DLQ Processor   │
                           │ - Analyze       │
                           │ - Log           │
                           │ - Alert         │
                           │ - Manual Fix    │
                           └─────────────────┘
```

### Code Pattern
```csharp
// Configure max delivery count
var queueOptions = new CreateQueueOptions(queueName)
{
    MaxDeliveryCount = 3
};

// Process message (on failure, retry count increments)
try
{
    await ProcessMessage(message);
    await receiver.CompleteMessageAsync(message);
}
catch (Exception ex)
{
    // Abandon increments delivery count
    await receiver.AbandonMessageAsync(message);
}

// Process DLQ messages
var dlqReceiver = client.CreateReceiver(queueName, 
    new ServiceBusReceiverOptions
    {
        SubQueue = SubQueue.DeadLetter
    });

await foreach (var dlqMessage in dlqReceiver.ReceiveMessagesAsync())
{
    Console.WriteLine($"DLQ Reason: {dlqMessage.DeadLetterReason}");
    Console.WriteLine($"Error: {dlqMessage.DeadLetterErrorDescription}");
}
```

### Characteristics
- ✅ Automatic poison message handling
- ✅ Failure analysis
- ✅ Separate processing logic
- ⚠️ Requires monitoring

---

## 8. Saga Pattern with Service Bus

### Description
Distributed transaction pattern using message choreography for long-running business processes.

### Use Cases
- Order fulfillment
- Travel booking
- Payment processing

### ASCII Diagram
```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Order      │ Event1  │  Inventory   │ Event2  │   Payment    │
│   Service    │ ──────> │   Service    │ ──────> │   Service    │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │ OrderCreated           │ ItemsReserved          │ PaymentProcessed
       │                        │                        │
       ▼                        ▼                        ▼
┌────────────────────────────────────────────────────────────────┐
│              Service Bus Topic: Order-Saga-Events              │
└───┬─────────────────────┬──────────────────────┬───────────────┘
    │                     │                      │
    │ Subscription        │ Subscription         │ Subscription
    │                     │                      │
    ▼                     ▼                      ▼
┌─────────┐         ┌─────────┐         ┌─────────┐
│Inventory│         │ Payment │         │Shipping │
│ Service │         │ Service │         │ Service │
└────┬────┘         └────┬────┘         └────┬────┘
     │                   │                   │
     │ If Failed         │ If Failed         │ Success
     │                   │                   │
     ▼                   ▼                   ▼
┌────────────────────────────────────────────────┐
│    Compensation Events (Rollback)              │
│  - CancelReservation                           │
│  - RefundPayment                               │
└────────────────────────────────────────────────┘
```

### Code Pattern
```csharp
// Order Service: Start Saga
await publisher.SendMessageAsync(new ServiceBusMessage(JsonSerializer.Serialize(
    new OrderCreatedEvent { OrderId = orderId }
)));

// Inventory Service: Reserve Items
processor.ProcessMessageAsync += async args =>
{
    var orderEvent = JsonSerializer.Deserialize<OrderCreatedEvent>(args.Message.Body);
    
    if (await ReserveItems(orderEvent.OrderId))
    {
        await publisher.SendMessageAsync(new ServiceBusMessage(
            JsonSerializer.Serialize(new ItemsReservedEvent { OrderId = orderEvent.OrderId })
        ));
    }
    else
    {
        // Compensation: Cancel order
        await publisher.SendMessageAsync(new ServiceBusMessage(
            JsonSerializer.Serialize(new OrderCancelledEvent { OrderId = orderEvent.OrderId })
        ));
    }
};
```

### Characteristics
- ✅ Distributed transactions
- ✅ Eventual consistency
- ✅ Compensation logic
- ⚠️ Complex coordination

---

## 9. Claim Check Pattern

### Description
Store large message payload externally (Blob Storage) and send only a reference through Service Bus.

### Use Cases
- Large file processing
- Image/video uploads
- Document processing

### ASCII Diagram
```
┌──────────┐              ┌─────────────────┐
│ Producer │              │  Blob Storage   │
│          │              │                 │
│          │  1. Upload   │  [Large File]   │
│          │ ───────────> │  file-123.zip   │
└────┬─────┘              └─────────────────┘
     │                             ▲
     │                             │
     │  2. Send claim check        │
     │     (reference only)        │
     │                             │
     ▼                             │
┌─────────────────┐                │
│  Service Bus    │                │
│     Queue       │                │
│                 │                │
│ ClaimCheck:     │                │
│ "file-123.zip"  │                │
└────────┬────────┘                │
         │                         │
         │  3. Receive             │
         │     claim check         │
         ▼                         │
    ┌──────────┐                  │
    │ Consumer │  4. Download     │
    │          │ ──────────────────┘
    └──────────┘
```

### Code Pattern
```csharp
// Producer: Upload to Blob and send claim check
var blobClient = containerClient.GetBlobClient(fileName);
await blobClient.UploadAsync(largeData);

var claimCheck = new ServiceBusMessage(JsonSerializer.Serialize(new
{
    BlobUrl = blobClient.Uri.ToString(),
    FileName = fileName
}));
await sender.SendMessageAsync(claimCheck);

// Consumer: Retrieve blob using claim check
var claimCheckData = JsonSerializer.Deserialize<ClaimCheck>(message.Body);
var blobClient = new BlobClient(new Uri(claimCheckData.BlobUrl));
var downloadedData = await blobClient.DownloadContentAsync();
```

### Characteristics
- ✅ Handles large payloads
- ✅ Reduces message size
- ✅ Cost-effective
- ⚠️ Additional storage costs
- ⚠️ Two-phase retrieval

---

## Pattern Selection Matrix

| Pattern | Use Case | Complexity | Ordering | Scalability |
|---------|----------|------------|----------|-------------|
| Point-to-Point | Simple messaging | Low | ✅ | Medium |
| Competing Consumers | High throughput | Low | ❌ | High |
| Pub-Sub | Event broadcasting | Medium | ✅ | High |
| Request-Reply | Sync-like operations | Medium | ✅ | Medium |
| Message Sessions | Ordered workflows | High | ✅ | Medium |
| Priority Queue | Priority handling | Medium | ⚠️ | Medium |
| Dead Letter | Error handling | Low | N/A | N/A |
| Saga | Distributed transactions | High | ✅ | Medium |
| Claim Check | Large payloads | Medium | ✅ | High |

---

## Best Practices Summary

1. **Use Peek-Lock mode** for reliable message processing
2. **Implement idempotency** for all message handlers
3. **Set appropriate TTL** on messages to prevent queue buildup
4. **Monitor Dead Letter Queues** regularly
5. **Use Sessions** when FIFO ordering is critical
6. **Batch send** for high-throughput scenarios
7. **Enable duplicate detection** for critical workflows
8. **Implement retry policies** with exponential backoff
9. **Use Claim Check** for messages > 256 KB
10. **Set MaxConcurrentCalls** based on workload and resources

