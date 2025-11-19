# Transactional Nature of Azure Messaging Services

## Overview
This document compares the transactional capabilities and guarantees of Azure Event Grid, Service Bus, and Event Hubs for both message sending (publishing) and receiving (consuming) operations.

---

## 1. Azure Service Bus - Strong Transactional Support

### 1.1 Sending (Publishing) Transactions

#### ✅ Full Transaction Support
Service Bus provides **comprehensive transactional sending** capabilities:

```csharp
// Example: Transactional Send
using var serviceBusClient = new ServiceBusClient(connectionString);
var sender = serviceBusClient.CreateSender(queueName);

using (var transactionScope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    // Send multiple messages atomically
    await sender.SendMessageAsync(new ServiceBusMessage("Message 1"));
    await sender.SendMessageAsync(new ServiceBusMessage("Message 2"));
    await sender.SendMessageAsync(new ServiceBusMessage("Message 3"));
    
    // Update database in same transaction
    await database.UpdateAsync("UPDATE Orders SET Status='Sent'");
    
    // All operations succeed or all fail together
    transactionScope.Complete();
}
```

#### Transaction Capabilities:
- **Atomic Multi-Message Send:** Send multiple messages to the same queue/topic as a single atomic operation
- **Cross-Entity Transactions:** Send to multiple queues/topics within the same namespace
- **Database Integration:** Coordinate with local database transactions using TransactionScope
- **AMQP Transactions:** Native protocol-level transaction support
- **Batch Operations:** Transactional batch sends (up to 100 messages per batch)

#### Guarantees:
- ✅ **All-or-Nothing:** Either all messages are enqueued or none are
- ✅ **Durability:** Once committed, messages are persisted to disk
- ✅ **Acknowledgment:** Sender receives confirmation after successful commit
- ✅ **No Duplicates:** Transaction rollback prevents duplicate sends
- ✅ **Ordering:** Transaction maintains message order within a session

### 1.2 Receiving (Consuming) Transactions

#### ✅ Peek-Lock Pattern (Default)
Service Bus provides **transactional receive semantics** through Peek-Lock:

```csharp
// Example: Transactional Receive with Peek-Lock
var receiver = serviceBusClient.CreateReceiver(queueName);

ServiceBusReceivedMessage message = await receiver.ReceiveMessageAsync();

try
{
    using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
    {
        // Process message
        await ProcessMessageAsync(message);
        
        // Update database
        await database.UpdateAsync($"UPDATE Orders SET Status='Processed'");
        
        // Complete message (remove from queue)
        await receiver.CompleteMessageAsync(message);
        
        scope.Complete(); // Commit transaction
    }
}
catch (Exception ex)
{
    // Abandon message (returns to queue)
    await receiver.AbandonMessageAsync(message);
    // Or explicitly dead-letter
    await receiver.DeadLetterMessageAsync(message, "Processing failed", ex.Message);
}
```

#### Message Lock Behavior:
1. **Receive:** Message is locked (invisible to other consumers) for lock duration (default: 60 seconds)
2. **Process:** Consumer processes the message
3. **Complete:** Message is deleted from queue (committed)
4. **Abandon:** Message is unlocked and returned to queue (rolled back)
5. **Dead-Letter:** Message is moved to dead-letter queue

#### Transaction Capabilities:
- **Atomic Receive-and-Process:** Receive, process, and complete as atomic operation
- **Multi-Message Transactions:** Receive and complete multiple messages together
- **Cross-Entity Transactions:** Receive from one queue and send to another atomically
- **Session Transactions:** Process entire session atomically

```csharp
// Example: Receive from one queue and send to another atomically
using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    var message = await receiver.ReceiveMessageAsync();
    await ProcessMessageAsync(message);
    
    // Send result to another queue in same transaction
    await resultSender.SendMessageAsync(new ServiceBusMessage("Result"));
    
    // Complete original message
    await receiver.CompleteMessageAsync(message);
    
    scope.Complete(); // All-or-nothing
}
```

