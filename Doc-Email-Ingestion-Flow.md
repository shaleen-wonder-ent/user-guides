# Document & Email Ingestion Flow

End-to-end ingestion pipeline that feeds raw emails (Exchange / Gmail) and
inspection PDFs (Web UI / DMS / email attachments) into the **Container Apps
Environment** for downstream DocAI / InspectAI processing.

All paths are **push-based**, **scale-to-zero**, and use **Managed Identity**
(no static secrets). Every event is durable via Event Grid → Service Bus,
with dead-lettering on each hop.

---

## 1. Components

### 1.1 Email sources (push-based)

| # | Component | Purpose |
|---|-----------|---------|
| 1 | **Microsoft 365 Exchange Online** | Shared mailbox(es), app-only `Mail.Read` permission |
| 2 | **Microsoft Graph Change Notifications** | Subscription that POSTs to our webhook when a new message arrives |
| 3 | **Graph Webhook Receiver Function** (Azure Functions, Consumption) | Validates handshake, normalizes payload, publishes a CloudEvent to Event Grid |
| 4 | **Renewal Timer Function** (Timer trigger, every 48 h) | Renews Graph subscriptions before the 3-day expiry |
| 5 | **Gmail (Google Workspace mailbox)** | Source mailbox |
| 6 | **Google Cloud Pub/Sub Topic** | Gmail Watch API publishes change notifications here |
| 7 | **Gmail → Event Grid Relay Function** (HTTP trigger from Pub/Sub push) | Fetches the message, normalizes to the same CloudEvent shape, publishes to Event Grid |

### 1.2 Document sources (InspectAI inputs)

| # | Component | Purpose |
|---|-----------|---------|
| 8 | **Web UI PDF Upload** | Operator-initiated upload of inspection PDFs |
| 9 | **DMS (Document Management System)** | Existing system-of-record for inspection documents |
| 10 | **Email Attachments** | Inline PDFs extracted by the email ingestion worker |

### 1.3 Messaging & storage backbone

| # | Component | Purpose |
|---|-----------|---------|
| 11 | **Azure Event Grid — `email-events` custom topic** | Single fan-in point for all email change events |
| 12 | **Azure Service Bus — Ingestion Queue** | Durable, ordered job queue consumed by Container Apps workers |
| 13 | **Azure Blob Storage — `emails` container** | Raw `.eml` + attachments (hot → cool @ 30 d → archive @ 90 d) |
| 14 | **Azure Blob Storage — `inspection-pdfs` container** | Raw inspection PDFs from Web UI / DMS / email attachments |
| 15 | **Email Ingestion Worker** (Container App, KEDA-scaled on Service Bus depth) | Fetches the raw email, writes blobs, splits attachments, enqueues downstream jobs |

---

## 2. End-to-end flow

### 2.1 Email ingestion (Exchange Online path)

```
Exchange Online ─► Graph Change Notification ─► Graph Webhook Receiver Fn
                                                       │
                                                       │  CloudEvent
                                                       ▼
                                               Event Grid (email-events)
                                                       │
                                                       │  push subscription
                                                       ▼
                                               Service Bus Ingestion Queue
                                                       │
                                                       │  KEDA scale (queue depth)
                                                       ▼
                                               Email Ingestion Worker  ──► Blob (raw .eml + attachments)
                                                                       └─► Service Bus (job for DocAI Worker)
```

### 2.2 Email ingestion (Gmail path)

```
Gmail mailbox ─► Pub/Sub Topic ─► HTTPS push ─► Gmail → Event Grid Relay Fn
                                                       │
                                                       │  CloudEvent (same shape as Graph)
                                                       ▼
                                               Event Grid (email-events)        ◄── converges here
                                                       │
                                                       ▼
                                               (continues exactly as 2.1)
```

### 2.3 Subscription renewal (control plane)

```
Renewal Timer Fn (every 48 h) ─► Microsoft Graph: PATCH /subscriptions/{id}  (new expiry = now + 3 d)
                                  Google Gmail:   users.watch (new expiry = now + 7 d)
                                          │
                                          ├─► on success: write new expiry to Azure Table
                                          └─► on failure: Application Insights alert + retry next tick
```

