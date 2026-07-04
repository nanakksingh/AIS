## Azure Message Router — Concepts & Notes
### The Big Picture First
Think of Azure Service Bus as a **post office for your applications**. Instead of App A talking directly to App B, messages go through this post office — making systems loosely coupled, resilient, and scalable.

### Core Concepts
### Queue vs Topic vs Subscription
### Queue — One sender, one receiver

- Messages sit in a line (FIFO)
- Only one consumer picks up each message
- Classic use case: order processing, job scheduling
- Like a single checkout lane — one customer served at a time

### Topic — One sender, many receivers

- Messages are broadcast to multiple subscribers
- Each subscriber gets their own copy
- Like a newspaper — one publisher, many readers
- Use case: notifications, event fan-out

Subscription — A named filter on a Topic

Each subscription is essentially a virtual queue under a topic
Subscribers listen on their subscription, not the topic directly
You can attach SQL filters to each subscription so it only receives relevant messages