#### Guarantees:
- ✅ **At-Least-Once Delivery:** Message redelivered if not completed before lock expires
- ✅ **Exactly-Once Processing:** Can achieve through idempotency and duplicate detection
- ✅ **Lock Renewal:** Extend lock duration for long-running operations
- ✅ **Automatic Retry:** Message returns to queue if abandoned or lock expires
- ✅ **Dead-Letter Queue:** Failed messages automatically moved after max delivery count

### 1.3 Receive-and-Delete Mode

#### ⚠️ Non-Transactional Alternative
```csharp
// Non-transactional receive (not recommended for critical scenarios)
var receiver = serviceBusClient.CreateReceiver(queueName, 
    new ServiceBusReceiverOptions { ReceiveMode = ServiceBusReceiveMode.ReceiveAndDelete });

var message = await receiver.ReceiveMessageAsync(); // Message immediately deleted
// If processing fails here, message is lost!
```

#### Characteristics:
- ❌ **No Rollback:** Message deleted immediately upon receipt
- ❌ **No Lock:** No protection against processing failure
- ⚠️ **Use Case:** Only for non-critical, loss-tolerant scenarios
- ✅ **Performance:** Lower latency, higher throughput

---

## 2. Azure Event Hubs - Limited Transactional Support

### 2.1 Sending (Publishing) Transactions

#### ⚠️ Batch-Level Atomicity Only
Event Hubs provides **limited transactional sending** through batching:

```csharp
// Example: Batch Send (atomic at batch level)
var producerClient = new EventHubProducerClient(connectionString, eventHubName);

using EventDataBatch eventBatch = await producerClient.CreateBatchAsync();

// Add events to batch
eventBatch.TryAdd(new EventData("Event 1"));
eventBatch.TryAdd(new EventData("Event 2"));
eventBatch.TryAdd(new EventData("Event 3"));

// Send batch - all events sent or none
await producerClient.SendAsync(eventBatch);
```

#### Transaction Capabilities:
- ✅ **Batch Atomicity:** All events in a batch succeed or fail together
- ✅ **Partition Assignment:** Batch sent to single partition atomically
- ❌ **No Cross-Partition Transactions:** Cannot span multiple partitions
- ❌ **No Multi-Batch Transactions:** Each batch is independent
- ❌ **No External Resource Coordination:** Cannot coordinate with databases

#### Guarantees:
- ✅ **Batch All-or-Nothing:** Entire batch accepted or rejected
- ✅ **Durability:** Events persisted to at least 2 replicas before ack
- ✅ **Ordering:** Events in same batch maintain order within partition
- ⚠️ **Partition-Scoped:** Ordering only within same partition key
- ❌ **No Global Transactions:** Cannot coordinate multiple operations

### 2.2 Receiving (Consuming) Transactions

#### ❌ No Built-in Transaction Support
Event Hubs operates on a **streaming, offset-based model** without transaction semantics:

```csharp
// Example: Event Hubs Consumer (manual offset management)
var consumerClient = new EventHubConsumerClient(consumerGroup, connectionString, eventHubName);

await foreach (PartitionEvent partitionEvent in consumerClient.ReadEventsAsync())
{
    // Process event
    await ProcessEventAsync(partitionEvent.Data);
    
    // Manual checkpoint (not transactional)
    await partitionEvent.Partition.UpdateCheckpointAsync(partitionEvent.Data);
    // If processing failed above, checkpoint still updated!
}
```

#### Characteristics:
- ❌ **No Message Locks:** Events are not locked for processing
- ❌ **No Automatic Retry:** Consumer must implement retry logic
- ❌ **No Dead-Letter Queue:** Failed events must be handled manually
- ⚠️ **Manual Checkpointing:** Consumer controls offset advancement
- ✅ **Replay Capability:** Can reset offset to reprocess events

#### Checkpoint-Based Processing:
```csharp
// Example: Checkpoint with error handling
var processor = new EventProcessorClient(blobContainerClient, consumerGroup, 
    connectionString, eventHubName);

processor.ProcessEventAsync += async eventArgs =>
{
    try
    {
        // Process event
        await ProcessEventAsync(eventArgs.Data);
        
        // Update checkpoint AFTER successful processing
        await eventArgs.UpdateCheckpointAsync();
    }
    catch (Exception ex)
    {
        // DO NOT checkpoint - will reprocess on next run
        logger.LogError(ex, "Failed to process event");
        // Must implement own retry/dead-letter logic
    }
};
```