#### Why this exists

Push notifications from both Microsoft Graph and Gmail are **not permanent**.
When we first call `POST /subscriptions` (Graph) or `users.watch` (Gmail), the
provider gives us back a subscription that has a **short, hard expiry**:

| Provider | Subscription type | Max lifetime | What happens at expiry |
|----------|-------------------|--------------|------------------------|
| Microsoft Graph (mail) | Webhook | **~3 days** (4230 min for mail) | Provider stops POSTing notifications to our webhook. Silent. No error. |
| Google Gmail | `users.watch` on Pub/Sub | **7 days** | Provider stops publishing to Pub/Sub. Silent. No error. |

If we don't renew before the deadline, **the entire push-based ingestion
pipeline goes dark** — Exchange/Gmail keep receiving mail, but our webhooks
never fire, nothing lands in Event Grid, the Service Bus queue stays empty,
KEDA scales the worker to zero, and we silently miss every email until
someone notices. There is no built-in retry from the provider side.

#### What the Renewal Timer does

A small **Timer-triggered Azure Function** wakes up every 48 hours (well
inside the 3-day Graph limit, with plenty of margin for transient failures):

1. **List active subscriptions** from a tracking table in Azure Storage
   (each row: `subscriptionId`, `provider`, `mailbox`, `expiresAt`).
2. For each Graph subscription, call
   `PATCH /subscriptions/{id}` with a new `expirationDateTime` = `now + 3 days`.
   This **extends** the existing subscription rather than creating a new one,
   so the `subscriptionId` and webhook URL stay the same.
3. For each Gmail mailbox, call `users.watch` again to refresh the 7-day
   Pub/Sub lease.
4. **Update the tracking table** with the new expiry.
5. **Emit a custom metric** (`graph_subscription_next_expiry_seconds`) so
   Azure Monitor can alert if any subscription's next renewal is < 6 h away
   (see §6).

#### Why every 48 h (not 72 h)?

3-day Graph limit − 48 h cadence = **24 h safety buffer**. If one renewal
run fails (Graph throttling, transient network blip, Function cold-start
problem), the next scheduled run will still catch it before the
subscription expires. A 72-hour cadence would leave zero margin.

#### What happens if it stops

This Function is the **single most critical thing keeping ingestion alive**.
If it stops running, we have ~3 days before email ingestion silently halts.
That's why §6 alerts on **both** the Function's own failure rate *and*
on any subscription whose next renewal slips inside 6 hours.

### 2.4 Document ingestion (InspectAI path)

```
Web UI Upload ─┐
DMS pull       ├─► Blob Storage (inspection-pdfs) ─► Service Bus (Document Queue) ─► InspectAI Worker
Email attach   ┘
```

---

## 3. Step-by-step (happy path, Exchange Online)

1. **Trigger.** A new message lands in the shared mailbox. Exchange Online fires a Graph change notification.
2. **Receive.** Graph POSTs to the `graph-webhook-receiver` Function with a validation token (first call) or a notification payload (subsequent calls). The Function responds **202** within 30 s.
3. **Idempotency check.** The Function reads `internetMessageId` from the notification and checks a small Azure Table (or Cosmos) for prior processing. Duplicates are dropped here.
4. **Normalize.** The Function shapes the notification into a CloudEvent v1.0:

   ```json
   {
     "specversion": "1.0",
     "type": "email.received.v1",
     "source": "exchange-online",
     "id": "{internetMessageId}",
     "subject": "{folder}/{messageId}",
     "data": {
       "tenantId": "...",
       "userId": "...",
       "messageId": "...",
       "receivedDateTime": "2026-05-19T14:02:11Z"
     }
   }
   ```

5. **Publish.** The CloudEvent is published to the `email-events` Event Grid custom topic via Managed Identity (no SAS keys).
6. **Fan-out.** An Event Grid push subscription forwards the event to the **Ingestion Queue** on Service Bus. (Dead-letter on Blob if delivery fails 5×.)
7. **Scale.** **KEDA** observes queue depth and scales the **Email Ingestion Worker** from 0 → N replicas.
8. **Fetch.** The worker uses the app-only token to call Graph `/messages/{id}/$value` and streams the raw MIME (`.eml`) and each attachment to **Blob Storage** under `emails/{yyyy}/{mm}/{dd}/{messageId}/`.
   - Attachments > 25 MB are streamed directly, never buffered in memory.
