## Azure Message Router — Concepts & Notes
### The Big Picture First
Think of Azure Service Bus as a **post office for your applications**. Instead of App A talking directly to App B, messages go through this post office — making systems loosely coupled, resilient, and scalable.

### Core Concepts
### Queue vs Topic vs Subscription
### Queue — One sender, one receiver

- Messages sit in a line (FIFO)
- Only one **consumer** picks up each message
- Classic use case: order processing, job scheduling
- Like a single checkout lane — one customer served at a time

### Topic — One sender, many receivers

- Messages are **broadcast** to multiple subscribers
- Each subscriber gets their own copy
- Like a newspaper — one publisher, many readers
- Use case: notifications, event fan-out

### Subscription — A named filter on a Topic

- Each subscription is essentially a virtual queue under a topic
- Subscribers listen on their subscription, not the topic directly
- You can attach SQL filters to each subscription so it only receives relevant messages

### SQL Filters on Subscriptions
Service Bus lets you write SQL-like expressions on **message properties** (not body):
-- Only retail orders land here
orderType = 'retail'

-- High-value wholesale
orderType = 'wholesale' AND orderValue > 10000

-- Catch-all / default rule
1 = 1

⚠️ Filters work on custom message properties (the metadata/headers), not the message body. You set these when sending the message.

### Three filter types:
**SQL Filter** — full expression-based, most flexible
**Correlation Filter** — matches exact property values, faster & cheaper
**True Filter** — 1=1, accepts everything (default)

### Peek-Lock vs Receive-and-Delete
 
| | Peek-Lock | Receive-and-Delete |
|---|---|---|
| Message deleted | Only after `Complete()` | Immediately on receive |
| If processing fails | Message reappears after lock timeout | Message gone forever |
| Delivery guarantee | At-least-once | At-most-once |
| Use when | Processing matters | Fire-and-forget |
 
**Peek-Lock flow:**
1. Consumer locks the message (invisible to others)
2. On success → `Complete()` → message deleted
3. On failure → `Abandon()` or lock expires → message back in queue
4. After MaxDeliveryCount failures → moved to Dead-Letter Queue
---
 
### Dead-Letter Queue (DLQ)
 
Every queue and subscription gets a DLQ automatically.
 
**Messages land in DLQ when:**
- MaxDeliveryCount exceeded (default 10, we use 3 for testing)
- TTL expires before processing
- Consumer explicitly calls `DeadLetter()`
- Filter evaluation error
**DLQ paths:**
```
incoming-orders/$DeadLetterQueue
orders-topic/retail-orders/$DeadLetterQueue
```
 
---
 
### MaxDeliveryCount Behavior
 
```
Message arrives
  → Attempt 1: fails, lock expires
  → Attempt 2: fails again
  → Attempt 3: still failing
  → 💀 Auto-moved to Dead-Letter Queue
```
 
- Each `Abandon()` or lock timeout increments the count
- `Defer()` does NOT increment the count
---
 
### Logic Apps Service Bus Trigger
 
- Uses **polling**, not push
- Checks queue every N seconds based on configured frequency
- Consumption plan — pay per action execution (~₹0.001 per run)
---
 
### Base64 Encoding
 
Service Bus delivers message body as Base64 in Logic Apps.
 
```
eyJvcmRlcklkIjogIkkwMDEi...
```
 
decodes to:
 
```json
{"orderId": "I001", "orderType": "international", "amount": 2200, "customer": "Carlos SA"}
```
 
Always decode in Parse JSON using:
 
```
base64ToString(triggerBody()?['ContentData'])
```
 
---
 
### Service Bus Tiers
 
| | Basic | Standard | Premium |
|---|---|---|---|
| Queues | ✅ | ✅ | ✅ |
| Topics & Subscriptions | ❌ | ✅ | ✅ |
| SQL Filters | ❌ | ✅ | ✅ |
| Max message size | 256 KB | 256 KB | 100 MB |
| VNET integration | ❌ | ❌ | ✅ |
| Cost | Lowest | ~₹70/month | High |
 