#### Guarantees:
- ✅ **At-Least-Once Delivery:** Events reprocessed if not checkpointed
- ❌ **No Exactly-Once:** Must implement idempotency in consumer
- ❌ **No Atomicity:** Processing and checkpointing are separate operations
- ⚠️ **Possible Duplicates:** Events may be reprocessed after failure
- ✅ **Historical Replay:** Can rewind to any previous offset

### 2.3 Comparison: Event Hubs vs Service Bus

| Feature | Service Bus | Event Hubs |
|---------|-------------|------------|
| **Message Lock** | ✅ Yes (Peek-Lock) | ❌ No |
| **Atomic Complete** | ✅ Yes | ❌ No |
| **Rollback Support** | ✅ Abandon message | ❌ Manual retry |
| **Dead-Letter Queue** | ✅ Built-in | ❌ Manual handling |
| **Transaction Scope** | ✅ Full support | ❌ Not supported |
| **Delivery Count** | ✅ Tracked automatically | ❌ Manual tracking |
| **Idempotency Required** | ⚠️ Recommended | ✅ Required |

---

## 3. Azure Event Grid - No Transaction Support

### 3.1 Sending (Publishing) Transactions

#### ❌ No Transaction Support
Event Grid provides **fire-and-forget publishing** without transactional guarantees:

```csharp
// Example: Event Grid Publishing (non-transactional)
var client = new EventGridPublisherClient(
    new Uri(topicEndpoint), 
    new AzureKeyCredential(topicKey));

// Each event published independently
await client.SendEventAsync(new EventGridEvent(
    subject: "/orders/123",
    eventType: "OrderCreated",
    dataVersion: "1.0",
    data: new { OrderId = 123 }));

// Cannot batch atomically or coordinate with other operations
```

#### Characteristics:
- ❌ **No Batching Transactions:** Each event published independently
- ❌ **No Multi-Event Atomicity:** Cannot send multiple events atomically
- ❌ **No Rollback:** Once sent, cannot be recalled
- ❌ **No External Coordination:** Cannot coordinate with database transactions
- ✅ **Fast Publishing:** Low latency, fire-and-forget

#### Publishing Guarantees:
- ✅ **HTTP Acknowledgment:** 200 OK confirms Event Grid received the event
- ✅ **Durable Storage:** Event stored before acknowledgment
- ⚠️ **Best Effort Ordering:** No guaranteed ordering
- ❌ **No Transaction Coordination:** Each publish is independent

### 3.2 Receiving (Consuming) Transactions

#### ❌ No Transaction Support (Push Model)
Event Grid uses a **webhook push model** without transaction semantics:

```csharp
// Example: Event Grid Webhook Handler (non-transactional)
[HttpPost]
public async Task<IActionResult> HandleEvent([FromBody] EventGridEvent[] events)
{
    foreach (var evt in events)
    {
        try
        {
            // Process event
            await ProcessEventAsync(evt);
            
            // Return 200 - Event Grid considers it delivered
            // If processing fails AFTER 200, event is lost!
        }
        catch (Exception ex)
        {
            // Return 5xx - Event Grid will retry
            return StatusCode(500);
        }
    }
    
    return Ok(); // All events marked as delivered
}
```

#### Characteristics:
- ❌ **No Message Lock:** Event delivered immediately
- ❌ **No Rollback:** Cannot return event to Event Grid
- ⚠️ **HTTP-Based Acknowledgment:** 200 = success, 5xx = retry, 4xx = drop
- ❌ **No Dead-Letter on Subscriber Side:** Must implement manually
- ✅ **Event Grid Dead-Letter:** Undelivered events stored in blob storage

#### Delivery Behavior:
1. **Event Grid pushes** event to webhook endpoint
2. **Webhook responds** with HTTP status:
   - **200-299:** Success - event marked as delivered
   - **500-599:** Failure - Event Grid retries (exponential backoff, up to 24 hours)
   - **400-413:** Permanent failure - no retry, dead-lettered immediately
3. **No Rollback:** Once 200 returned, event considered delivered

