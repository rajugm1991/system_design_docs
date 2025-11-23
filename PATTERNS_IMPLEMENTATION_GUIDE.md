# Advanced Microservices Patterns - Complete Implementation Guide

This guide provides comprehensive implementations and examples for all advanced microservices patterns.

## Quick Navigation
- [1. Rate Limiting](#1-rate-limiting)
- [2. Resilience4j Patterns](#2-resilience4j-patterns)
- [3. Event-Driven Architecture](#3-event-driven-architecture)
- [4. Saga Pattern](#4-saga-pattern)
- [5. Distributed Locking](#5-distributed-locking)
- [6. Bulkhead Pattern](#6-bulkhead-pattern)
- [7. Redis Caching](#7-redis-caching)
- [8. Stream Processing](#8-stream-processing)
- [9. NoSQL Usage](#9-nosql-usage)
- [10. Service Mesh](#10-service-mesh)

---

## 1. Rate Limiting

### Current Implementation
- **Location:** `api-gateway/src/main/java/.../filter/RateLimitingFilter.java`
- **Type:** In-memory token bucket (Bucket4j)
- **Limits:** 60 requests/minute, 10 requests/second per IP

### Redis-Based Distributed Rate Limiting

**File:** `api-gateway/src/main/java/.../filter/RedisRateLimitingFilter.java`

**Features:**
- Distributed across multiple gateway instances
- Per-user rate limiting (premium: 120/min, free: 60/min)
- Per-endpoint rate limiting
- Redis-backed for consistency

**Interview Answer:**
"I implement distributed rate limiting using Redis with Bucket4j. Each request gets a key based on user ID or IP address, and tokens are stored in Redis. This ensures consistent rate limiting across multiple gateway instances. For premium users, I allow higher rate limits. The token bucket algorithm allows bursts while maintaining average rate limits."

**Key Points:**
- Uses Redis for distributed state
- Token bucket algorithm for smooth rate limiting
- Per-user and per-IP limits
- Configurable limits per user tier

---

## 2. Resilience4j Patterns

### Implementation Plan

We'll add Resilience4j to product-service for calling seller-service.

**Dependencies:**
```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.1.0</version>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-circuitbreaker</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-retry</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-timelimiter</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-bulkhead</artifactId>
</dependency>
```

### Circuit Breaker

**Purpose:** Stop calling a failing service to prevent cascading failures

**Configuration:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      sellerService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
```

**Usage:**
```java
@CircuitBreaker(name = "sellerService", fallbackMethod = "getProductsFallback")
public List<ProductDTO> getProductsFromSeller(Long sellerId) {
    // Call seller-service
}
```

### Retry

**Purpose:** Retry failed requests with exponential backoff

**Configuration:**
```yaml
resilience4j:
  retry:
    instances:
      sellerService:
        maxAttempts: 3
        waitDuration: 1000
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.net.SocketTimeoutException
```

### Timeout

**Purpose:** Fail fast for slow services

**Configuration:**
```yaml
resilience4j:
  timelimiter:
    instances:
      sellerService:
        timeoutDuration: 5s
```

### Bulkhead

**Purpose:** Isolate resources to prevent one failure from affecting others

**Configuration:**
```yaml
resilience4j:
  bulkhead:
    instances:
      sellerService:
        maxConcurrentCalls: 10
        maxWaitDuration: 0
```

**Interview Answer:**
"I use Resilience4j for resilience patterns. Circuit breaker stops calling failing services, retry with exponential backoff handles transient failures, timeout fails fast for slow services, and bulkhead isolates resources. I combine these patterns to make services resilient to failures."

---

## 3. Event-Driven Architecture

### Kafka Implementation

**Purpose:** Decouple services using events

**Events:**
- `OrderCreated` - When order is placed (topic: `order-created`)
- `InventoryReserved` - When inventory is reserved (topic: `inventory-reserved`)
- `PaymentProcessed` - When payment is processed (topic: `payment-processed`)
- `OrderCompleted` - When order is completed (topic: `order-completed`)
- `OrderCancelled` - When order is cancelled (topic: `order-cancelled`)

**Interview Answer:**
"I use event-driven architecture with Kafka for decoupling services. When an order is placed, I publish an OrderCreated event to the `order-created` topic. Multiple services subscribe to these topics through consumer groups: inventory service reserves inventory, payment service processes payment, and notification service sends confirmation. Kafka provides high throughput, fault tolerance, and the ability to replay events, making it ideal for event-driven microservices. Each topic is partitioned for scalability, and consumer groups allow multiple instances of a service to process events in parallel."

---

## 4. Saga Pattern - Complete Guide

### What is Saga Pattern?

**Problem:** In microservices, you can't use traditional ACID transactions across multiple services.

**Solution:** Saga pattern manages distributed transactions by breaking them into a series of local transactions with compensation logic.

```
Traditional Transaction (Not possible in microservices):
┌─────────────────────────────────────┐
│  BEGIN TRANSACTION                  │
│    Service A: Update                │
│    Service B: Update                │
│    Service C: Update                │
│  COMMIT (all or nothing)            │
└─────────────────────────────────────┘

Saga Pattern (Works in microservices):
┌─────────────────────────────────────┐
│  Step 1: Service A (local tx) ✅    │
│  Step 2: Service B (local tx) ✅    │
│  Step 3: Service C (local tx) ❌    │
│  Compensate: Service B (undo)       │
│  Compensate: Service A (undo)       │
└─────────────────────────────────────┘
```

### Two Types of Saga

#### 1. Orchestration-Based Saga (What We Use)

```
Central Orchestrator coordinates all steps

┌─────────────────────────────────────┐
│     Saga Orchestrator               │
│                                     │
│  1. Call Service A                  │
│  2. Call Service B                  │
│  3. Call Service C                  │
│                                     │
│  If failure:                        │
│    Compensate in reverse order      │
└─────────────────────────────────────┘
        │         │         │
        ▼         ▼         ▼
   Service A  Service B  Service C
```

**Benefits:**
- Centralized control
- Easy to understand flow
- Better error handling
- Easier to test

#### 2. Choreography-Based Saga

```
Each service knows what to do next

Service A → Event → Service B → Event → Service C
    │                                    │
    └─── If failure: Compensate ←───────┘
```

**Benefits:**
- Decoupled services
- No single point of failure
- More scalable

**We use Orchestration** because it's easier to manage and debug.

---

### When to Use Saga Pattern?

#### ✅ Use Saga When:

1. **Distributed Transactions Required**
   - Operations span multiple services
   - Need to maintain consistency across services

2. **Long-Running Transactions**
   - Operations take time (seconds/minutes)
   - Can't hold locks for long periods

3. **Eventual Consistency Acceptable**
   - Don't need immediate consistency
   - Can tolerate temporary inconsistency

4. **Compensatable Operations**
   - Each step can be undone
   - Compensation logic is feasible

#### ❌ Don't Use Saga When:

1. **Strong Consistency Required**
   - Need immediate consistency
   - Use synchronous calls or 2PC

2. **Non-Compensatable Operations**
   - Can't undo operations (e.g., sending email)
   - Use TCC (Try-Confirm-Cancel) pattern

3. **Simple Operations**
   - Single service operation
   - Use regular transactions

---

### How Saga Pattern Works

#### Complete Order Processing Saga

```
┌─────────────────────────────────────────────────────────┐
│              ORDER PROCESSING SAGA                       │
└─────────────────────────────────────────────────────────┘

Step 1: Create Order
    │
    ├─→ Order Service: Create order (Status: PENDING)
    │
    ▼
Step 2: Reserve Inventory
    │
    ├─→ Inventory Service: Reserve stock
    │
    ▼
Step 3: Process Payment
    │
    ├─→ Payment Service: Charge card
    │
    ▼
Step 4: Update Inventory
    │
    ├─→ Inventory Service: Deduct stock
    │
    ▼
Step 5: Send Notification
    │
    ├─→ Notification Service: Send email
    │
    ▼
✅ Saga Completed Successfully
```

#### Visual Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│              SAGA ORCHESTRATOR                          │
└─────────────────────────────────────────────────────────┘
                    │
                    │ Start Saga
                    ▼
        ┌───────────────────────┐
        │  Step 1: Create Order │
        └───────────────────────┘
                    │
                    ├─→ Success ✅
                    │
                    ▼
        ┌──────────────────────────┐
        │  Step 2: Reserve Inventory│
        └──────────────────────────┘
                    │
                    ├─→ Success ✅
                    │
                    ▼
        ┌──────────────────────────┐
        │  Step 3: Process Payment │
        └──────────────────────────┘
                    │
                    ├─→ Success ✅
                    │
                    ▼
        ┌──────────────────────────┐
        │  Step 4: Update Inventory│
        └──────────────────────────┘
                    │
                    ├─→ Success ✅
                    │
                    ▼
        ┌──────────────────────────┐
        │  Step 5: Send Notification│
        └──────────────────────────┘
                    │
                    ├─→ Success ✅
                    │
                    ▼
            ✅ Saga Completed
```

---

### Scenario 1: Successful Saga Execution

```
Time →
│
├─ 0ms:   User clicks "Checkout"
│
├─ 10ms:  Saga Orchestrator starts
│
├─ 20ms:  Step 1: Create Order ✅
│         Order created: ORD-123
│
├─ 50ms:  Step 2: Reserve Inventory ✅
│         Inventory reserved: SKU-1, Qty: 2
│
├─ 200ms: Step 3: Process Payment ✅
│         Payment processed: $100
│
├─ 250ms: Step 4: Update Inventory ✅
│         Inventory deducted: SKU-1, Qty: 2
│
├─ 300ms: Step 5: Send Notification ✅
│         Email sent to user
│
└─ 310ms: ✅ Saga Completed Successfully
```

**Result:** Order processed, all steps completed.

---

### Scenario 2: Saga Failure with Compensation

```
Time →
│
├─ 0ms:   User clicks "Checkout"
│
├─ 10ms:  Saga Orchestrator starts
│
├─ 20ms:  Step 1: Create Order ✅
│         Order created: ORD-123
│
├─ 50ms:  Step 2: Reserve Inventory ✅
│         Inventory reserved: SKU-1, Qty: 2
│
├─ 200ms: Step 3: Process Payment ❌
│         Payment failed: Insufficient funds
│
│         ⚠️  Saga Failed! Starting Compensation...
│
├─ 210ms: Compensate Step 2: Release Inventory ✅
│         Inventory released: SKU-1, Qty: 2
│
├─ 250ms: Compensate Step 1: Cancel Order ✅
│         Order cancelled: ORD-123
│
└─ 260ms: ❌ Saga Failed - All steps compensated
```

**Result:** System returned to initial state, no partial updates.

---

### Detailed Compensation Flow

#### Visual: Compensation in Action

```
┌─────────────────────────────────────────────────────────┐
│              SAGA EXECUTION FLOW                         │
└─────────────────────────────────────────────────────────┘

Forward Execution:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Step 1   │───→│ Step 2   │───→│ Step 3   │───→│ Step 4   │
│ Create   │    │ Reserve  │    │ Payment  │    │ Update   │
│ Order    │    │ Inventory│    │ Process  │    │ Inventory│
└──────────┘    └──────────┘    └──────────┘    └──────────┘
    ✅              ✅              ❌ FAILED!

When Step 3 Fails:

Completed Steps:
├─ Step 1: Create Order ✅
└─ Step 2: Reserve Inventory ✅

Compensation (Reverse Order):
│
┌─────────────────────────────────────────────────────────┐
│              COMPENSATION FLOW                           │
└─────────────────────────────────────────────────────────┘

┌──────────┐    ┌──────────┐
│Compensate│←───│Compensate│
│Step 2    │    │Step 1    │
│Release   │    │Cancel    │
│Inventory │    │Order     │
└──────────┘    └──────────┘
    ✅              ✅

Result: System back to initial state ✅
```

#### Step-by-Step Compensation

```
When Payment Step Fails:

State Before Saga:
┌─────────────────────┐
│ Order: None         │
│ Inventory: Available│
│ Payment: None       │
└─────────────────────┘

After Step 1 (Create Order):
┌─────────────────────┐
│ Order: ORD-123      │  ← Created
│ Inventory: Available│
│ Payment: None       │
└─────────────────────┘

After Step 2 (Reserve Inventory):
┌─────────────────────┐
│ Order: ORD-123      │
│ Inventory: Reserved │  ← Reserved
│ Payment: None       │
└─────────────────────┘

Step 3 (Payment) Fails ❌:
┌─────────────────────┐
│ Order: ORD-123      │
│ Inventory: Reserved │  ← Still reserved!
│ Payment: Failed     │  ← Failed
└─────────────────────┘

⚠️  INCONSISTENT STATE! Need compensation.

Compensate Step 2 (Release Inventory):
┌─────────────────────┐
│ Order: ORD-123      │
│ Inventory: Released │  ← Released ✅
│ Payment: Failed     │
└─────────────────────┘

Compensate Step 1 (Cancel Order):
┌─────────────────────┐
│ Order: CANCELLED    │  ← Cancelled ✅
│ Inventory: Available│  ← Available ✅
│ Payment: None       │
└─────────────────────┘

✅ CONSISTENT STATE RESTORED!
```

#### Compensation Flow Diagram

```
Forward Flow:
Step 1 → Step 2 → Step 3 ❌

Compensation Flow (Reverse):
Step 3 ❌ → Compensate Step 2 → Compensate Step 1

Final State: Same as before saga started

Visual Timeline:
─────────────────────────────────────────────────
Time →
│
├─ 0ms:   Start Saga
│
├─ 10ms:  Step 1: Create Order ✅
│
├─ 30ms:  Step 2: Reserve Inventory ✅
│
├─ 100ms: Step 3: Process Payment ❌
│
│         ⚠️  FAILURE DETECTED
│
├─ 110ms: Compensate Step 2: Release Inventory ✅
│
├─ 140ms: Compensate Step 1: Cancel Order ✅
│
└─ 150ms: ✅ System restored to initial state
```

---

### Implementation Details

#### Saga Step Interface

```java
public interface SagaStep {
    /**
     * Execute the saga step
     * Must be idempotent (can be retried safely)
     */
    void execute(SagaContext context) throws Exception;
    
    /**
     * Compensate the saga step (undo the operation)
     * Must be idempotent
     */
    void compensate(SagaContext context) throws Exception;
}
```

#### Example: Reserve Inventory Step

```java
@Component
public class ReserveInventoryStep implements SagaStep {
    
    @Override
    public void execute(SagaContext context) throws Exception {
        // Reserve inventory
        context.getRequest().getCartItems().forEach(item -> {
            inventoryService.reserve(item.getSkuId(), item.getQuantity());
        });
        context.setInventoryReserved(true);
    }
    
    @Override
    public void compensate(SagaContext context) throws Exception {
        // Release reserved inventory
        if (context.isInventoryReserved()) {
            context.getRequest().getCartItems().forEach(item -> {
                inventoryService.release(item.getSkuId(), item.getQuantity());
            });
            context.setInventoryReserved(false);
        }
    }
}
```

#### Saga Orchestrator

```java
@Service
public class OrderSagaOrchestrator {
    
    public SagaResult processOrder(Long buyerId, CheckoutRequest request) {
        SagaContext context = new SagaContext(buyerId, request);
        List<SagaStep> completedSteps = new ArrayList<>();
        
        try {
            // Execute steps in order
            completedSteps.add(createOrderStep);
            createOrderStep.execute(context);
            
            completedSteps.add(reserveInventoryStep);
            reserveInventoryStep.execute(context);
            
            completedSteps.add(processPaymentStep);
            processPaymentStep.execute(context);
            
            completedSteps.add(updateInventoryStep);
            updateInventoryStep.execute(context);
            
            completedSteps.add(sendNotificationStep);
            sendNotificationStep.execute(context);
            
            return new SagaResult(true, "Success", context);
            
        } catch (Exception e) {
            // Compensate in reverse order
            compensate(completedSteps, context);
            return new SagaResult(false, e.getMessage(), context);
        }
    }
    
    private void compensate(List<SagaStep> completedSteps, SagaContext context) {
        // Compensate in reverse order
        for (int i = completedSteps.size() - 1; i >= 0; i--) {
            SagaStep step = completedSteps.get(i);
            try {
                step.compensate(context);
            } catch (Exception e) {
                logger.error("Compensation failed", e);
                // Continue with other compensations
            }
        }
    }
}
```

---

### Best Practices

#### 1. Make Steps Idempotent

```java
@Override
public void execute(SagaContext context) throws Exception {
    // Check if already executed
    if (context.isInventoryReserved()) {
        return; // Already done, skip
    }
    
    // Execute operation
    inventoryService.reserve(context.getOrderId(), context.getItems());
    context.setInventoryReserved(true);
}
```

**Why?** Allows safe retries if saga fails mid-way.

#### 2. Store Saga State

```java
@Entity
public class SagaState {
    private String sagaId;
    private String currentStep;
    private SagaStatus status; // PENDING, IN_PROGRESS, COMPLETED, FAILED, COMPENSATING
    private List<String> completedSteps;
    private String errorMessage;
}
```

**Why?** Can resume saga if orchestrator crashes.

#### 3. Use Distributed Locks

```java
@Override
public void execute(SagaContext context) throws Exception {
    String lockKey = "saga:" + context.getSagaId();
    
    lockService.executeWithLock(lockKey, Duration.ofMinutes(5), () -> {
        // Execute step
        inventoryService.reserve(context.getOrderId(), context.getItems());
        return null;
    });
}
```

**Why?** Prevents concurrent execution of same saga.

#### 4. Log All Steps

```java
@Override
public void execute(SagaContext context) throws Exception {
    logger.info("Executing step: ReserveInventory, sagaId: {}", context.getSagaId());
    
    try {
        inventoryService.reserve(context.getOrderId(), context.getItems());
        logger.info("Step completed: ReserveInventory, sagaId: {}", context.getSagaId());
    } catch (Exception e) {
        logger.error("Step failed: ReserveInventory, sagaId: {}", context.getSagaId(), e);
        throw e;
    }
}
```

**Why?** Easier debugging and audit trail.

#### 5. Handle Compensation Failures

```java
private void compensate(List<SagaStep> completedSteps, SagaContext context) {
    for (int i = completedSteps.size() - 1; i >= 0; i--) {
        SagaStep step = completedSteps.get(i);
        try {
            step.compensate(context);
        } catch (Exception e) {
            // Log but continue
            logger.error("Compensation failed for step: {}", step.getClass().getSimpleName(), e);
            
            // Store for manual intervention
            compensationFailureService.recordFailure(context.getSagaId(), step, e);
        }
    }
}
```

**Why?** Some compensations might fail, but we should try all.

#### 6. Set Timeouts

```java
@SagaStep(timeout = 30, unit = TimeUnit.SECONDS)
public void execute(SagaContext context) throws Exception {
    // Step must complete within 30 seconds
    paymentService.processPayment(context.getOrderId(), context.getAmount());
}
```

**Why?** Prevents saga from hanging indefinitely.

#### 7. Use Events for Async Steps

```java
// Instead of waiting synchronously
public void execute(SagaContext context) throws Exception {
    // Publish event
    eventPublisher.publishInventoryReservationRequested(context);
    
    // Don't wait, saga continues when event is processed
}
```

**Why?** Better for long-running operations.

---

### Saga vs Other Patterns

#### Saga vs 2PC (Two-Phase Commit)

```
2PC:
┌─────────────────────────────────────┐
│  Phase 1: Prepare (all services)    │
│  Phase 2: Commit (all services)     │
│                                     │
│  Problem: Blocks resources          │
│  Problem: Single point of failure   │
└─────────────────────────────────────┘

Saga:
┌─────────────────────────────────────┐
│  Step 1: Service A (local tx)       │
│  Step 2: Service B (local tx)       │
│  Step 3: Service C (local tx)       │
│                                     │
│  Benefit: No blocking               │
│  Benefit: Fault tolerant            │
└─────────────────────────────────────┘
```

#### Saga vs Event Sourcing

```
Saga: Manages distributed transactions
Event Sourcing: Stores state as events

Can be used together:
- Saga orchestrates transactions
- Events store state changes
```

---

### Interview Questions & Answers

#### Q1: What is Saga Pattern?

**Answer:**
"Saga pattern is used to manage distributed transactions in microservices. Instead of using traditional ACID transactions (which don't work across services), Saga breaks a transaction into a series of local transactions. Each step has a compensating transaction that can undo its effects. If any step fails, we execute compensating transactions in reverse order to return the system to a consistent state. This provides eventual consistency without distributed locks."

#### Q2: What are the two types of Saga?

**Answer:**
"There are two types:

1. **Orchestration-Based Saga:** A central orchestrator coordinates all steps. It calls each service in sequence and handles compensation if any step fails. This is what we use because it's easier to manage and debug.

2. **Choreography-Based Saga:** Each service knows what to do next. Services communicate through events. More decoupled but harder to track and debug.

We use orchestration because it provides better control and visibility."

#### Q3: How do you handle compensation failures?

**Answer:**
"Compensation failures are tricky. When compensating, if one compensation fails, we:
1. Log the failure
2. Continue compensating other steps
3. Store the failure for manual intervention
4. Send alerts to operations team

We also make compensations idempotent so they can be retried. For critical compensations, we might implement a retry mechanism with exponential backoff."

#### Q4: How do you ensure idempotency in Saga steps?

**Answer:**
"Each step checks if it's already been executed before performing the operation. We store the saga state and check flags like `isInventoryReserved()` before reserving inventory. If already done, we skip. This allows safe retries if the saga fails mid-way and needs to be restarted."

#### Q5: What happens if the orchestrator crashes during saga execution?

**Answer:**
"We store saga state in a database with the current step and status. When the orchestrator restarts, it can:
1. Query for incomplete sagas
2. Resume from the last completed step
3. Or restart from the beginning if steps are idempotent

We also use distributed locks to prevent concurrent execution of the same saga."

#### Q6: How do you test Saga pattern?

**Answer:**
"We test in multiple ways:
1. **Unit Tests:** Mock each step and test orchestrator logic
2. **Integration Tests:** Test with real services but mock external dependencies
3. **Failure Scenarios:** Intentionally fail steps and verify compensation
4. **Idempotency Tests:** Execute same step multiple times and verify no side effects

We also use test containers to test with real Kafka and databases."

#### Q7: When would you use Saga vs synchronous calls?

**Answer:**
"Use Saga when:
- Operations span multiple services
- Long-running transactions (seconds/minutes)
- Eventual consistency is acceptable
- Operations are compensatable

Use synchronous calls when:
- Need immediate consistency
- Simple operations (single service)
- Fast operations (milliseconds)
- Strong consistency required"

#### Q8: How do you monitor Saga execution?

**Answer:**
"We monitor:
1. **Saga Success Rate:** Percentage of successful sagas
2. **Average Execution Time:** How long sagas take
3. **Compensation Rate:** How often compensation is needed
4. **Step Failure Rate:** Which steps fail most often
5. **Stuck Sagas:** Sagas that haven't completed in expected time

We use metrics like Prometheus and logs for debugging. We also have dashboards showing saga health."

#### Q9: What are the challenges with Saga pattern?

**Answer:**
"Main challenges:
1. **Compensation Logic:** Not all operations can be easily undone
2. **Eventual Consistency:** System might be temporarily inconsistent
3. **Complexity:** More complex than simple transactions
4. **Testing:** Harder to test all failure scenarios
5. **Debugging:** Harder to debug distributed transactions

We mitigate these by:
- Careful design of compensation logic
- Comprehensive logging
- Good monitoring
- Idempotent steps
- State management"

#### Q10: How does Saga ensure data consistency?

**Answer:**
"Saga ensures eventual consistency, not immediate consistency. During saga execution, the system might be temporarily inconsistent (e.g., order created but payment not processed). However, if the saga completes successfully, all services are consistent. If it fails, compensation ensures we return to a consistent state. This is acceptable for most business scenarios where immediate consistency isn't required."

---

### Saga State Transitions

```
Saga State Machine:

┌──────────┐
│ PENDING  │  ← Saga created, not started
└──────────┘
    │
    │ Start
    ▼
┌──────────────┐
│ IN_PROGRESS  │  ← Executing steps
└──────────────┘
    │
    ├─→ All steps succeed
    │       │
    │       ▼
    │   ┌──────────┐
    │   │COMPLETED │  ← Saga successful
    │   └──────────┘
    │
    └─→ Step fails
            │
            ▼
        ┌──────────────┐
        │ COMPENSATING │  ← Undoing steps
        └──────────────┘
            │
            ├─→ All compensations succeed
            │       │
            │       ▼
            │   ┌────────┐
            │   │ FAILED │  ← Saga failed, compensated
            │   └────────┘
            │
            └─→ Compensation fails
                    │
                    ▼
                ┌──────────────┐
                │ COMPENSATION │  ← Manual intervention needed
                │   FAILED     │
                └──────────────┘
```

### Real-World Example: E-Commerce Order

```
Complete Order Saga with All Scenarios:

Scenario A: Success
─────────────────────────────────────────
Step 1: Create Order ✅
Step 2: Reserve Inventory ✅
Step 3: Process Payment ✅
Step 4: Update Inventory ✅
Step 5: Send Notification ✅
Result: Order completed successfully

Scenario B: Payment Fails
─────────────────────────────────────────
Step 1: Create Order ✅
Step 2: Reserve Inventory ✅
Step 3: Process Payment ❌ (Insufficient funds)
Compensate Step 2: Release Inventory ✅
Compensate Step 1: Cancel Order ✅
Result: Order cancelled, inventory released

Scenario C: Inventory Fails
─────────────────────────────────────────
Step 1: Create Order ✅
Step 2: Reserve Inventory ❌ (Out of stock)
Compensate Step 1: Cancel Order ✅
Result: Order cancelled (no inventory to release)

Scenario D: Notification Fails (Non-critical)
─────────────────────────────────────────
Step 1: Create Order ✅
Step 2: Reserve Inventory ✅
Step 3: Process Payment ✅
Step 4: Update Inventory ✅
Step 5: Send Notification ❌ (Email service down)
Result: Order completed, but notification failed
Note: Notification failure doesn't require compensation
      (order is already complete)
```

### Saga Pattern in Our System

```
Our Implementation:

┌─────────────────────────────────────────────────────────┐
│         OrderSagaOrchestrator (product-service)         │
└─────────────────────────────────────────────────────────┘
                    │
                    │ Coordinates all steps
                    ▼
        ┌───────────────────────────────────┐
        │  Steps (in order):                │
        │                                   │
        │  1. CreateOrderStep               │
        │  2. ReserveInventoryStep          │
        │  3. ProcessPaymentStep            │
        │  4. UpdateInventoryStep           │
        │  5. SendNotificationStep          │
        └───────────────────────────────────┘
                    │
                    │ Each step can call external services
                    ▼
        ┌───────────────────────────────────┐
        │  External Services:               │
        │                                   │
        │  - Order Service                  │
        │  - Inventory Service              │
        │  - Payment Service                │
        │  - Notification Service           │
        └───────────────────────────────────┘
```

### Summary

**Saga Pattern:**
- ✅ Manages distributed transactions
- ✅ Uses local transactions + compensation
- ✅ Provides eventual consistency
- ✅ No distributed locks needed
- ✅ Fault tolerant

**Key Points:**
- Make steps idempotent
- Store saga state
- Handle compensation failures
- Monitor saga execution
- Use distributed locks for concurrency

**When to Use:**
- ✅ Distributed transactions across services
- ✅ Long-running operations
- ✅ Eventual consistency acceptable
- ✅ Operations are compensatable

**When NOT to Use:**
- ❌ Need immediate consistency
- ❌ Non-compensatable operations
- ❌ Simple single-service operations

---

## 5. Distributed Locking

### Redis Distributed Lock

**Purpose:** Prevent concurrent modifications

**Use Cases:**
- Inventory updates
- Order processing
- Critical resource access

**Implementation:**
```java
// Acquire lock
String lockKey = "lock:inventory:" + skuId;
boolean acquired = redisTemplate.opsForValue()
    .setIfAbsent(lockKey, "locked", Duration.ofSeconds(30));

if (acquired) {
    try {
        // Update inventory
    } finally {
        // Release lock
        redisTemplate.delete(lockKey);
    }
}
```

**Interview Answer:**
"I use Redis distributed locks for critical operations like inventory updates. I use SETNX with expiration to acquire locks, and always release them in finally blocks. For long operations, I implement lock renewal. This prevents race conditions in distributed systems."

---

## 6. Bulkhead Pattern

### Thread Pool Isolation

**Purpose:** Isolate resources to prevent cascading failures

**Implementation:**
```java
@Configuration
public class BulkheadConfig {
    @Bean
    public ThreadPoolExecutor criticalOperationsExecutor() {
        return new ThreadPoolExecutor(10, 10, 0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<>(100));
    }
    
    @Bean
    public ThreadPoolExecutor normalOperationsExecutor() {
        return new ThreadPoolExecutor(20, 20, 0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<>(200));
    }
}
```

**Interview Answer:**
"I use bulkhead pattern with thread pool isolation. Critical operations have dedicated thread pools, so a slow operation doesn't block critical ones. I also isolate connection pools per service, so one failing service doesn't exhaust all database connections."

---

## 7. Redis Caching

### Cache Implementation

**Purpose:** Improve performance by caching frequently accessed data

**Cache Strategy:**
- Cache products for 5 minutes
- Cache inventory for 1 minute
- Cache user data for 10 minutes
- Invalidate cache on updates

**Interview Answer:**
"I use Redis for caching to improve performance. I cache product data, inventory, and user data with appropriate TTLs. I use cache-aside pattern where I check cache first, then database, then update cache. On updates, I invalidate cache to ensure consistency."

---

## 8. Stream Processing

### Kafka Streams

**Purpose:** Process events in real-time

**Use Cases:**
- Real-time analytics
- Fraud detection
- Recommendation engine
- Event sourcing

**Interview Answer:**
"I use Kafka Streams for real-time event processing. I process order events to calculate real-time metrics, detect fraud patterns, and update recommendations. Kafka provides high throughput and fault tolerance for stream processing."

---

## 9. NoSQL Usage

### When to Use

**Redis:** Caching, session storage, rate limiting
**MongoDB:** Product catalog, user profiles, flexible schema
**Cassandra:** Time-series data, analytics, high write throughput
**Neo4j:** Recommendations, social networks, graph queries

**Interview Answer:**
"I use NoSQL based on data access patterns. Redis for caching because it's extremely fast, MongoDB for product catalogs because of flexible schema, Cassandra for analytics because of high write throughput, and Neo4j for recommendations because of efficient graph queries."

---

## 10. Service Mesh

### Istio Configuration

**Purpose:** Handle service-to-service communication

**Features:**
- Traffic management
- Security (mTLS)
- Observability
- Policy enforcement

**Interview Answer:**
"I use service mesh like Istio for service-to-service communication. It handles traffic management, security with mTLS, observability with distributed tracing, and policy enforcement. The mesh intercepts all service communication, so I get these features without modifying application code."

---

## Next Steps

1. Implement Resilience4j in product-service
2. ✅ Kafka configured for event-driven architecture
3. Implement Saga pattern for order processing
4. Add Redis distributed locking
5. Implement bulkhead pattern
6. Add Redis caching
7. Create Kafka stream processing setup
8. Add MongoDB for product catalog
9. Create comprehensive examples

Let's start implementing!