> This project requires **Standard** tier minimum.
 
---
 
### SQL Filter vs Correlation Filter
 
| | SQL Filter | Correlation Filter |
|---|---|---|
| How it works | Expression engine | Hash-table lookup |
| Flexibility | Any expression | Exact match only |
| Performance | Slower | Faster |
| Cost | Higher | Lower |
| Use when | Complex conditions | Simple equality checks |
 
---
 
## Step-by-Step Build
 
### Phase 1 — Service Bus Namespace & Queue
 
**Step 1 — Create Resource Group**
- Portal → Resource Groups → Create
- Name: `rg-message-router`
- Region: East US
**Step 2 — Create Service Bus Namespace**
- Portal → Service Bus → Create
- Name: `sb-orderrouter-[yourname]`
- Tier: **Standard** (mandatory — Basic has no Topics)
- Resource group: `rg-message-router`
**Step 3 — Create incoming-orders Queue**
- Namespace → Entities → Queues → + Queue
- Name: `incoming-orders`
- Max delivery count: `3`
- Lock duration: `30 seconds`
- Enable dead lettering on message expiration: ✅
**Step 4 — Save Connection String**
- Namespace → Settings → Shared access policies → RootManageSharedAccessKey
- Copy **Primary Connection String** → save in Notepad
---
 
### Phase 2 — Topic & Subscriptions
 
**Step 1 — Create Topic**
- Namespace → Entities → Topics → + Topic
- Name: `orders-topic`
**Step 2 — Create Three Subscriptions**
 
Under `orders-topic` → Subscriptions → + Subscription (repeat 3 times):
 
| Name | Max delivery count | Dead-letter on expiry |
|---|---|---|
| `retail-orders` | 3 | ✅ |
| `wholesale-orders` | 3 | ✅ |
| `international-orders` | 3 | ✅ |
 
---
 
### Phase 3 — SQL Filters
 
For each subscription:
1. Subscription → Filters
2. Delete the `$Default` filter (1=1)
3. Add new filter → SQL Filter
| Subscription | Filter name | SQL expression |
|---|---|---|
| retail-orders | retail-filter | `orderType = 'retail'` |
| wholesale-orders | wholesale-filter | `orderType = 'wholesale'` |
| international-orders | intl-filter | `orderType = 'international'` |
 
> ⚠️ Always delete the default filter first — otherwise messages duplicate across all subscriptions.
 
---
 
### Phase 4 — Logic App: la-order-router
 
**Step 1 — Create Logic App**
- Portal → Logic Apps → + Add
- Name: `la-order-router`
- Plan type: **Consumption**
- Resource group: `rg-message-router`
**Step 2 — Trigger**
- Designer → Blank Logic App
- Search: Service Bus → `When a message is received in a queue (peek-lock)`
- Connection: paste connection string → name `sb-connection`
- Queue name: `incoming-orders`
- Frequency: `30 seconds`
**Step 3 — Parse JSON Action**
- \+ New step → Parse JSON
- Content (switch to expression mode):
```
base64ToString(triggerBody()?['ContentData'])
```
- Schema:
```json
{
  "type": "object",
  "properties": {
    "orderId": { "type": "string" },
    "orderType": { "type": "string" },
    "amount": { "type": "number" },
    "customer": { "type": "string" }
  }
}
```
 
**Step 4 — Switch Action**
- \+ New step → Control → Switch
- On: `orderType` (from Parse JSON dynamic content)
| Case | Equals | Action | Properties to set |
|---|---|---|---|
| Case 1 | `retail` | Send message to `orders-topic` | Key: `orderType` Value: `retail` |
| Case 2 | `wholesale` | Send message to `orders-topic` | Key: `orderType` Value: `wholesale` |
| Case 3 | `international` | Send message to `orders-topic` | Key: `orderType` Value: `international` |
 