#### Delivery Guarantees:
- ✅ **At-Least-Once Delivery:** Event redelivered on failure (5xx)
- ❌ **No Exactly-Once:** Retries may cause duplicates
- ❌ **No Atomicity:** Cannot coordinate with other operations
- ⚠️ **Idempotency Required:** Must handle duplicate deliveries
- ✅ **Dead-Letter Support:** Configure blob storage for failed deliveries

---

## 4. Transaction Support Comparison Matrix

### 4.1 Sending (Publishing) Transactions

| Feature | Service Bus | Event Hubs | Event Grid |
|---------|-------------|------------|------------|
| **Atomic Multi-Message Send** | ✅ Yes | ⚠️ Batch only | ❌ No |
| **Cross-Entity Transactions** | ✅ Yes (same namespace) | ❌ No | ❌ No |
| **Database Coordination** | ✅ TransactionScope | ❌ No | ❌ No |
| **Rollback Support** | ✅ Yes | ⚠️ Batch level | ❌ No |
| **Duplicate Prevention** | ✅ Built-in | ❌ Manual | ❌ Manual |
| **Ordered Delivery** | ✅ Sessions | ⚠️ Partition-scoped | ❌ No |
| **Protocol-Level Transactions** | ✅ AMQP | ❌ No | ❌ No |

### 4.2 Receiving (Consuming) Transactions

| Feature | Service Bus | Event Hubs | Event Grid |
|---------|-------------|------------|------------|
| **Message Lock** | ✅ Peek-Lock | ❌ No | ❌ No |
| **Atomic Processing** | ✅ Complete/Abandon | ❌ Manual checkpoint | ❌ HTTP response |
| **Rollback Capability** | ✅ Abandon message | ⚠️ Don't checkpoint | ❌ Return 5xx |
| **Dead-Letter Queue** | ✅ Built-in | ❌ Manual | ⚠️ Event Grid-level |
| **Delivery Count Tracking** | ✅ Automatic | ❌ Manual | ✅ Automatic |
| **Transaction Scope Support** | ✅ Full | ❌ No | ❌ No |
| **Exactly-Once Capable** | ✅ With dedup | ❌ Idempotency required | ❌ Idempotency required |
| **Retry Logic** | ✅ Automatic | ❌ Manual | ✅ Automatic |

---

## 5. Transactional Patterns and Best Practices

### 5.1 Service Bus Transactional Patterns

#### Pattern 1: Request-Response with Transaction
```csharp
// Atomic receive from request queue and send to response queue
using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    var request = await requestReceiver.ReceiveMessageAsync();
    var response = await ProcessRequest(request.Body.ToString());
    
    await responseSender.SendMessageAsync(new ServiceBusMessage(response));
    await requestReceiver.CompleteMessageAsync(request);
    
    scope.Complete(); // Both operations succeed or both fail
}
```

#### Pattern 2: Aggregate Processing
```csharp
// Process multiple related messages atomically
using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    var messages = await receiver.ReceiveMessagesAsync(maxMessages: 10);
    
    foreach (var message in messages)
    {
        await ProcessMessageAsync(message);
        await receiver.CompleteMessageAsync(message);
    }
    
    await database.CommitBatchAsync();
    scope.Complete();
}
```

#### Pattern 3: Saga Pattern with Compensation
```csharp
// Multi-step transaction with rollback
var message = await receiver.ReceiveMessageAsync();
var compensationActions = new List<Func<Task>>();

try
{
    // Step 1
    await service1.Process();
    compensationActions.Add(async () => await service1.Rollback());
    
    // Step 2
    await service2.Process();
    compensationActions.Add(async () => await service2.Rollback());
    
    // Success - complete message
    await receiver.CompleteMessageAsync(message);
}
catch
{
    // Compensate in reverse order
    compensationActions.Reverse();
    foreach (var compensate in compensationActions)
        await compensate();
    
    await receiver.AbandonMessageAsync(message);
}
```

### 5.2 Event Hubs Patterns (Compensating for Lack of Transactions)

