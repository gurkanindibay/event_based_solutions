# Azure Event Grid Detailed Reference

## 1. Overview
Azure Event Grid is a highly scalable, serverless event broker that lets you integrate applications using events. It simplifies building event-driven architectures.

- **Primary Use Case:** Reactive programming, ops automation, serverless triggers.
- **Mechanism:** Push-push (Event Grid pushes to the subscriber).

## 2. Core Concepts

### Events
The smallest amount of information that fully describes something that happened in the system.

### Event Sources (Publishers)
Where the event comes from.
- **Azure Services:** Blob Storage, Resource Groups, Subscriptions, IoT Hub, Service Bus, etc.
- **Custom Sources:** Your own applications sending events to a Custom Topic.
- **SaaS Sources:** Partner events (e.g., Auth0).

### Topics
The endpoint where publishers send events.
- **System Topics:** Built-in topics for Azure services (hidden management).
- **Custom Topics:** Application-specific topics you create.
- **Domains:** Management tool for large numbers of topics (e.g., one topic per customer for a SaaS app).

### Event Subscriptions
The mechanism to route events from a topic to a handler. Contains filtering logic.

### Event Handlers
The app or service reacting to the event.
- **Supported:** Azure Functions, Logic Apps, Webhooks, Event Hubs, Service Bus Queue/Topic, Storage Queue.

## 3. Filtering
Event Grid allows filtering at the subscription level to ensure subscribers only receive relevant events.
- **Event Type Filtering:** Subscribe only to `Microsoft.Storage.BlobCreated`.
- **Subject Filtering:** "Begins with" (`/blobServices/default/containers/logs/`) or "Ends with" (`.jpg`).
- **Advanced Filtering:** Filter based on values in the data payload (e.g., `data.size > 1024`).

## 4. Data Integration Model: Push-Push

Event Grid follows a **push-push** delivery model:

### Publisher Side (Push)
- **Event sources actively push events** to Event Grid topics.
- Publishers (Azure services or custom applications) send events via HTTP POST to the Event Grid endpoint.
- No polling required from Event Grid; events arrive as they occur.

### Subscriber Side (Push)
- **Event Grid actively pushes events** to subscribers.
- Subscribers (webhooks, Azure Functions, Logic Apps, etc.) receive events via HTTP POST.
- Event Grid manages the delivery, retry logic, and failure handling.
- Subscribers must expose an HTTP endpoint to receive events.

### Benefits
- **Low latency:** Events are delivered immediately as they occur.
- **Serverless-friendly:** No need for subscribers to maintain polling loops.
- **Decoupling:** Publishers don't need to know about subscribers; Event Grid handles routing.

### Considerations
- Subscribers must be able to handle incoming HTTP requests.
- Endpoint must be publicly accessible or use private endpoints.
- Requires endpoint validation before event delivery starts.

## 5. Architecture Patterns
### Reactive Automation
- Blob Created → Event Grid → Azure Function → Database Update

## 6. Best Practices
- **Scaling:** Auto-scaling; monitor subscriber endpoints for throttling.
- **Reliability:** Implement idempotent event handlers.

## 6. Advanced Features

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
