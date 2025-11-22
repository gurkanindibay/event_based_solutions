# Azure Event Hubs Detailed Reference

## 1. Overview
Azure Event Hubs is a big data streaming platform and event ingestion service. It can receive and process millions of events per second. Data sent to an event hub can be transformed and stored using any real-time analytics provider or batching/storage adapters.

- **Primary Use Case:** Telemetry ingestion, log aggregation, real-time analytics.
- **Protocol Support:** AMQP, HTTPS, Apache Kafka.

## 2. Core Concepts

### Namespace
A management container for Event Hubs. It provides a unique FQDN and serves as a container for multiple Event Hub instances.

### Partitions
- **Definition:** Ordered sequences of events that are held in an Event Hub.
- **Function:** Partitions are the mechanism for parallelism. As data volume grows, you increase partitions to handle higher throughput.
- **Retention:** Events are retained for a configurable period (1-90 days depending on tier) and cannot be deleted explicitly; they expire.

### Consumer Groups
- **Definition:** A view (state, position, or offset) of an entire event hub.
- **Function:** Enable consuming applications to each have a separate view of the event stream. They read the stream independently at their own pace and with their own offsets.

### Throughput Units (TUs) / Processing Units (PUs)
- **Standard Tier:** Uses TUs. 1 TU = 1 MB/s or 1000 events/s ingress, 2 MB/s or 4096 events/s egress.
- **Premium/Dedicated:** Uses PUs or CUs for isolated resources and predictable latency.

## 3. Key Features

### Event Capture
Automatically capture the streaming data in Azure Blob Storage or Azure Data Lake Storage Gen 2.
- **Format:** Avro.
- **Trigger:** Time-based (e.g., every 5 mins) or Size-based (e.g., every 100 MB).

### Apache Kafka Compatibility
Event Hubs provides an endpoint compatible with Kafka producer and consumer APIs. You can use existing Kafka applications without running your own Kafka cluster.
### Schema Registry Considerations
- **No built-in Confluent registry:** Event Hubs does not include a Confluent Schema Registry service. Schema registries are separate services and must be hosted/available to your clients.
- **Typically no code changes:** If your Kafka producers/consumers already use an external schema registry (e.g., Confluent, Apicurio) and standard Kafka serializers (Avro/Protobuf), you usually only need to update serializer configuration (registry URLs and auth) to point to the registry; your application code typically does not require changes.
- **Using Azure Schema Registry:** If you adopt Azure Schema Registry instead of your existing registry, you may need to switch to Azure-compatible serializer libraries or update serializer configuration; this can require minor code or dependency changes.
- **Network & auth:** Ensure any registry (Confluent or Azure) is reachable from your client environment and that client serializers are configured with the correct authentication and TLS settings.

### Checkpointing
Consumers store their position in the partition stream. If a worker fails, a new one picks up from the last checkpoint.

## 4. Data Integration Model: Push-Pull

Event Hubs follows a **push-pull** delivery model:

### Publisher Side (Push)
- **Producers actively push events** to Event Hubs.
- Events are sent via AMQP, HTTPS, or Kafka protocols.
- Producers send events to partitions (either explicitly or via partition key/round-robin).
- Events are immediately written to the partition and persisted.

### Consumer Side (Pull)
- **Consumers actively pull events** from Event Hubs partitions.
- Consumers read from specific partitions at their own pace.
- Each consumer group maintains its own offset (position) in the partition.
- Consumers control when and how fast they read data.

### Benefits
- **High throughput:** Multiple consumers can read from the same partition independently.
- **Replay capability:** Consumers can reset their offset and re-read historical data.
- **Backpressure handling:** Consumers control their read rate, preventing overload.
- **Scalability:** Add partitions and consumers independently.

### Considerations
- Consumers must implement polling logic and checkpoint management.
- Requires consumer-side state management (tracking offsets).
- More complex consumer implementation compared to push models.
- Network bandwidth consumed by continuous polling.

## 5. Architecture & Flow
```
[Producers] --> [Event Hub Namespace]
                    |
                    +--> [Event Hub A]
                          |--> [Partition 1] --+--> [Consumer Group A] --> [App Instance 1]
                          |--> [Partition 2] --|
                          |--> [Partition 3] --+--> [Consumer Group B] --> [Stream Analytics]
```

## 6. Access Control (Security)

### Authentication
Event Hubs supports two primary authentication mechanisms:

1. **Microsoft Entra ID (Recommended):**
   - Authenticate using Azure AD identities (users, groups, or managed identities).
   - Eliminates the need to store connection strings in code.
   - Supports OAuth 2.0.

2. **Shared Access Signatures (SAS):**
   - Uses cryptographic keys to generate tokens with specific permissions and expiry times.
   - Useful for clients that cannot use Entra ID.
   - Can be defined at the Namespace or Event Hub level.

### Application Identity Strategy
Applications connecting to Event Hubs need an identity to be authenticated and authorized.

1. **Managed Identities (Recommended for Azure-hosted apps):**
   - If your app runs on Azure (VM, App Service, AKS, Functions, Container Apps), enable a **System-Assigned** or **User-Assigned** Managed Identity.
   - **No App Registration needed:** The identity is managed by the Azure platform.
   - Grant RBAC roles (e.g., *Azure Event Hubs Data Receiver*) directly to the Managed Identity.
   - **Benefit:** No secrets to rotate or store in code.

2. **App Registrations (Service Principals) (For external/local apps):**
   - If your app runs outside Azure (on-prem, other clouds, or local dev), create an **App Registration** in Entra ID.
   - This creates a **Service Principal**.
   - You must manage a **Client Secret** or **Certificate**.
   - Grant RBAC roles to the Service Principal.
   - **Benefit:** Secure, role-based access for external applications.

### Authorization
Authorization determines what an authenticated principal can do.

#### Azure RBAC (Role-Based Access Control)
When using Entra ID, assign built-in roles to scopes (Resource Group, Namespace, or Event Hub):
- **Azure Event Hubs Data Owner:** Full access to data (send and receive).
- **Azure Event Hubs Data Sender:** Send access only.
- **Azure Event Hubs Data Receiver:** Receive access only.

#### SAS Policies
When using SAS, policies grant specific rights:
- **Send:** Permission to send events.
- **Listen:** Permission to receive events.
- **Manage:** Permission to manage topology (create/delete consumer groups, etc.) + Send + Listen.

## 7. Best Practices
- **Partition Count:** Set at creation (difficult to change later). Match roughly to expected concurrent consumers.
- **Batching:** Send events in batches to improve throughput.
- **Security:** Use Shared Access Signatures (SAS) or Azure Active Directory (Entra ID) for access control.
- **Geo-Recovery:** Use Geo-Disaster Recovery to replicate namespace metadata (not data) to a secondary region.