#### Pattern 1: Checkpoint After Success
```csharp
// Only checkpoint after successful processing
async Task ProcessEventsAsync(ProcessEventArgs args)
{
    bool success = false;
    try
    {
        await ProcessEventAsync(args.Data);
        await database.SaveAsync();
        success = true;
    }
    finally
    {
        if (success)
            await args.UpdateCheckpointAsync(); // Only checkpoint on success
    }
}
```

#### Pattern 2: Idempotent Processing with Deduplication
```csharp
// Store processed event IDs to prevent duplicates
async Task ProcessEventAsync(EventData eventData)
{
    var eventId = eventData.MessageId;
    
    // Check if already processed (idempotency)
    if (await database.IsProcessedAsync(eventId))
        return; // Skip duplicate
    
    using (var transaction = await database.BeginTransactionAsync())
    {
        await ProcessDataAsync(eventData.Body);
        await database.MarkAsProcessedAsync(eventId); // Track processed IDs
        await transaction.CommitAsync();
    }
}
```

#### Pattern 3: Manual Dead-Letter to Blob Storage
```csharp
// Implement manual dead-letter for failed events
async Task ProcessWithDeadLetterAsync(EventData eventData)
{
    int retryCount = 0;
    int maxRetries = 3;
    
    while (retryCount < maxRetries)
    {
        try
        {
            await ProcessEventAsync(eventData);
            return; // Success
        }
        catch (Exception ex)
        {
            retryCount++;
            if (retryCount >= maxRetries)
            {
                // Move to dead-letter (manual)
                await blobClient.UploadAsync(
                    $"dead-letter/{eventData.MessageId}.json",
                    eventData.Body);
                logger.LogError(ex, "Event dead-lettered after {Retries} retries", maxRetries);
            }
            else
            {
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, retryCount)));
            }
        }
    }
}
```

### 5.3 Event Grid Patterns (Compensating for Lack of Transactions)

#### Pattern 1: Idempotent Webhook Handler
```csharp
[HttpPost]
public async Task<IActionResult> HandleEvent([FromBody] EventGridEvent[] events)
{
    foreach (var evt in events)
    {
        // Use event ID for idempotency
        if (await database.IsEventProcessedAsync(evt.Id))
            continue; // Skip duplicate
        
        try
        {
            using (var transaction = await database.BeginTransactionAsync())
            {
                await ProcessEventAsync(evt);
                await database.MarkEventProcessedAsync(evt.Id);
                await transaction.CommitAsync();
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed to process event {EventId}", evt.Id);
            return StatusCode(500); // Trigger Event Grid retry
        }
    }
    
    return Ok();
}
```

#### Pattern 2: Two-Phase Processing
```csharp
// Store event first, process later (durability)
[HttpPost]
public async Task<IActionResult> HandleEvent([FromBody] EventGridEvent[] events)
{
    try
    {
        // Phase 1: Store events durably (fast response)
        await eventStore.StoreEventsAsync(events);
        
        // Phase 2: Process asynchronously (background job)
        _ = Task.Run(() => ProcessStoredEventsAsync(events));
        
        return Ok(); // Quick response to Event Grid
    }
    catch
    {
        return StatusCode(500); // Trigger retry
    }
}

async Task ProcessStoredEventsAsync(EventGridEvent[] events)
{
    foreach (var evt in events)
    {
        try
        {
            await ProcessEventAsync(evt);
            await eventStore.MarkCompletedAsync(evt.Id);
        }
        catch
        {
            await eventStore.MarkFailedAsync(evt.Id);
            // Implement own retry logic
        }
    }
}
```

#### Pattern 3: Event Grid + Service Bus Hybrid
```csharp
// Use Event Grid for fast distribution, Service Bus for reliability
[HttpPost]
public async Task<IActionResult> HandleEvent([FromBody] EventGridEvent[] events)
{
    // Forward to Service Bus for transactional processing
    var sender = serviceBusClient.CreateSender(queueName);
    
    try
    {
        foreach (var evt in events)
        {
            var message = new ServiceBusMessage(JsonSerializer.Serialize(evt));
            await sender.SendMessageAsync(message);
        }
        
        return Ok(); // Event Grid delivery complete
    }
    catch
    {
        return StatusCode(500);
    }
}

// Separate Service Bus consumer with full transaction support
async Task ProcessFromServiceBusAsync()
{
    var receiver = serviceBusClient.CreateReceiver(queueName);
    
    while (true)
    {
        var message = await receiver.ReceiveMessageAsync();
        
        using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
        {
            var evt = JsonSerializer.Deserialize<EventGridEvent>(message.Body);
            await ProcessEventAsync(evt);
            await receiver.CompleteMessageAsync(message);
            scope.Complete();
        }
    }
}
```