9. **Filter.** Apply configured rules (sender allow-list, subject patterns, attachment type). Filtered-out items are tagged and short-circuited to an audit-only path.
10. **Hand-off.** The worker enqueues a downstream job to the DocAI Service Bus queue with `{ blobUri, source, internetMessageId, receivedAt }`.
11. **Audit.** An `EMAIL_FETCHED` event is appended to **Azure SQL Ledger**, and the blob digest is written to the **WORM** audit container.

The Gmail path is identical from step 5 onward; only steps 1–4 differ.

---

## 4. Reliability, retries & poison handling

| Concern | Mechanism |
|---------|-----------|
| Webhook timeouts | Function returns 202 fast; heavy work happens in the worker, not the function |
| Lost notifications | Periodic Graph **delta query** reconciliation (configurable, default every 15 min) |
| Subscription expiry | `renewal-timer-fn` runs every 48 h (Graph max lifetime = 3 days) |
| Event Grid delivery failure | Built-in exponential retry (24 h), then dead-letter → `eventgrid-deadletter` blob container |
| Service Bus poison message | `MaxDeliveryCount = 5` → dead-letter sub-queue → Azure Monitor alert |
| Duplicate emails | Idempotency on `internetMessageId` (table check at step 3) |
| Blob write failure | Worker retries with exponential backoff; if all fail, the SB message is abandoned and re-delivered |
| Region outage | Blob = RA-GRS, Service Bus = Premium ZR, Event Grid = regional with DR pair |

---

## 5. Identity, secrets, network

- **All compute uses System-Assigned Managed Identity.** No connection strings in env vars.
  - Functions → Event Grid: `EventGrid Data Sender` role
  - Worker → Blob: `Storage Blob Data Contributor` (scoped to `emails` container)
  - Worker → Service Bus: `Azure Service Bus Data Sender` + `Receiver`
- **Graph & Gmail OAuth client secrets** live in **Azure Key Vault**, retrieved via Container Apps Key Vault references.
- All Azure-to-Azure traffic stays on the **VNet via Private Endpoints**. No public ingress on Storage, Service Bus, or Event Grid.
- Inbound webhook endpoints (Graph, Pub/Sub) terminate at **Azure Front Door + WAF + DDoS Standard** and are then routed through the Container Apps Environment ingress / dedicated Function endpoints.

---

## 6. Observability

| Signal | Source | Alert threshold |
|--------|--------|-----------------|
| End-to-end latency (`EMAIL_RECEIVED` → `EMAIL_FETCHED`) | Application Insights distributed trace | p95 > 60 s for 5 min |
| Service Bus queue depth | Azure Monitor | depth > 5 000 for 10 min |
| Dead-letter count (Event Grid + Service Bus) | Azure Monitor | any non-zero delta in 5 min |
| Graph subscription expiry | Custom metric from renewal Fn | next expiry < 6 h |
| Webhook 5xx rate | Functions host metric | > 1 % over 5 min |

Every event also writes a structured audit record to **Azure SQL Ledger** (append-only, cryptographic chain) and a digest to **Blob WORM** with a 7-year retention policy.

---

## 7. Why this shape

- **Push, not poll** → near-real-time (sub-second from Exchange/Gmail to Event Grid) and cheap (no idle compute).
- **Event Grid as the single fan-in** → Gmail and Exchange normalize to one event schema, downstream code doesn't care about the source.
- **Service Bus between Event Grid and workers** → gives us ordered, durable, dead-letterable work units with KEDA-driven horizontal scale.
- **Functions only at the edge** → tiny, stateless adapters for the two SaaS notification protocols; everything heavy runs in the Container Apps Environment.
- **Blob first, then enqueue** → the SB message carries only a blob URI + metadata, so messages stay small (< 4 KB) and replays are cheap.
