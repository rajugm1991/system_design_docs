# Event-Driven Architecture: Simple Guide with Diagrams

## 📚 Table of Contents
1. [What is Event-Driven Architecture?](#what-is-event-driven-architecture)
2. [Simple Example: How It Works](#simple-example-how-it-works)
3. [Before vs After: Visual Comparison](#before-vs-after-visual-comparison)
4. [Kafka Basics: Key Concepts](#kafka-basics-key-concepts)
5. [Real-World Flow: Step by Step](#real-world-flow-step-by-step)
6. [Scaling: How Kafka Handles Load](#scaling-how-kafka-handles-load)
7. [Code Examples](#code-examples)
8. [Best Practices](#best-practices)

---

# Apache Kafka Overview

## What is Kafka?
Apache Kafka is a distributed event streaming platform used for building real-time data pipelines and streaming applications. It is designed for:
- High throughput
- Fault tolerance
- Scalability
- Real-time event streaming

---

## Why Kafka?
- Handles **millions of events per second**
- Highly scalable using **partitions**
- Fault-tolerant using **replication**
- Decouples microservices (event-driven architecture)

---

# Core Concepts

## 1. Topic
A category where messages (events) are stored.

## 2. Partition
A topic is divided into partitions for:
- Parallelism
- High throughput
- Ordering inside a partition
Example:  
`orders` topic → `P0, P1, P2`

## 3. Producer
Sends messages to Kafka topics.

## 4. Consumer
Reads messages from topics.

## 5. Consumer Group
Multiple consumers collaborating to read a topic.
- Each partition is consumed by only **one** consumer within a group
- Ensures load balancing

## 6. Broker
A single Kafka server.

## 7. Replication
Each partition has copies across brokers for fault tolerance.

---

# Kafka Example Flow

1. Producer sends:



4. Consumers process messages in parallel.

---

# Real-World Use Cases
- Microservices communication
- Payment events
- Order-processing (e-commerce)
- Logging pipelines
- Fraud detection
- User activity tracking
- Monitoring and analytics

---

# Interview Questions (with Answers)

## 1. What is Kafka?
A distributed event streaming system for high-throughput real-time data pipelines.

## 2. What is a partition?
A unit of parallelism that stores ordered messages.  
More partitions → more consumers → more throughput.

## 3. What is a consumer group?
A group of consumers that share work.  
Each partition is consumed by exactly one consumer in the group.

## 4. How does Kafka achieve fault tolerance?
- Replication across brokers
- Leader-follower architecture

## 5. How does Kafka achieve ordering?
Order is guaranteed **only within a partition**, not across partitions.

## 6. Is Kafka pull-based or push-based?
Kafka is **pull-based**.

## 7. How does producer decide which partition to write to?
- By key hashing
- Round-robin (if key not provided)
- Custom partitioner

## 8. When does rebalance happen?
When:
- A consumer joins the group
- A consumer leaves
- Partitions change

## 9. What is replication factor?
Number of copies of each partition:
- RF=3 → 1 leader + 2 followers

## 10. How to scale consumers?
Increase number of partitions.

---

# End of Document


## What is Event-Driven Architecture?

### Simple Explanation

**Think of it like a newspaper:**

```
┌─────────────────┐
│   Newspaper     │  ← Kafka (Event Broker)
│   Publisher     │
└─────────────────┘
         │
         │ Publishes news
         ▼
┌─────────────────┐
│   Subscribers   │
│  (Consumers)    │
│                 │
│  📰 Person A    │  ← Reads sports news
│  📰 Person B    │  ← Reads business news
│  📰 Person C    │  ← Reads all news
└─────────────────┘
```

**In our system:**
- **Publisher** = Order Service (creates order)
- **Newspaper** = Kafka (stores events)
- **Subscribers** = Inventory, Payment, Notification Services (react to order)

### Key Concepts

```
┌──────────────┐
│   EVENT      │  = Something that happened
│              │    Example: "Order #123 created"
└──────────────┘
       │
       ▼
┌──────────────┐
│   PRODUCER   │  = Service that creates events
│              │    Example: Order Service
└──────────────┘
       │
       │ Sends event
       ▼
┌──────────────┐
│   KAFKA      │  = Message broker (like a mailbox)
│   (Broker)   │    Stores events temporarily
└──────────────┘
       │
       │ Delivers event
       ▼
┌──────────────┐
│   CONSUMER   │  = Service that reacts to events
│              │    Example: Inventory Service
└──────────────┘
```

---

## Simple Example: How It Works

### Scenario: User Places an Order

```
Step 1: User clicks "Buy Now"
        │
        ▼
┌─────────────────────┐
│   Order Service     │
│                     │
│  1. Save order      │
│  2. Send event      │  ← "Order #123 created!"
└─────────────────────┘
        │
        │ Event: OrderCreated
        ▼
┌─────────────────────┐
│      KAFKA          │
│   Topic: orders     │  ← Stores the event
└─────────────────────┘
        │
        │ Broadcasts to all subscribers
        ├──────────┬──────────┬──────────┐
        ▼          ▼          ▼          ▼
   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
   │Inventory│ │Payment │ │Email   │ │Analytics│
   │Service │ │Service │ │Service │ │Service │
   │        │ │        │ │        │ │        │
   │Reserve │ │Charge  │ │Send    │ │Track   │
   │Stock   │ │Card    │ │Email   │ │Sale    │
   └────────┘ └────────┘ └────────┘ └────────┘
```

**Key Point:** Order Service doesn't wait! It sends the event and responds immediately to the user.

---

## Before vs After: Visual Comparison

### ❌ OLD WAY: Direct Service Calls (Synchronous)

```
User Request
    │
    ▼
┌──────────────┐
│ Order Service│
└──────────────┘
    │
    ├───→ Inventory Service (wait 100ms) ⏳
    │
    ├───→ Payment Service (wait 200ms) ⏳
    │
    ├───→ Email Service (wait 50ms) ⏳
    │
    └───→ Analytics Service (wait 30ms) ⏳
    
Total Time: 380ms ⏱️
User waits: 380ms 😞
```

**Problems:**
- Slow (waits for all services)
- If one service fails, everything fails
- Hard to add new services

### ✅ NEW WAY: Event-Driven (Asynchronous)

```
User Request
    │
    ▼
┌──────────────┐
│ Order Service│
│              │
│ 1. Save order│
│ 2. Send event│  ← Takes only 5ms!
└──────────────┘
    │
    │ Event sent
    ▼
┌──────────────┐
│    KAFKA     │  ← Stores event
└──────────────┘
    │
    │ (Services process in background)
    ├───→ Inventory Service (processes async)
    ├───→ Payment Service (processes async)
    ├───→ Email Service (processes async)
    └───→ Analytics Service (processes async)
    
Total Time: 5ms ⚡
User waits: 5ms 😊
```

**Benefits:**
- Fast (responds immediately)
- If one service fails, others still work
- Easy to add new services

---

## Kafka Basics: Key Concepts

### 1. Topic = Category of Events

```
┌─────────────────────────────────────┐
│         KAFKA BROKER                │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   Topic: order-created       │  │
│  │   (Like a folder)            │  │
│  │                              │  │
│  │  [Event1] [Event2] [Event3] │  │
│  └──────────────────────────────┘  │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   Topic: payment-processed   │  │
│  │                              │  │
│  │  [Event1] [Event2]           │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
```

### 2. Partition = Sub-folder for Parallel Processing

```
Topic: order-created
│
├─── Partition 0 ────┐
│    [Order1]        │
│    [Order4]        │  ← Consumer 1 processes these
│    [Order7]        │
│                    │
├─── Partition 1 ────┤
│    [Order2]        │
│    [Order5]        │  ← Consumer 2 processes these
│    [Order8]        │
│                    │
└─── Partition 2 ────┘
     [Order3]        │
     [Order6]        │  ← Consumer 3 processes these
     [Order9]        │
```

**Why Partitions?**
- Allows multiple consumers to work in parallel
- 3 partitions = 3 consumers can work simultaneously
- Faster processing!

### 3. Consumer Group = Team of Workers

```
Consumer Group: inventory-service-group
│
├─── Consumer 1 (Server 1) ────→ Partition 0
│
├─── Consumer 2 (Server 2) ────→ Partition 1
│
└─── Consumer 3 (Server 3) ────→ Partition 2

All working in parallel! 🚀
```

**Visual Example:**

```
┌─────────────────────────────────────────┐
│      KAFKA: order-created Topic         │
│                                          │
│  P0: [O1] [O4] [O7]  ←── Consumer 1    │
│  P1: [O2] [O5] [O8]  ←── Consumer 2    │
│  P2: [O3] [O6] [O9]  ←── Consumer 3    │
└─────────────────────────────────────────┘
```

---

## Real-World Flow: Step by Step

### ⚠️ Important: Payment Confirmation Flow

**Question:** Without payment confirmation, how will order be processed?

**Answer:** We need to wait for payment confirmation before processing the order! Here's the correct flow:

### Complete Order Flow with Payment Confirmation

```
┌─────────────────────────────────────────────────────────┐
│              STEP 1: User Places Order                  │
└─────────────────────────────────────────────────────────┘

👤 User clicks "Place Order"
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│              STEP 2: Order Created (PENDING)            │
└─────────────────────────────────────────────────────────┘

┌──────────────────────┐
│   Order Service      │
│                      │
│  1. ✅ Save to DB    │
│     Status: PENDING  │
│  2. 📤 Send Event    │  ← "Order #123 created!"
└──────────────────────┘
    │
    │ Event: order-created
    ▼
┌──────────────────────────────────────┐
│         KAFKA BROKER                 │
│                                      │
│  Topic: order-created                │
│  [Order #123 - Status: PENDING]      │
└──────────────────────────────────────┘
    │
    │ Only Payment Service reacts!
    ▼
┌─────────────────────────────────────────────────────────┐
│              STEP 3: Payment Processing                 │
└─────────────────────────────────────────────────────────┘

┌──────────────────────┐
│   Payment Service    │
│                      │
│  1. 💳 Charge Card   │
│  2. ✅ Payment OK    │
│  3. 📤 Send Event    │  ← "Payment confirmed!"
└──────────────────────┘
    │
    │ Event: payment-confirmed
    ▼
┌──────────────────────────────────────┐
│         KAFKA BROKER                 │
│                                      │
│  Topic: payment-confirmed            │
│  [Order #123 - Payment: SUCCESS]     │
└──────────────────────────────────────┘
    │
    │ NOW other services can process!
    ├──────────┬──────────┬──────────┐
    ▼          ▼          ▼          ▼
┌─────────────────────────────────────────────────────────┐
│        STEP 4: Order Processing (After Payment)         │
└─────────────────────────────────────────────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Inventory   │  │   Shipping   │  │ Notification │  │  Analytics   │
│   Service    │  │   Service    │  │   Service    │  │   Service    │
│              │  │              │  │              │  │              │
│ ✅ Reserve   │  │ ✅ Prepare   │  │ ✅ Send      │  │ ✅ Track     │
│    Stock     │  │    Shipment  │  │    Email     │  │    Sale      │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### Order States Flow

```
Order Status Flow:

PENDING ──→ (Payment Processing) ──→ PAID ──→ (Processing) ──→ SHIPPED ──→ DELIVERED
   │              │                    │            │              │
   │              │                    │            │              │
   └──────────────┴────────────────────┴────────────┴──────────────┘
                    If payment fails
                         │
                         ▼
                    CANCELLED
```

### Visual Timeline with Payment

```
Time →
│
├─ 0ms:   User clicks "Place Order"
│
├─ 10ms:  Order saved (Status: PENDING)
│
├─ 15ms:  Event: order-created sent
│
├─ 20ms:  ✅ Response to user: "Order placed, processing payment..."
│
│         (Background: Payment processing)
│
├─ 50ms:  Payment Service: Charging card...
│
├─ 200ms: Payment Service: ✅ Payment successful!
│
├─ 205ms: Event: payment-confirmed sent
│
│         (NOW other services can process)
│
├─ 250ms: Inventory Service: ✅ Stock reserved
│
├─ 300ms: Shipping Service: ✅ Shipment prepared
│
├─ 350ms: Email Service: ✅ Confirmation sent
│
└─ 400ms: Analytics Service: ✅ Sale tracked
```

**Key Point:** User gets response in 20ms, but order processing only starts AFTER payment confirmation!

### Payment-First Flow (Correct Approach)

```
┌─────────────────────────────────────────────────────────┐
│              CORRECT FLOW: Payment First                │
└─────────────────────────────────────────────────────────┘

Order Created Event
    │
    ▼
┌──────────────────────┐
│  Payment Service     │  ← Only this reacts first!
│                      │
│  Processes payment   │
└──────────────────────┘
    │
    ├─→ Payment Success ──→ Payment Confirmed Event
    │                           │
    │                           ▼
    │                   ┌──────────────────────┐
    │                   │  Other Services      │
    │                   │  (Inventory, etc.)   │
    │                   │  NOW can process!    │
    │                   └──────────────────────┘
    │
    └─→ Payment Failed ──→ Order Cancelled Event
                                │
                                ▼
                        ┌──────────────────────┐
                        │  Release resources   │
                        │  Notify user         │
                        └──────────────────────┘
```

---

## Scaling: How Kafka Handles Load

### ⚠️ Important: How Multiple Service Instances Work

**Question:** If I have 3 instances of Inventory Service running, how does Kafka ensure each event is consumed by only ONE instance?

**Answer:** Kafka uses **Consumer Groups** and **Partition Assignment** to ensure exactly one consumer per partition!

### Consumer Groups: The Key Concept

```
Consumer Group = Team of workers from the same service

Rule: Each partition can be consumed by ONLY ONE consumer in a group
```

### Visual: Multiple Service Instances

#### Scenario: 3 Instances of Inventory Service

```
┌─────────────────────────────────────────────────────────┐
│              KAFKA: order-created Topic                 │
│                                                         │
│  Partition 0: [O1] [O4] [O7] [O10] ...                │
│  Partition 1: [O2] [O5] [O8] [O11] ...                │
│  Partition 2: [O3] [O6] [O9] [O12] ...                │
└─────────────────────────────────────────────────────────┘
                    │
                    │ Consumer Group: inventory-service-group
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Instance 1 │ │  Instance 2 │ │  Instance 3 │
│  (Server 1) │ │  (Server 2) │ │  (Server 3) │
│             │ │             │ │             │
│  Consumer 1 │ │  Consumer 2 │ │  Consumer 3 │
└─────────────┘ └─────────────┘ └─────────────┘
        │               │               │
        │               │               │
        ▼               ▼               ▼
   Partition 0     Partition 1     Partition 2
   
✅ Each partition consumed by ONLY ONE instance!
✅ No duplicate processing!
```

### How Kafka Assigns Partitions

```
Step 1: All instances join the same Consumer Group
    │
    ▼
┌─────────────────────────────────────┐
│  Consumer Group: inventory-group    │
│                                     │
│  Instance 1 (Consumer 1)            │
│  Instance 2 (Consumer 2)            │
│  Instance 3 (Consumer 3)            │
└─────────────────────────────────────┘
    │
        ▼
Step 2: Kafka automatically assigns partitions
        │
    ▼
┌─────────────────────────────────────┐
│  Partition Assignment:              │
│                                     │
│  Consumer 1 → Partition 0           │
│  Consumer 2 → Partition 1           │
│  Consumer 3 → Partition 2           │
└─────────────────────────────────────┘
    │
        ▼
Step 3: Each consumer processes ONLY its assigned partition
```

### Detailed Example: 3 Instances, 3 Partitions

```
┌─────────────────────────────────────────────────────────┐
│                    KAFKA TOPIC                          │
│              Topic: order-created                       │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ Partition 0  │  │ Partition 1  │  │ Partition 2  │ │
│  │              │  │              │  │              │ │
│  │ [Order #1]   │  │ [Order #2]   │  │ [Order #3]   │ │
│  │ [Order #4]   │  │ [Order #5]   │  │ [Order #6]   │ │
│  │ [Order #7]   │  │ [Order #8]   │  │ [Order #9]   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
                    │
                    │ Consumer Group: inventory-service-group
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
┌─────────────────────────────────────────────────────────┐
│              INVENTORY SERVICE INSTANCES                │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  Instance 1  │  │  Instance 2  │  │  Instance 3  │ │
│  │  (Server 1)  │  │  (Server 2)  │  │  (Server 3)  │ │
│  │              │  │              │  │              │ │
│  │ Consumer 1   │  │ Consumer 2   │  │ Consumer 3   │ │
│  │              │  │              │  │              │ │
│  │ Reads from   │  │ Reads from   │  │ Reads from   │ │
│  │ Partition 0  │  │ Partition 1  │  │ Partition 2  │ │
│  │              │  │              │  │              │ │
│  │ Processes:   │  │ Processes:   │  │ Processes:   │ │
│  │ Order #1     │  │ Order #2     │  │ Order #3     │ │
│  │ Order #4     │  │ Order #5     │  │ Order #6     │ │
│  │ Order #7     │  │ Order #8     │  │ Order #9     │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘

✅ Order #1 processed ONLY by Instance 1
✅ Order #2 processed ONLY by Instance 2
✅ Order #3 processed ONLY by Instance 3
✅ NO duplicate processing!
```

### What Happens When You Add/Remove Instances?

#### Scenario 1: Add 4th Instance (More instances than partitions)

```
Before: 3 Instances, 3 Partitions
┌──────┐  ┌──────┐  ┌──────┐
│  I1  │  │  I2  │  │  I3  │
│  P0  │  │  P1  │  │  P2  │
└──────┘  └──────┘  └──────┘

After: 4 Instances, 3 Partitions
┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│  I1  │  │  I2  │  │  I3  │  │  I4  │
│  P0  │  │  P1  │  │  P2  │  │ IDLE │  ← Instance 4 is idle
└──────┘  └──────┘  └──────┘  └──────┘

⚠️  Instance 4 waits as backup (standby)
✅ If Instance 1, 2, or 3 fails, Instance 4 takes over
```

#### Scenario 2: Remove 1 Instance (Rebalancing)

```
Before: 3 Instances, 3 Partitions
┌──────┐  ┌──────┐  ┌──────┐
│  I1  │  │  I2  │  │  I3  │
│  P0  │  │  P1  │  │  P2  │
└──────┘  └──────┘  └──────┘
    │
    │ Instance 2 crashes!
    ▼
After: 2 Instances, 3 Partitions (Kafka Rebalances)
┌──────┐              ┌──────┐
│  I1  │              │  I3  │
│  P0  │              │  P2  │
│  P1  │  ← Takes over│      │
└──────┘              └──────┘

✅ Instance 1 now handles Partition 0 AND Partition 1
✅ Automatic rebalancing - no manual intervention!
```

### Consumer Group Configuration

```java
// All instances use the SAME group-id
@KafkaListener(
    topics = "order-created", 
    groupId = "inventory-service-group"  // ← Same for all instances!
)
public void handleOrder(OrderCreatedEvent event) {
    // This method runs on only ONE instance per event
    inventoryService.reserve(event.getItems());
}
```

**Key Point:** All instances of the same service MUST use the same `groupId`!

### Visual: How Kafka Ensures No Duplicates

```
Event Flow:

1. Order #123 arrives in Partition 0
        │
        ▼
2. Kafka checks: Who is assigned to Partition 0?
        │
        ▼
3. Only Consumer 1 (Instance 1) is assigned
        │
        ▼
4. Kafka delivers event ONLY to Consumer 1
        │
        ▼
5. Instance 1 processes the event
        │
        ▼
6. ✅ Event processed exactly ONCE

Instance 2 and Instance 3 never see this event!
```

### Scenario: Black Friday - 10,000 orders/second

#### Without Scaling (1 Consumer)

```
┌─────────────────────────────────────┐
│      KAFKA: order-created           │
│                                     │
│  P0: [O1][O2][O3]...[O1000]        │
│  P1: [O1][O2][O3]...[O1000]        │  ← All processed by
│  P2: [O1][O2][O3]...[O1000]        │    ONE consumer
│                                     │
│  ⚠️  Consumer 1 (overloaded!)      │
└─────────────────────────────────────┘
     │
     │ Can only process 1,000/sec
     ▼
😞 Slow! Queue building up!
```

#### With Scaling (3 Consumers)

```
┌─────────────────────────────────────┐
│      KAFKA: order-created           │
│                                     │
│  P0: [O1][O4][O7]...  ← Consumer 1 │
│  P1: [O2][O5][O8]...  ← Consumer 2 │  ← Each handles
│  P2: [O3][O6][O9]...  ← Consumer 3 │    one partition
│                                     │
│  ✅ All consumers working!          │
└─────────────────────────────────────┘
     │
     │ Can process 3,000/sec (3x faster!)
     ▼
😊 Fast! No queue!
```

### Key Rules

```
✅ Rule 1: One partition = One consumer (in same group)
✅ Rule 2: Same groupId = Same team (work together)
✅ Rule 3: Different groupId = Different teams (all get events)
✅ Rule 4: Kafka automatically assigns partitions
✅ Rule 5: If consumer dies, Kafka reassigns its partition
```

### Example: Multiple Services (Different Groups)

```
┌─────────────────────────────────────┐
│      KAFKA: order-created           │
│                                     │
│  P0: [O1] [O4] [O7]                │
│  P1: [O2] [O5] [O8]                │
│  P2: [O3] [O6] [O9]                │
└─────────────────────────────────────┘
        │
        ├─────────────────┬─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Group:       │  │ Group:       │  │ Group:       │
│ inventory    │  │ payment      │  │ email        │
│              │  │              │  │              │
│ Instance 1   │  │ Instance 1   │  │ Instance 1   │
│ Instance 2   │  │ Instance 2   │  │ Instance 2   │
│ Instance 3   │  │ Instance 3   │  │ Instance 3   │
│              │  │              │  │              │
│ Each gets    │  │ Each gets    │  │ Each gets    │
│ ONE partition│  │ ONE partition│  │ ONE partition│
└──────────────┘  └──────────────┘  └──────────────┘

✅ Each service group gets ALL events
✅ Within each group, events distributed across instances
✅ No duplicate processing within a group
```

---

## Code Examples

### 1. Publishing an Event (Producer)

```java
@Service
public class OrderService {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public Order createOrder(OrderRequest request) {
        // 1. Save order to database
        Order order = orderRepository.save(new Order(request));
        
        // 2. Create event
        OrderCreatedEvent event = new OrderCreatedEvent(
        order.getId(),
        order.getBuyerId(),
        order.getItems(),
            order.getTotalAmount()
        );
        
        // 3. Send to Kafka (non-blocking, fast!)
        kafkaTemplate.send("order-created", order.getId(), event);
        
        // 4. Return immediately (user doesn't wait)
        return order;
    }
}
```

**Visual Flow:**
```
createOrder()
    │
    ├─→ Save to DB
    │
    ├─→ Create Event
    │
    └─→ Send to Kafka ──→ Returns immediately ✅
```

### 2. Consuming an Event (Consumer)

```java
@Service
public class InventoryService {
    
    // All instances of Inventory Service use SAME groupId
    @KafkaListener(topics = "order-created", groupId = "inventory-group")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // This runs automatically when event arrives
        // BUT only on ONE instance (the one assigned to this partition)
        System.out.println("Processing order: " + event.getOrderId() + 
                          " on instance: " + getInstanceId());
        
        // Reserve inventory
        for (OrderItem item : event.getItems()) {
            inventoryRepository.reserve(item.getProductId(), item.getQuantity());
        }
    }
}
```

**Visual Flow:**
```
Kafka receives event in Partition 0
    │
    ▼
Kafka checks: Who is assigned to Partition 0?
    │
    ▼
Only Consumer 1 (Instance 1) is assigned
    │
    ▼
@KafkaListener on Instance 1 detects event
    │
    ▼
handleOrderCreated() runs on Instance 1 ONLY
    │
    ▼
Inventory reserved ✅

Instance 2 and Instance 3 never see this event!
```

### 3. Payment-First Pattern (Correct Implementation)

```java
// Step 1: Order Created - Only Payment Service reacts
@KafkaListener(topics = "order-created", groupId = "payment-group")
public void processPayment(OrderCreatedEvent event) {
    // Process payment
    PaymentResult result = paymentService.charge(
        event.getOrderId(), 
        event.getTotalAmount()
    );
    
    if (result.isSuccess()) {
        // Publish payment confirmed event
        PaymentConfirmedEvent confirmedEvent = new PaymentConfirmedEvent(
            event.getOrderId(),
            event.getBuyerId(),
            event.getItems(),
            result.getTransactionId()
        );
        kafkaTemplate.send("payment-confirmed", event.getOrderId(), confirmedEvent);
    } else {
        // Publish payment failed event
        PaymentFailedEvent failedEvent = new PaymentFailedEvent(
            event.getOrderId(),
            result.getFailureReason()
        );
        kafkaTemplate.send("payment-failed", event.getOrderId(), failedEvent);
    }
}

// Step 2: Payment Confirmed - Other services can now process
@KafkaListener(topics = "payment-confirmed", groupId = "inventory-group")
public void reserveInventory(PaymentConfirmedEvent event) {
    // Only reserve inventory AFTER payment confirmed
    inventoryService.reserve(event.getOrderId(), event.getItems());
}

@KafkaListener(topics = "payment-confirmed", groupId = "shipping-group")
public void prepareShipment(PaymentConfirmedEvent event) {
    // Only prepare shipment AFTER payment confirmed
    shippingService.prepareShipment(event.getOrderId(), event.getItems());
}

@KafkaListener(topics = "payment-confirmed", groupId = "email-group")
public void sendConfirmation(PaymentConfirmedEvent event) {
    // Send confirmation email AFTER payment confirmed
    emailService.sendOrderConfirmation(event.getBuyerId(), event.getOrderId());
}

// Step 3: Handle Payment Failure
@KafkaListener(topics = "payment-failed", groupId = "order-group")
public void handlePaymentFailure(PaymentFailedEvent event) {
    // Update order status to CANCELLED
    orderService.cancelOrder(event.getOrderId(), "Payment failed");
    
    // Notify user
    notificationService.notifyPaymentFailure(event.getOrderId());
}
```

**Visual Flow:**
```
Order Created Event
    │
    ▼
┌─────────────────┐
│ Payment Service │  ← Only this reacts first!
│                 │
│ Charge card     │
└─────────────────┘
    │
    ├─→ Success ──→ Payment Confirmed Event
    │                   │
    │                   ├─→ Inventory Service
    │                   ├─→ Shipping Service
    │                   ├─→ Email Service
    │                   └─→ Analytics Service
    │
    └─→ Failed ──→ Payment Failed Event
                        │
                        └─→ Order Service (Cancel order)
```

### 4. Order State Management

```java
@Service
public class OrderService {
    
    @KafkaListener(topics = "order-created", groupId = "order-group")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Update order status to PENDING
        Order order = orderRepository.findById(event.getOrderId());
        order.setStatus(OrderStatus.PENDING);
        orderRepository.save(order);
    }
    
    @KafkaListener(topics = "payment-confirmed", groupId = "order-group")
    public void handlePaymentConfirmed(PaymentConfirmedEvent event) {
        // Update order status to PAID
        Order order = orderRepository.findById(event.getOrderId());
        order.setStatus(OrderStatus.PAID);
        order.setPaymentTransactionId(event.getTransactionId());
        orderRepository.save(order);
    }
    
    @KafkaListener(topics = "payment-failed", groupId = "order-group")
    public void handlePaymentFailed(PaymentFailedEvent event) {
        // Update order status to CANCELLED
        Order order = orderRepository.findById(event.getOrderId());
        order.setStatus(OrderStatus.CANCELLED);
        order.setCancellationReason(event.getFailureReason());
        orderRepository.save(order);
    }
}
```

**State Flow Diagram:**
```
┌─────────┐
│ PENDING │  ← Order created, waiting for payment
└─────────┘
    │
    ├─→ Payment Success ──→ ┌─────────┐
    │                       │  PAID   │  ← Payment confirmed
    │                       └─────────┘
    │                           │
    │                           ├─→ Inventory reserved
    │                           ├─→ Shipment prepared
    │                           └─→ ┌──────────┐
    │                               │ SHIPPED  │
    │                               └──────────┘
    │
    └─→ Payment Failed ──→ ┌────────────┐
                           │ CANCELLED  │  ← Order cancelled
                           └────────────┘
```

---

## Payment Confirmation Pattern

### Why This Pattern?

**Problem:** We can't process orders without payment confirmation!

**Solution:** Use event chaining - wait for payment confirmation before processing.

### Event Chain Flow

```
Event 1: order-created
    │
    ▼
Payment Service processes payment
    │
    ├─→ Success ──→ Event 2: payment-confirmed
    │                   │
    │                   └─→ Triggers: Inventory, Shipping, Email
    │
    └─→ Failed ──→ Event 2: payment-failed
                        │
                        └─→ Triggers: Order cancellation
```

### Implementation Pattern

```java
// Pattern: Event Chaining with State Check
    
// 1. Order Created - Start payment
    @KafkaListener(topics = "order-created")
public void initiatePayment(OrderCreatedEvent event) {
    paymentService.process(event);
}

// 2. Payment Confirmed - Process order
@KafkaListener(topics = "payment-confirmed")
public void processOrder(PaymentConfirmedEvent event) {
    // Now safe to process - payment is confirmed!
    inventoryService.reserve(event.getOrderId(), event.getItems());
    shippingService.prepare(event.getOrderId());
}

// 3. Payment Failed - Cancel order
@KafkaListener(topics = "payment-failed")
public void cancelOrder(PaymentFailedEvent event) {
    orderService.cancel(event.getOrderId());
}
```

### Safety Check Pattern

```java
@KafkaListener(topics = "payment-confirmed", groupId = "inventory-group")
public void reserveInventory(PaymentConfirmedEvent event) {
    // Double-check: Verify order is actually paid
    Order order = orderRepository.findById(event.getOrderId());
    
    if (order.getStatus() != OrderStatus.PAID) {
        logger.warn("Order {} not paid, skipping inventory reservation", 
                   event.getOrderId());
        return; // Don't process if not paid
    }
    
    // Safe to reserve inventory
    inventoryService.reserve(event.getOrderId(), event.getItems());
}
```

---

## Dead Letter Queue (DLQ) - Handling Failed Messages

### What is Dead Letter Queue?

**Problem:** Some messages fail to process (database errors, invalid data, etc.)

**Solution:** Send failed messages to a separate topic (DLQ) for manual investigation.

### Visual Flow

```
Normal Flow:
Event arrives → Process → Success ✅

Failed Flow:
Event arrives → Process → Error ❌ → Send to DLQ → Manual investigation
```

### Implementation

```java
@Service
public class OrderEventConsumer {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @KafkaListener(topics = "order-created", groupId = "inventory-group")
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // Try to process
            inventoryService.reserve(event.getOrderId(), event.getItems());
            
        } catch (Exception e) {
            // Log error
            logger.error("Failed to process order: {}", event.getOrderId(), e);
            
            // Send to Dead Letter Queue
            DeadLetterEvent dlqEvent = new DeadLetterEvent(
                event.getOrderId(),
                event,
                e.getMessage(),
                LocalDateTime.now()
            );
            
            kafkaTemplate.send("order-created-dlq", event.getOrderId(), dlqEvent);
        }
    }
}
```

### DLQ Consumer for Manual Review

```java
@Service
public class DeadLetterQueueHandler {
    
    @KafkaListener(topics = "order-created-dlq", groupId = "dlq-handler-group")
    public void handleDeadLetter(DeadLetterEvent dlqEvent) {
        // Store in database for manual review
        dlqRepository.save(new DeadLetterRecord(
            dlqEvent.getOriginalEvent(),
            dlqEvent.getErrorMessage(),
            dlqEvent.getTimestamp()
        ));
        
        // Send alert to operations team
        alertService.sendAlert("Failed event in DLQ: " + dlqEvent.getOrderId());
    }
}
```

### Retry Pattern with DLQ

```java
@KafkaListener(topics = "order-created", groupId = "inventory-group")
public void handleOrderCreated(OrderCreatedEvent event) {
    int maxRetries = 3;
    int retryCount = 0;
    
    while (retryCount < maxRetries) {
        try {
            inventoryService.reserve(event.getOrderId(), event.getItems());
            return; // Success!
            
        } catch (Exception e) {
            retryCount++;
            
            if (retryCount >= maxRetries) {
                // Max retries reached, send to DLQ
                logger.error("Max retries reached for order: {}", event.getOrderId());
                sendToDLQ(event, e);
            } else {
                // Wait and retry
                Thread.sleep(1000 * retryCount); // Exponential backoff
                logger.warn("Retrying order: {} (attempt {})", event.getOrderId(), retryCount);
            }
        }
    }
}
```

### DLQ Structure

```
Topics:
├── order-created (main topic)
├── order-created-dlq (dead letter queue)
└── order-created-retry (retry topic - optional)

Flow:
order-created → Process fails → Retry 3 times → Still fails → order-created-dlq
```

### Monitoring DLQ

```java
@Service
public class DLQMonitor {
    
    @Scheduled(fixedRate = 60000) // Check every minute
    public void monitorDLQ() {
        long dlqCount = dlqRepository.countUnprocessed();
        
        if (dlqCount > 100) {
            alertService.sendAlert("DLQ has " + dlqCount + " unprocessed messages!");
        }
    }
}
```

---

## Message Memory and Retention

### How Kafka Stores Messages

```
Kafka stores messages on disk, not just in memory!

┌─────────────────────────────────────┐
│         KAFKA BROKER                │
│                                     │
│  Topic: order-created               │
│  ┌───────────────────────────────┐  │
│  │ Partition 0                   │  │
│  │                               │  │
│  │ [Message 1] ← Offset 0        │  │
│  │ [Message 2] ← Offset 1        │  │
│  │ [Message 3] ← Offset 2        │  │
│  │ ...                           │  │
│  │ [Message N] ← Offset N        │  │
│  │                               │  │
│  │ Stored on DISK (persistent)   │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Message Retention Policies

Kafka keeps messages based on two policies:

#### 1. Time-Based Retention

```yaml
# Keep messages for 7 days
retention.ms: 604800000  # 7 days in milliseconds
```

**Visual:**
```
Messages older than 7 days are automatically deleted

Day 1: [M1] [M2] [M3] ... [M1000]
Day 2: [M1] [M2] [M3] ... [M2000]
...
Day 7: [M1] [M2] [M3] ... [M7000]
Day 8: [M1] deleted (too old) [M2] [M3] ... [M8000]
```

#### 2. Size-Based Retention

```yaml
# Keep maximum 10GB of messages per partition
retention.bytes: 10737418240  # 10GB in bytes
```

**Visual:**
```
When partition reaches 10GB, oldest messages are deleted

[Old Messages] [New Messages]
     ↓              ↓
  Deleted      Kept (within limit)
```

### Configuration Example

```yaml
# application.yaml or server.properties
spring:
  kafka:
    consumer:
      # Consumer settings
      max-poll-records: 500  # Max messages per poll
      fetch-min-size: 1MB    # Wait for 1MB before returning
      fetch-max-wait: 500ms  # Max wait time

# Kafka server configuration (server.properties)
log.retention.hours=168        # 7 days
log.retention.bytes=10737418240 # 10GB per partition
log.segment.bytes=1073741824   # 1GB per segment
```

### Memory Usage

```
Kafka Memory Usage:

┌─────────────────────────────────────┐
│         KAFKA MEMORY                │
│                                     │
│  1. Page Cache (OS Level)           │
│     - Recently accessed messages    │
│     - Fast read access              │
│                                     │
│  2. Producer Buffer                 │
│     - Messages waiting to send      │
│     - batch.size: 16KB              │
│                                     │
│  3. Consumer Buffer                 │
│     - Messages fetched but not      │
│       processed yet                 │
│     - fetch.min.bytes: 1MB          │
│                                     │
│  4. Index Files                     │
│     - Offset indexes                │
│     - Time indexes                  │
└─────────────────────────────────────┘
```

### Best Practices for Memory

```java
// 1. Batch Processing (Reduces memory per message)
@KafkaListener(topics = "order-created", groupId = "inventory-group")
public void handleOrders(List<OrderCreatedEvent> events) {
    // Process multiple events at once
    inventoryService.batchReserve(events);
}

// 2. Limit Batch Size
@KafkaListener(
    topics = "order-created",
    groupId = "inventory-group",
    containerFactory = "batchKafkaListenerContainerFactory"
)
public void handleOrders(List<OrderCreatedEvent> events) {
    // Process in batches
}

// Configuration
@Bean
public ConsumerFactory<String, Object> consumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100); // Max 100 per poll
    props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024 * 1024); // 1MB
    return new DefaultKafkaConsumerFactory<>(props);
}
```

### Monitoring Memory Usage

```java
@Service
public class KafkaMemoryMonitor {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @Scheduled(fixedRate = 60000)
    public void monitorMemory() {
        // Monitor consumer lag
        long lag = getConsumerLag();
        meterRegistry.gauge("kafka.consumer.lag", lag);
        
        // Monitor partition size
        long partitionSize = getPartitionSize("order-created", 0);
        meterRegistry.gauge("kafka.partition.size", partitionSize);
        
        // Alert if lag is too high
        if (lag > 10000) {
            alertService.sendAlert("High consumer lag: " + lag);
        }
    }
}
```

---

## Monitoring Kafka

### Key Metrics to Monitor

#### 1. Consumer Lag

**What it is:** How far behind consumers are from producers

```
Producer writes: [M1] [M2] [M3] [M4] [M5] [M6] [M7] [M8]
                                    ↑
Consumer reads:  [M1] [M2] [M3] [M4]
                                    ↑
                              Lag = 4 messages
```

**Why it matters:** High lag = consumers can't keep up

**Monitoring:**
```java
@Service
public class ConsumerLagMonitor {
    
    @Scheduled(fixedRate = 30000) // Every 30 seconds
    public void checkConsumerLag() {
        Map<TopicPartition, Long> lag = kafkaConsumer.endOffsets(partitions);
        
        lag.forEach((partition, endOffset) -> {
            long currentOffset = getCurrentOffset(partition);
            long lagCount = endOffset - currentOffset;
            
            if (lagCount > 1000) {
                logger.warn("High lag on {}: {} messages", partition, lagCount);
                alertService.sendAlert("High consumer lag detected!");
            }
        });
    }
}
```

#### 2. Throughput (Messages per Second)

**Monitoring:**
```java
@KafkaListener(topics = "order-created")
public void handleOrder(OrderCreatedEvent event) {
    // Track throughput
    meterRegistry.counter("kafka.messages.processed", 
                         "topic", "order-created").increment();
    
    processOrder(event);
}
```

#### 3. Error Rate

**Monitoring:**
```java
@KafkaListener(topics = "order-created")
public void handleOrder(OrderCreatedEvent event) {
    try {
        processOrder(event);
        meterRegistry.counter("kafka.messages.success").increment();
    } catch (Exception e) {
        meterRegistry.counter("kafka.messages.error").increment();
        throw e;
    }
}
```

#### 4. Processing Time

**Monitoring:**
```java
@KafkaListener(topics = "order-created")
public void handleOrder(OrderCreatedEvent event) {
    Timer.Sample sample = Timer.start(meterRegistry);
    
    try {
        processOrder(event);
    } finally {
        sample.stop(meterRegistry.timer("kafka.processing.time"));
    }
}
```

### Monitoring Dashboard

```
┌─────────────────────────────────────────────────┐
│         KAFKA MONITORING DASHBOARD              │
├─────────────────────────────────────────────────┤
│                                                 │
│  Consumer Lag:       1,234 messages            │
│  Throughput:         500 msg/sec               │
│  Error Rate:         0.1%                      │
│  Avg Processing:     50ms                      │
│                                                 │
│  Topics:                                        │
│  ├─ order-created:   10,000 messages           │
│  ├─ payment-confirmed: 8,000 messages          │
│  └─ order-created-dlq: 5 messages (⚠️)        │
│                                                 │
│  Partitions:                                    │
│  ├─ P0: Lag 400, Size 2GB                      │
│  ├─ P1: Lag 350, Size 1.8GB                    │
│  └─ P2: Lag 484, Size 2.1GB                    │
└─────────────────────────────────────────────────┘
```

### Tools for Monitoring

#### 1. Kafka Manager / CMAK

```
Features:
- View topics and partitions
- Monitor consumer groups
- Check consumer lag
- View broker status
```

#### 2. Prometheus + Grafana

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['localhost:9092']
```

#### 3. JMX Metrics

```java
// Enable JMX in application.yaml
spring:
  kafka:
    consumer:
      properties:
        spring.jmx.enabled: true
```

### Alerting Rules

```java
@Service
public class KafkaAlerts {
    
    @Scheduled(fixedRate = 60000)
    public void checkAlerts() {
        // Alert 1: High Consumer Lag
        if (consumerLag > 10000) {
            alertService.sendAlert("CRITICAL: Consumer lag > 10,000");
        }
        
        // Alert 2: High Error Rate
        double errorRate = getErrorRate();
        if (errorRate > 5.0) {
            alertService.sendAlert("CRITICAL: Error rate > 5%");
        }
        
        // Alert 3: DLQ Growing
        long dlqSize = getDLQSize();
        if (dlqSize > 100) {
            alertService.sendAlert("WARNING: DLQ has " + dlqSize + " messages");
        }
        
        // Alert 4: Low Throughput
        double throughput = getThroughput();
        if (throughput < 100) {
            alertService.sendAlert("WARNING: Throughput < 100 msg/sec");
        }
    }
}
```

### Health Checks

```java
@Component
public class KafkaHealthIndicator implements HealthIndicator {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Override
    public Health health() {
        try {
            // Try to get metadata
            kafkaTemplate.getProducerFactory().createProducer().partitionsFor("order-created");
            
            return Health.up()
                .withDetail("kafka", "Connected")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("kafka", "Disconnected")
                .withException(e)
                .build();
        }
    }
}
```

---

## Best Practices

### 1. Always Check for Duplicates (Idempotency)

```java
@KafkaListener(topics = "order-created")
public void handleOrder(OrderCreatedEvent event) {
    // Check if already processed
    if (orderRepository.existsById(event.getOrderId())) {
        return; // Skip duplicate
    }
    
    // Process order
    processOrder(event);
}
```

**Why?** Kafka might send the same event twice!

### 2. Handle Errors Gracefully

```java
@KafkaListener(topics = "order-created")
public void handleOrder(OrderCreatedEvent event) {
    try {
        processOrder(event);
    } catch (Exception e) {
        // Log error and send to dead letter queue
        logger.error("Failed to process order", e);
        kafkaTemplate.send("order-created-dlq", event);
    }
}
```

**Visual:**
```
Event arrives
    │
    ▼
Try to process
    │
    ├─→ Success ✅ → Done
    │
    └─→ Error ❌ → Send to DLQ for investigation
```

### 3. Use Keys for Ordering

```java
// Same orderId always goes to same partition
kafkaTemplate.send("order-created", orderId, event);
//                           topic    key    value
```

**Why?** Maintains order for same order ID!

```
Order #123 events:
  - Created
  - Paid
  - Shipped
  
All go to same partition → Processed in order ✅
```

---

## Configuration

### application.yaml

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all  # Wait for confirmation
      
    consumer:
      group-id: my-service-group
      auto-offset-reset: earliest  # Start from beginning
```

### Simple Explanation

```
bootstrap-servers: Where is Kafka? (localhost:9092)
key-serializer: How to convert key to bytes
value-serializer: How to convert event to bytes
acks: all = Wait for confirmation (safer)
group-id: Which team am I in?
auto-offset-reset: Start from beginning if new
```

---

## Summary

### Key Takeaways

```
✅ Events = Things that happened
✅ Kafka = Message storage (like a mailbox)
✅ Producers = Services that send events
✅ Consumers = Services that react to events
✅ Topics = Categories of events
✅ Partitions = Allow parallel processing
✅ Consumer Groups = Teams of workers
```

### Benefits

```
🚀 Fast: Respond immediately, process later
🔌 Loose Coupling: Services don't know each other
📈 Scalable: Add more consumers = faster processing
🛡️  Fault Tolerant: If one service fails, others work
🔄 Replayable: Can replay events for debugging
```

### When to Use

```
✅ High volume (many events)
✅ Multiple services need to react
✅ Need fast response times
✅ Services should be independent
✅ Need audit trail / event history
✅ Need to chain events (e.g., payment → processing)
```

### Important: Event Ordering

```
⚠️  Remember: Some events must happen in order!

Order Created → Payment Confirmed → Order Processing

Use event chaining to ensure proper order!
```

---

## Quick Reference

### Publishing Event
```java
kafkaTemplate.send("topic-name", key, event);
```

### Consuming Event
```java
@KafkaListener(topics = "topic-name", groupId = "my-group")
public void handle(Event event) {
    // Process event
}
```

### Key Concepts
- **Topic** = Category (e.g., "order-created")
- **Partition** = Sub-category for parallel processing
- **Consumer Group** = Team of workers (same service instances)
- **Key** = Ensures same key goes to same partition
- **Group ID** = Must be same for all instances of same service

### Multiple Instances Rules

```
✅ Same groupId = Same team (work together, no duplicates)
✅ One partition = One consumer (in same group)
✅ Kafka automatically assigns partitions
✅ If instance dies, Kafka reassigns partition to another instance
✅ Different services = Different groupIds (all get events)
```

---

## Next Steps

1. ✅ Understand the basics (you're here!)
2. 📝 Implement producer in your service
3. 📝 Implement consumer in your service
4. 🧪 Test with local Kafka
5. 📊 Monitor and optimize

---

**Remember:** Event-driven architecture is like a newspaper - publishers don't wait for readers, they just publish and move on! 📰