---

## 6. Decision Matrix: When to Use Each Service

### 6.1 Use Service Bus When:
✅ **Transactions are Critical**
- Financial transactions
- Order processing
- Payment workflows
- Inventory management

✅ **Exactly-Once Processing Required**
- No duplicate tolerance
- Critical business operations

✅ **Strong Ordering Needed**
- Sequential processing within sessions
- FIFO guarantees

✅ **Complex Message Patterns**
- Request-response
- Saga patterns
- Multi-step workflows

### 6.2 Use Event Hubs When:
✅ **High Throughput Streaming**
- Telemetry ingestion (millions of events/sec)
- Log aggregation
- Real-time analytics

✅ **Historical Replay Needed**
- Reprocess events from any point
- Multiple consumers at different speeds

⚠️ **Eventual Consistency Acceptable**
- Can tolerate duplicates (with idempotency)
- Order within partition is sufficient

❌ **Avoid for:**
- Transactional business operations
- Workflows requiring rollback

### 6.3 Use Event Grid When:
✅ **Event-Driven Architectures**
- Reactive programming
- Microservices communication
- System integration events

✅ **Low Latency Distribution**
- Fast event fanout to multiple subscribers
- Serverless triggers

✅ **Azure Service Integration**
- Built-in events from Azure resources
- Ops automation

⚠️ **Idempotency Must Be Implemented**
- Cannot rely on exactly-once delivery
- Must handle retries

❌ **Avoid for:**
- Transactional workflows
- Ordered processing requirements
- Critical financial operations

---

## 7. Summary Table

| Aspect | Service Bus | Event Hubs | Event Grid |
|--------|-------------|------------|------------|
| **Send Transaction** | ✅ Full (AMQP) | ⚠️ Batch only | ❌ None |
| **Receive Transaction** | ✅ Peek-Lock | ❌ Checkpoint-based | ❌ HTTP response |
| **Rollback Support** | ✅ Abandon/DLQ | ❌ Manual | ⚠️ HTTP 5xx retry |
| **Exactly-Once** | ✅ With dedup | ❌ Requires idempotency | ❌ Requires idempotency |
| **Message Lock** | ✅ 60s (renewable) | ❌ No | ❌ No |
| **Dead-Letter Queue** | ✅ Built-in | ❌ Manual | ⚠️ Blob storage |
| **TransactionScope** | ✅ Supported | ❌ Not supported | ❌ Not supported |
| **Best For** | Transactional workflows | Streaming analytics | Event distribution |
| **Complexity** | High (but powerful) | Medium | Low |
| **Reliability** | Highest | Medium | Medium |

---

## 8. Code Examples: Complete Scenarios

### 8.1 Service Bus: E-Commerce Order Processing (Transactional)

```csharp
public class OrderProcessor
{
    private readonly ServiceBusClient _serviceBusClient;
    private readonly IDbContext _dbContext;

    public async Task ProcessOrderTransactionallyAsync()
    {
        var receiver = _serviceBusClient.CreateReceiver("orders");
        var notificationSender = _serviceBusClient.CreateSender("notifications");
        var inventorySender = _serviceBusClient.CreateSender("inventory");

        ServiceBusReceivedMessage orderMessage = await receiver.ReceiveMessageAsync();

        try
        {
            // All operations succeed or all fail
            using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
            {
                // 1. Deserialize order
                var order = JsonSerializer.Deserialize<Order>(orderMessage.Body);

                // 2. Update database
                await _dbContext.Orders.AddAsync(order);
                await _dbContext.SaveChangesAsync();

                // 3. Send notification (atomically)
                var notification = new ServiceBusMessage($"Order {order.Id} received");
                await notificationSender.SendMessageAsync(notification);

                // 4. Update inventory (atomically)
                var inventoryUpdate = new ServiceBusMessage($"Reserve {order.ProductId}");
                await inventorySender.SendMessageAsync(inventoryUpdate);

                // 5. Complete order message
                await receiver.CompleteMessageAsync(orderMessage);

                // Commit all operations
                scope.Complete();
            }

            Console.WriteLine($"Order {orderMessage.MessageId} processed successfully");
        }
        catch (Exception ex)
        {
            // Transaction rolled back automatically
            // Message returned to queue for retry
            await receiver.AbandonMessageAsync(orderMessage);
            Console.WriteLine($"Order processing failed: {ex.Message}");
        }
    }
}
```

