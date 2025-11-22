# Azure Event Hubs - Messaging Patterns with ASCII Diagrams

## Table of Contents
1. [Event Streaming Pattern](#1-event-streaming-pattern)
2. [Event Sourcing Pattern](#2-event-sourcing-pattern)
3. [Consumer Group Pattern](#3-consumer-group-pattern)
4. [Partition-Based Processing Pattern](#4-partition-based-processing-pattern)
5. [Checkpointing Pattern](#5-checkpointing-pattern)
6. [Event Capture Pattern](#6-event-capture-pattern)
7. [Stream Processing Pipeline](#7-stream-processing-pipeline)
8. [Time Window Aggregation](#8-time-window-aggregation)
9. [Event Replay Pattern](#9-event-replay-pattern)

---

## 1. Event Streaming Pattern

### Description
Continuous ingestion and processing of high-volume event streams in real-time.

### Use Cases
- IoT telemetry
- Application logging
- Clickstream analytics
- Real-time monitoring

### ASCII Diagram
```
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Device  │  │ Device  │  │ Device  │  │ Device  │
│   #1    │  │   #2    │  │   #3    │  │   #N    │
└────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
     │            │            │            │
     │ Events     │ Events     │ Events     │ Events
     │            │            │            │
     └────────────┴────────────┴────────────┘
                      │
                      ▼
         ┌─────────────────────────┐
         │   Azure Event Hubs      │
         │                         │
         │  ┌──────────────────┐   │
         │  │ Partition 0      │   │ ◄── [E1][E2][E3][E4]
         │  ├──────────────────┤   │
         │  │ Partition 1      │   │ ◄── [E5][E6][E7][E8]
         │  ├──────────────────┤   │
         │  │ Partition 2      │   │ ◄── [E9][E10][E11][E12]
         │  ├──────────────────┤   │
         │  │ Partition 3      │   │ ◄── [E13][E14][E15][E16]
         │  └──────────────────┘   │
         └─────────────┬───────────┘
                       │
                       │ Pull Events
                       │
                       ▼
              ┌─────────────────┐
              │  Stream         │
              │  Processor      │
              │  (Real-time)    │
              └─────────────────┘
```

### Code Pattern
```csharp
// Producer: Send events
var producer = new EventHubProducerClient(connectionString, eventHubName);

var batch = await producer.CreateBatchAsync();
batch.TryAdd(new EventData(Encoding.UTF8.GetBytes("Temperature: 25.3°C")));
await producer.SendAsync(batch);

// Consumer: Process events
var consumer = new EventHubConsumerClient(
    EventHubConsumerClient.DefaultConsumerGroupName,
    connectionString,
    eventHubName);

await foreach (PartitionEvent partitionEvent in consumer.ReadEventsAsync())
{
    string data = Encoding.UTF8.GetString(partitionEvent.Data.EventBody.ToArray());
    Console.WriteLine($"Event: {data}");
}
```

### Characteristics
- ✅ High throughput (millions of events/sec)
- ✅ Low latency
- ✅ Horizontal scaling via partitions
- ✅ Event retention (1-90 days)

---

## 2. Event Sourcing Pattern

### Description
Store all state changes as a sequence of events, enabling event replay and audit trails.

### Use Cases
- Financial transactions
- Order history
- Audit logging
- State reconstruction

### ASCII Diagram
```
┌──────────────────┐
│  Application     │
│  (Commands)      │
└────────┬─────────┘
         │
         │ State Changes as Events
         │
         ▼
┌─────────────────────────────────────┐
│      Azure Event Hubs               │
│                                     │
│  Event Stream (Append-Only)         │
│  ┌───────────────────────────────┐  │
│  │ OrderCreated    → Offset 0    │  │
│  │ ItemAdded       → Offset 1    │  │
│  │ ItemAdded       → Offset 2    │  │
│  │ OrderConfirmed  → Offset 3    │  │
│  │ PaymentReceived → Offset 4    │  │
│  │ OrderShipped    → Offset 5    │  │
│  └───────────────────────────────┘  │
└──────────┬──────────────────────────┘
           │
           │ Read Events
           │
           ▼
┌─────────────────────────┐
│  Event Processor        │
│                         │
│  Replay Events          │
│  Rebuild State          │
│                         │
│  Current State:         │
│  Order: #123            │
│  Status: Shipped        │
│  Items: 2               │
│  Total: $150.00         │
└─────────────────────────┘
```

### Code Pattern
```csharp
// Store events
public async Task SaveEvent(DomainEvent domainEvent)
{
    var eventData = new EventData(JsonSerializer.SerializeToUtf8Bytes(domainEvent))
    {
        PartitionKey = domainEvent.AggregateId  // Same partition for same aggregate
    };
    
    await producer.SendAsync(new[] { eventData });
}

// Rebuild state from events
public async Task<Order> ReconstructOrder(string orderId)
{
    var order = new Order();
    
    await foreach (var partitionEvent in consumer.ReadEventsFromPartitionAsync(
        partitionId, EventPosition.Earliest))
    {
        var evt = JsonSerializer.Deserialize<DomainEvent>(
            partitionEvent.Data.EventBody);
        
        if (evt.AggregateId == orderId)
        {
            order.Apply(evt);  // Replay event
        }
    }
    
    return order;
}
```

### Characteristics
- ✅ Complete audit trail
- ✅ Time travel (replay to any point)
- ✅ Event replay capability
- ⚠️ Requires event versioning strategy

---

## 3. Consumer Group Pattern

### Description
Multiple independent applications process the same event stream with separate offsets.

### Use Cases
- Multi-purpose analytics
- Independent processing pipelines
- Real-time + batch processing

### ASCII Diagram
```
                    ┌──────────────────────────┐
                    │   Azure Event Hubs       │
                    │                          │
                    │  [E1][E2][E3][E4][E5]    │
                    │        Partition 0       │
                    └────┬──────────┬──────┬───┘
                         │          │      │
         Consumer Group  │          │      │  Consumer Group
              "cg1"      │          │      │       "cg2"
                         │          │      │
                    ┌────┴────┐     │  ┌───┴──────┐
                    │ Offset: │     │  │ Offset:  │
                    │    3    │     │  │    5     │
                    └────┬────┘     │  └───┬──────┘
                         │          │      │
                         ▼          │      ▼
                  ┌────────────┐    │  ┌────────────┐
                  │  Stream    │    │  │  Batch     │
                  │ Analytics  │    │  │ Processor  │
                  │ (Real-time)│    │  │ (Delayed)  │
                  └────────────┘    │  └────────────┘
                                    │
                      Consumer Group│
                           "cg3"    │
                                    │
                                ┌───┴──────┐
                                │ Offset:  │
                                │    2     │
                                └───┬──────┘
                                    │
                                    ▼
                               ┌────────────┐
                               │  Archive   │
                               │  Service   │
                               └────────────┘
```

### Code Pattern
```csharp
// Consumer Group 1: Real-time analytics
var realTimeConsumer = new EventHubConsumerClient(
    "analytics-cg",  // Consumer group name
    connectionString,
    eventHubName);

// Consumer Group 2: Batch processing
var batchConsumer = new EventHubConsumerClient(
    "batch-cg",      // Different consumer group
    connectionString,
    eventHubName);

// Consumer Group 3: Archive
var archiveConsumer = new EventHubConsumerClient(
    "archive-cg",    // Independent offset tracking
    connectionString,
    eventHubName);

// Each reads independently at their own pace
```

### Characteristics
- ✅ Independent consumption
- ✅ Separate offset management
- ✅ Multiple processing speeds
- ✅ No interference between consumers

---

## 4. Partition-Based Processing Pattern

### Description
Distribute processing load across multiple partitions for parallel processing.

### Use Cases
- High-throughput scenarios
- Parallel data processing
- Scalable analytics

### ASCII Diagram
```
┌──────────────────────────────────────────────┐
│           Azure Event Hubs                   │
│                                              │
│  ┌────────────────┐   ┌────────────────┐    │
│  │  Partition 0   │   │  Partition 1   │    │
│  │ [E1][E2][E3]   │   │ [E4][E5][E6]   │    │
│  └───────┬────────┘   └────────┬───────┘    │
│          │                     │             │
│  ┌───────┴────────┐   ┌────────┴───────┐    │
│  │  Partition 2   │   │  Partition 3   │    │
│  │ [E7][E8][E9]   │   │ [E10][E11][E12]│    │
│  └───────┬────────┘   └────────┬───────┘    │
└──────────┼─────────────────────┼────────────┘
           │                     │
           │  Parallel           │
           │  Processing         │
           │                     │
    ┌──────┴──────┐       ┌──────┴──────┐
    │             │       │             │
    ▼             ▼       ▼             ▼
┌─────────┐  ┌─────────┐ ┌─────────┐ ┌─────────┐
│Consumer │  │Consumer │ │Consumer │ │Consumer │
│Instance │  │Instance │ │Instance │ │Instance │
│   #1    │  │   #2    │ │   #3    │ │   #4    │
└─────────┘  └─────────┘ └─────────┘ └─────────┘
     │            │           │           │
     └────────────┴───────────┴───────────┘
                      │
              Aggregated Results
```

### Code Pattern
```csharp
// Send to specific partition using partition key
var eventData = new EventData(Encoding.UTF8.GetBytes("Event data"))
{
    PartitionKey = customerId  // Same key → same partition
};
await producer.SendAsync(new[] { eventData });

// Process specific partitions
var processor = new EventProcessorClient(
    blobContainerClient,
    consumerGroup,
    connectionString,
    eventHubName);

processor.ProcessEventAsync += async args =>
{
    Console.WriteLine($"Partition {args.Partition.PartitionId}: {args.Data.EventBody}");
    await args.UpdateCheckpointAsync();
};

await processor.StartProcessingAsync();
```

### Characteristics
- ✅ Parallel processing
- ✅ Load distribution
- ✅ Partition key for related events
- ⚠️ Partition count affects scalability

---

## 5. Checkpointing Pattern

### Description
Track processing progress by storing offset positions, enabling recovery from failures.

### Use Cases
- Reliable event processing
- Failure recovery
- Exactly-once semantics

### ASCII Diagram
```
┌──────────────────────────────────────────────┐
│        Azure Event Hubs (Partition 0)        │
│                                              │
│  [E1][E2][E3][E4][E5][E6][E7][E8][E9][E10]   │
│   0   1   2   3   4   5   6   7   8   9      │
└───────────────────┬──────────────────────────┘
                    │
                    │ Read Events
                    ▼
            ┌────────────────┐
            │  Processor     │
            │                │
            │  Process E1-E5 │
            └────────┬───────┘
                     │
                     │ 1. Process
                     │ 2. Checkpoint
                     ▼
            ┌────────────────────┐
            │  Blob Storage      │
            │  (Checkpoints)     │
            │                    │
            │  Partition 0:      │
            │  Offset = 5        │
            │  Timestamp: ...    │
            └────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        │ Failure Occurs          │
        │ Processor Restarts      │
        │                         │
        ▼                         │
┌────────────────┐                │
│  Recovery      │                │
│                │ ◄──────────────┘
│  Resume from   │
│  Offset 5      │
│                │
│  Process       │
│  [E6][E7]...   │
└────────────────┘
```

### Code Pattern
```csharp
// Setup checkpoint store
var blobContainerClient = new BlobContainerClient(
    storageConnectionString, 
    checkpointContainer);

var processor = new EventProcessorClient(
    blobContainerClient,      // Checkpoint storage
    consumerGroup,
    connectionString,
    eventHubName);

processor.ProcessEventAsync += async args =>
{
    // Process event
    await ProcessEvent(args.Data);
    
    // Checkpoint after processing
    await args.UpdateCheckpointAsync();  // Stores offset to Blob Storage
};

processor.ProcessErrorAsync += async args =>
{
    Console.WriteLine($"Error on partition {args.PartitionId}: {args.Exception}");
    // Processor will resume from last checkpoint
};

await processor.StartProcessingAsync();
```

### Characteristics
- ✅ Automatic failure recovery
- ✅ Progress tracking
- ✅ Prevents duplicate processing
- ⚠️ Requires external storage (Blob)

---

## 6. Event Capture Pattern

### Description
Automatically capture and archive event stream data to Blob Storage or Data Lake.

### Use Cases
- Long-term data retention
- Compliance requirements
- Batch analytics
- Data lake ingestion

### ASCII Diagram
```
                    ┌──────────────────────────┐
                    │   Azure Event Hubs       │
                    │                          │
   Producers ────>  │  [E1][E2][E3][E4][E5]    │
                    │  [E6][E7][E8][E9][E10]   │
                    └────┬──────────────────────┘
                         │
                         │ Automatic Capture
                         │ (No code required)
                         │
                         ▼
         ┌───────────────────────────────────┐
         │    Azure Blob Storage /           │
         │    Data Lake Storage              │
         │                                   │
         │  Capture Files (Avro Format):    │
         │                                   │
         │  /2024/01/15/                     │
         │    ├─ 00/                         │
         │    │  └─ events_00.avro           │
         │    ├─ 01/                         │
         │    │  └─ events_01.avro           │
         │    └─ 02/                         │
         │       └─ events_02.avro           │
         │                                   │
         └───────────┬───────────────────────┘
                     │
                     │ Process archived data
                     │
                     ▼
            ┌─────────────────┐
            │  Batch          │
            │  Processing     │
            │  - Spark        │
            │  - Databricks   │
            │  - Synapse      │
            └─────────────────┘
```

### Code Pattern
```csharp
// Enable capture via Azure Portal or ARM template
// Or programmatically:

var captureDescription = new CaptureDescription
{
    Enabled = true,
    Encoding = EncodingCaptureDescription.Avro,
    IntervalInSeconds = 300,     // Every 5 minutes
    SizeLimitInBytes = 314572800, // Or 300 MB
    Destination = new Destination
    {
        Name = "EventHubArchive.AzureBlockBlob",
        BlobContainer = "captured-events",
        ArchiveNameFormat = "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"
    }
};

// Read captured files
var blobServiceClient = new BlobServiceClient(storageConnectionString);
var containerClient = blobServiceClient.GetBlobContainerClient("captured-events");

await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
{
    var blobClient = containerClient.GetBlobClient(blobItem.Name);
    var downloadedBlob = await blobClient.DownloadContentAsync();
    
    // Parse Avro format
    using var stream = new MemoryStream(downloadedBlob.Value.Content.ToArray());
    using var reader = DataFileReader<GenericRecord>.OpenReader(stream);
    
    foreach (var record in reader.NextEntries)
    {
        // Process archived event
    }
}
```

### Characteristics
- ✅ Automatic archival
- ✅ No code required (configuration only)
- ✅ Avro format (efficient)
- ✅ Time or size-based triggers
- ⚠️ Additional storage costs

---

## 7. Stream Processing Pipeline

### Description
Multi-stage processing pipeline for real-time data transformation and enrichment.

### Use Cases
- ETL pipelines
- Data enrichment
- Real-time filtering and aggregation

### ASCII Diagram
```
┌──────────┐         ┌──────────────┐
│  Source  │ ──────> │  Event Hub   │
│  Events  │         │   (Raw)      │
└──────────┘         └──────┬───────┘
                            │
                            │ Stage 1: Ingest
                            ▼
                    ┌────────────────┐
                    │  Stream        │
                    │  Analytics     │
                    │  (Filter)      │
                    └───────┬────────┘
                            │
                            │ Stage 2: Transform
                            ▼
                    ┌────────────────┐
                    │  Event Hub     │
                    │  (Filtered)    │
                    └───────┬────────┘
                            │
                            │ Stage 3: Enrich
                            ▼
                    ┌────────────────┐
                    │  Azure         │
                    │  Function      │
                    │  (Enrich)      │
                    └───────┬────────┘
                            │
                            │ Stage 4: Store
                            ▼
                  ┌──────────────────────┐
                  │  Cosmos DB /         │
                  │  SQL Database        │
                  └──────────────────────┘
```

### Code Pattern
```csharp
// Stage 1: Ingest and filter
var processor = new EventProcessorClient(/* config */);

processor.ProcessEventAsync += async args =>
{
    var data = JsonSerializer.Deserialize<TelemetryData>(args.Data.EventBody);
    
    // Filter: Only high-priority events
    if (data.Priority >= 8)
    {
        // Stage 2: Send to next Event Hub
        await filteredEventHubProducer.SendAsync(new EventData(
            JsonSerializer.SerializeToUtf8Bytes(data)
        ));
    }
    
    await args.UpdateCheckpointAsync();
};

// Stage 3: Enrich (Azure Function)
[FunctionName("EnrichEvents")]
public async Task Run(
    [EventHubTrigger("filtered-events")] EventData[] events)
{
    foreach (var evt in events)
    {
        var data = JsonSerializer.Deserialize<TelemetryData>(evt.EventBody);
        
        // Enrich with reference data
        data.Location = await GetLocationData(data.DeviceId);
        
        // Stage 4: Store
        await cosmosContainer.CreateItemAsync(data);
    }
}
```

### Characteristics
- ✅ Multi-stage processing
- ✅ Data transformation
- ✅ Decoupled stages
- ⚠️ Increased latency

---

## 8. Time Window Aggregation

### Description
Aggregate events over time windows (tumbling, sliding, session windows).

### Use Cases
- Real-time dashboards
- Metrics aggregation
- Anomaly detection

### ASCII Diagram
```
Event Stream:
─────────────────────────────────────────────>
E1  E2  E3  E4  E5  E6  E7  E8  E9  E10
│   │   │   │   │   │   │   │   │   │
└───────────┘   └───────────┘   └─────────
   Window 1        Window 2      Window 3
  (5 minutes)     (5 minutes)   (5 minutes)

┌──────────────────────────────────────────┐
│      Tumbling Window (5 minutes)         │
│                                          │
│  Window 1: 00:00 - 00:05                 │
│  Events: [E1, E2, E3]                    │
│  Sum: 150, Avg: 50, Count: 3             │
│                                          │
│  Window 2: 00:05 - 00:10                 │
│  Events: [E4, E5, E6]                    │
│  Sum: 220, Avg: 73, Count: 3             │
│                                          │
│  Window 3: 00:10 - 00:15                 │
│  Events: [E7, E8, E9]                    │
│  Sum: 180, Avg: 60, Count: 3             │
└──────────────────────────────────────────┘

         │ Output aggregated results
         ▼
┌──────────────────────┐
│  Real-time Dashboard │
└──────────────────────┘
```

### Code Pattern
```csharp
// Azure Stream Analytics Query (SQL-like)
SELECT
    System.Timestamp() AS WindowEnd,
    DeviceId,
    AVG(Temperature) AS AvgTemp,
    MAX(Temperature) AS MaxTemp,
    COUNT(*) AS EventCount
INTO
    [OutputEventHub]
FROM
    [InputEventHub]
GROUP BY
    DeviceId,
    TumblingWindow(minute, 5)
HAVING
    AVG(Temperature) > 25.0

// Or using .NET (manual aggregation)
var windowData = new Dictionary<string, List<double>>();
var windowDuration = TimeSpan.FromMinutes(5);
var windowStart = DateTime.UtcNow;

processor.ProcessEventAsync += async args =>
{
    var data = JsonSerializer.Deserialize<TelemetryData>(args.Data.EventBody);
    
    // Check if window expired
    if (DateTime.UtcNow - windowStart > windowDuration)
    {
        // Publish aggregated results
        foreach (var kvp in windowData)
        {
            var avg = kvp.Value.Average();
            await outputProducer.SendAsync(new EventData(
                JsonSerializer.SerializeToUtf8Bytes(new { DeviceId = kvp.Key, Average = avg })
            ));
        }
        
        // Reset window
        windowData.Clear();
        windowStart = DateTime.UtcNow;
    }
    
    // Add to current window
    if (!windowData.ContainsKey(data.DeviceId))
        windowData[data.DeviceId] = new List<double>();
    
    windowData[data.DeviceId].Add(data.Temperature);
};
```

### Characteristics
- ✅ Real-time aggregation
- ✅ Multiple window types
- ✅ Efficient for analytics
- ⚠️ State management complexity

---

## 9. Event Replay Pattern

### Description
Replay historical events from any point in time for debugging, testing, or reprocessing.

### Use Cases
- Bug investigation
- System recovery
- Testing new logic
- Data migration

### ASCII Diagram
```
┌────────────────────────────────────────────────┐
│         Azure Event Hubs                       │
│                                                │
│  Event History (Retention: 7 days)             │
│                                                │
│  Offset: 0    100   200   300   400   500      │
│          │     │     │     │     │     │       │
│  ─────┬──┴─────┴─────┴─────┴─────┴─────┴──>    │
│       │                                        │
│       │  [Day 1][Day 2][Day 3][Day 4][Day 5]  │
└───────┼────────────────────────────────────────┘
        │
        │ Replay from specific offset/time
        │
        ▼
┌─────────────────────────────────────────┐
│  Replay Scenarios:                      │
│                                         │
│  1. From Beginning (Offset 0)           │
│     → Full replay                       │
│                                         │
│  2. From Specific Offset (200)          │
│     → Partial replay                    │
│                                         │
│  3. From Timestamp (3 days ago)         │
│     → Time-based replay                 │
│                                         │
│  4. Latest (Offset -1)                  │
│     → Current processing                │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────┐
│  Processor      │
│  (Test/Debug)   │
└─────────────────┘
```

### Code Pattern
```csharp
// Replay from beginning
var consumer = new EventHubConsumerClient(
    consumerGroup, connectionString, eventHubName);

// Option 1: From earliest
await foreach (var evt in consumer.ReadEventsFromPartitionAsync(
    partitionId, 
    EventPosition.Earliest))  // Start from offset 0
{
    Console.WriteLine($"Replaying: {evt.Data.EventBody}");
}

// Option 2: From specific offset
await foreach (var evt in consumer.ReadEventsFromPartitionAsync(
    partitionId, 
    EventPosition.FromOffset(200)))  // Start from offset 200
{
    Console.WriteLine($"Replaying: {evt.Data.EventBody}");
}

// Option 3: From specific time
var startTime = DateTimeOffset.UtcNow.AddDays(-3);
await foreach (var evt in consumer.ReadEventsFromPartitionAsync(
    partitionId, 
    EventPosition.FromEnqueuedTime(startTime)))  // Start from 3 days ago
{
    Console.WriteLine($"Replaying: {evt.Data.EventBody}");
}

// Option 4: From latest (default processing)
await foreach (var evt in consumer.ReadEventsFromPartitionAsync(
    partitionId, 
    EventPosition.Latest))  // Only new events
{
    Console.WriteLine($"Processing: {evt.Data.EventBody}");
}
```

### Characteristics
- ✅ Time travel capability
- ✅ Debugging support
- ✅ Reprocessing flexibility
- ⚠️ Limited by retention period

---

## Pattern Selection Matrix

| Pattern | Use Case | Complexity | Throughput | Latency |
|---------|----------|------------|------------|---------|
| Event Streaming | IoT/Telemetry | Low | Very High | Low |
| Event Sourcing | State management | High | Medium | Medium |
| Consumer Groups | Multi-purpose processing | Low | High | Low |
| Partition Processing | Parallel workloads | Medium | Very High | Low |
| Checkpointing | Reliable processing | Medium | High | Low |
| Event Capture | Archival | Low | High | N/A |
| Stream Pipeline | Multi-stage ETL | High | Medium | Medium |
| Time Windows | Real-time analytics | High | Medium | Low |
| Event Replay | Debugging/Recovery | Low | Medium | N/A |

---

## Best Practices Summary

1. **Choose partition count wisely** - difficult to change later
2. **Use partition keys** for related events
3. **Implement checkpointing** for reliable processing
4. **Enable capture** for long-term retention
5. **Use consumer groups** for independent processing
6. **Batch send events** to improve throughput
7. **Monitor partition balance** to avoid hot partitions
8. **Set appropriate retention** based on replay needs
9. **Handle backpressure** in consumers
10. **Use EventPosition** strategically for replay scenarios

---

## Throughput Optimization

### Batching Strategy
```csharp
// Efficient batching
var batch = await producer.CreateBatchAsync();
foreach (var data in dataItems)
{
    var eventData = new EventData(Encoding.UTF8.GetBytes(data));
    if (!batch.TryAdd(eventData))
    {
        await producer.SendAsync(batch);
        batch = await producer.CreateBatchAsync();
        batch.TryAdd(eventData);
    }
}
await producer.SendAsync(batch);  // Send remaining
```

### Partition Key Usage
```csharp
// Same partition key → same partition → guaranteed ordering
var eventData = new EventData(data)
{
    PartitionKey = deviceId  // All events from same device go to same partition
};
```