For each Send message action:
1. Topic name: `orders-topic`
2. Content: `Body` from Parse JSON dynamic content
3. Add new parameter → Properties → Add new item
4. Key: `orderType` → Value: `retail` / `wholesale` / `international`
**Step 5 — Complete the Message**
- \+ New step (after Switch) → Service Bus → `Complete the message in a queue (peek-lock)`
- Queue name: `incoming-orders`
- Lock token: `Lock Token` from trigger dynamic content
---
 
### Phase 5 — Logic App: la-dlq-alerter
 
**Step 1 — Create Logic App**
- Name: `la-dlq-alerter`
- Plan: Consumption
**Step 2 — Trigger**
- Service Bus → `When a message is received in a queue (peek-lock)`
- Queue name: `incoming-orders/$DeadLetterQueue`
- Frequency: `1 minute`
**Step 3 — Send Email Action**
- \+ New step → Office 365 Outlook → `Send an email (V2)`
- To: your email
- Subject: `⚠️ Dead-lettered order message`
- Body:
```
Message ID:     @{triggerBody()?['MessageId']}
Enqueued:       @{triggerBody()?['EnqueuedTimeUtc']}
Delivery Count: @{triggerBody()?['DeliveryCount']}
Body:           @{base64ToString(triggerBody()?['ContentData'])}
```
 
**Step 4 — Complete DLQ Message**
- Service Bus → `Complete the message in a queue (peek-lock)`
- Queue name: `incoming-orders/$DeadLetterQueue`
- Lock token: `Lock Token` from trigger
---
 
### Phase 6 — Test
 
**Send test messages (Python)**
 
```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage
import json
 
CONN_STR = "Endpoint=sb://sb-orderrouter-..."
QUEUE = "incoming-orders"
 
orders = [
    {"orderId": "R001", "orderType": "retail", "amount": 150, "customer": "Alice"},
    {"orderId": "W001", "orderType": "wholesale", "amount": 5000, "customer": "Bob Corp"},
    {"orderId": "I001", "orderType": "international", "amount": 2200, "customer": "Carlos SA"},
]
 
with ServiceBusClient.from_connection_string(CONN_STR) as client:
    with client.get_queue_sender(QUEUE) as sender:
        for order in orders:
            msg = ServiceBusMessage(
                json.dumps(order),
                application_properties={"orderType": order["orderType"]}
            )
            sender.send_messages(msg)
            print(f"Sent: {order['orderId']} [{order['orderType']}]")
 
print("All messages sent!")
```
 
**Verify routing**
- `incoming-orders` queue → active message count drops to 0
- Each subscription shows 1 active message
- `la-order-router` → Run history → 3 Succeeded runs
**Test DLQ path**
1. Disable `la-order-router`
2. Send one message via Python
3. Wait for TTL expiry or manually dead-letter via Service Bus Explorer
4. Check email — alert should arrive within 1 minute
5. Re-enable `la-order-router`
**Peek messages without consuming**
- Namespace → Queue or Subscription → Service Bus Explorer
- Mode: Peek → select Active or DeadLetter from dropdown
- Click any message to inspect body, properties, DeliveryCount, EnqueuedTime
---
 
### Cleanup
 
```
Portal → Resource Groups → rg-message-router → Delete resource group
```
 
Deletes everything — namespace, queues, topics, subscriptions, both Logic Apps.
 
---
 
## Key Interview Questions
 
| Question | Answer |
|---|---|
| Why Standard tier? | Basic has no Topics or Subscriptions |
| Queue vs Topic | Queue = one consumer; Topic = multiple subscribers |
| Peek-Lock vs Receive-Delete | Peek-Lock = safe, at-least-once; Receive-Delete = fast, at-most-once |
| What triggers DLQ? | MaxDeliveryCount exceeded, TTL expired, explicit DeadLetter(), filter error |
| SQL filter limitation | Reads message properties, not the body |
| How Logic Apps triggers | Polling-based, not push |
| Why base64ToString()? | Logic Apps receives Service Bus body as Base64 encoded |
| Correlation vs SQL filter | Correlation = faster equality match; SQL = flexible expressions |
| What is lock duration? | Time a message stays locked while being processed before returning to queue |
| Does Defer() increment delivery count? | No — only Abandon() and lock timeout do |