### 8.2 Event Hubs: Telemetry Processing (Idempotent Pattern)

```csharp
public class TelemetryProcessor
{
    private readonly EventProcessorClient _processor;
    private readonly IDbContext _dbContext;
    private readonly BlobContainerClient _deadLetterContainer;

    public async Task StartProcessingAsync()
    {
        _processor.ProcessEventAsync += ProcessEventHandler;
        _processor.ProcessErrorAsync += ProcessErrorHandler;

        await _processor.StartProcessingAsync();
    }

    private async Task ProcessEventHandler(ProcessEventArgs args)
    {
        var eventData = args.Data;
        var eventId = eventData.MessageId;
        
        // Idempotency check
        if (await _dbContext.ProcessedEvents.AnyAsync(e => e.EventId == eventId))
        {
            await args.UpdateCheckpointAsync(); // Already processed, just checkpoint
            return;
        }

        int retryCount = 0;
        const int maxRetries = 3;

        while (retryCount < maxRetries)
        {
            try
            {
                // Process within database transaction for consistency
                using (var transaction = await _dbContext.Database.BeginTransactionAsync())
                {
                    // 1. Process telemetry
                    var telemetry = JsonSerializer.Deserialize<Telemetry>(eventData.Body);
                    await _dbContext.Telemetry.AddAsync(telemetry);

                    // 2. Mark as processed (prevent duplicates)
                    await _dbContext.ProcessedEvents.AddAsync(new ProcessedEvent 
                    { 
                        EventId = eventId,
                        ProcessedAt = DateTime.UtcNow 
                    });

                    await _dbContext.SaveChangesAsync();
                    await transaction.CommitAsync();
                }

                // 3. Checkpoint ONLY after successful processing
                await args.UpdateCheckpointAsync();
                return; // Success
            }
            catch (Exception ex)
            {
                retryCount++;
                
                if (retryCount >= maxRetries)
                {
                    // Manual dead-letter to blob storage
                    await _deadLetterContainer.UploadBlobAsync(
                        $"dead-letter/{eventId}_{DateTime.UtcNow:yyyyMMddHHmmss}.json",
                        new BinaryData(eventData.Body));
                    
                    // Checkpoint to move forward (event is saved for manual review)
                    await args.UpdateCheckpointAsync();
                    Console.WriteLine($"Event {eventId} dead-lettered after {maxRetries} retries");
                }
                else
                {
                    // Exponential backoff before retry
                    await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, retryCount)));
                }
            }
        }
    }

    private Task ProcessErrorHandler(ProcessErrorEventArgs args)
    {
        Console.WriteLine($"Partition {args.PartitionId} error: {args.Exception.Message}");
        return Task.CompletedTask;
    }
}
```

### 8.3 Event Grid: Webhook Handler (Idempotent + Durable)

```csharp
[ApiController]
[Route("api/eventgrid")]
public class EventGridWebhookController : ControllerBase
{
    private readonly IDbContext _dbContext;
    private readonly ILogger<EventGridWebhookController> _logger;

    [HttpPost]
    public async Task<IActionResult> HandleEventGridEvents(
        [FromBody] EventGridEvent[] events)
    {
        foreach (var evt in events)
        {
            // Handle subscription validation
            if (evt.EventType == "Microsoft.EventGrid.SubscriptionValidationEvent")
            {
                var validationData = JsonSerializer.Deserialize<SubscriptionValidationEventData>(
                    evt.Data.ToString());
                return Ok(new { validationResponse = validationData.ValidationCode });
            }

            // Idempotency: Check if event already processed
            var existingEvent = await _dbContext.ProcessedEventGridEvents
                .FirstOrDefaultAsync(e => e.EventId == evt.Id);

            if (existingEvent != null)
            {
                _logger.LogInformation($"Event {evt.Id} already processed, skipping");
                continue; // Skip duplicate
            }

            try
            {
                // Store event first for durability
                using (var transaction = await _dbContext.Database.BeginTransactionAsync())
                {
                    // 1. Store event (durable)
                    var eventRecord = new ProcessedEventGridEvent
                    {
                        EventId = evt.Id,
                        EventType = evt.EventType,
                        Subject = evt.Subject,
                        EventData = evt.Data.ToString(),
                        ReceivedAt = DateTime.UtcNow,
                        ProcessedAt = null,
                        Status = "Received"
                    };
                    
                    await _dbContext.ProcessedEventGridEvents.AddAsync(eventRecord);
                    await _dbContext.SaveChangesAsync();
                    await transaction.CommitAsync();
                }

                // 2. Process event asynchronously (don't block Event Grid)
                _ = Task.Run(async () =>
                {
                    try
                    {
                        await ProcessEventAsync(evt);
                        
                        // Update status
                        var record = await _dbContext.ProcessedEventGridEvents
                            .FirstAsync(e => e.EventId == evt.Id);
                        record.ProcessedAt = DateTime.UtcNow;
                        record.Status = "Completed";
                        await _dbContext.SaveChangesAsync();
                    }
                    catch (Exception ex)
                    {
                        _logger.LogError(ex, $"Failed to process event {evt.Id}");
                        
                        var record = await _dbContext.ProcessedEventGridEvents
                            .FirstAsync(e => e.EventId == evt.Id);
                        record.Status = "Failed";
                        record.ErrorMessage = ex.Message;
                        await _dbContext.SaveChangesAsync();
                    }
                });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Failed to store event {evt.Id}");
                // Return 5xx to trigger Event Grid retry
                return StatusCode(500, $"Failed to process event: {ex.Message}");
            }
        }

        // Return 200 quickly to Event Grid
        return Ok();
    }

    private async Task ProcessEventAsync(EventGridEvent evt)
    {
        // Actual business logic here
        switch (evt.EventType)
        {
            case "Microsoft.Storage.BlobCreated":
                await HandleBlobCreatedAsync(evt);
                break;
            case "Microsoft.Resources.ResourceWriteSuccess":
                await HandleResourceUpdateAsync(evt);
                break;
            default:
                _logger.LogWarning($"Unknown event type: {evt.EventType}");
                break;
        }
    }

    private async Task HandleBlobCreatedAsync(EventGridEvent evt)
    {
        // Process blob created event
        await Task.Delay(100); // Simulate processing
    }

    private async Task HandleResourceUpdateAsync(EventGridEvent evt)
    {
        // Process resource update event
        await Task.Delay(100); // Simulate processing
    }
}

// Entity models
public class ProcessedEventGridEvent
{
    public int Id { get; set; }
    public string EventId { get; set; } // Event Grid event ID
    public string EventType { get; set; }
    public string Subject { get; set; }
    public string EventData { get; set; }
    public DateTime ReceivedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public string Status { get; set; } // Received, Completed, Failed
    public string ErrorMessage { get; set; }
}
```

---

## 9. Key Takeaways

### Service Bus: The Transactional Champion
- **Use for:** Critical business workflows requiring transactions
- **Strength:** Full ACID transaction support
- **Complexity:** Higher, but provides strongest guarantees
- **Pattern:** Peek-Lock with TransactionScope

### Event Hubs: The Streaming Specialist
- **Use for:** High-throughput streaming and analytics
- **Strength:** Massive scale and replay capability
- **Limitation:** No built-in transactions
- **Pattern:** Idempotent processing with manual checkpointing

### Event Grid: The Event Distributor
- **Use for:** Event-driven architectures and integrations
- **Strength:** Simple, serverless, low latency
- **Limitation:** No transactions, fire-and-forget
- **Pattern:** Idempotent webhook handlers with durability

### Golden Rule
**For transactional workflows → Service Bus**  
**For streaming analytics → Event Hubs**  
**For event distribution → Event Grid**
