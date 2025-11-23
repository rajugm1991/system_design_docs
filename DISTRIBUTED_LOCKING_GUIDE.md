# Distributed Locking - Complete Implementation Guide

## Table of Contents
1. [What is Distributed Locking?](#what-is-distributed-locking)
2. [How Distributed Locking is Used in Our Codebase](#how-distributed-locking-is-used-in-our-codebase)
3. [Code Structure and Implementation](#code-structure-and-implementation)
4. [When to Use Distributed Locking](#when-to-use-distributed-locking)
5. [How Distributed Locking Works](#how-distributed-locking-works)
6. [Best Practices](#best-practices)
7. [Advanced Interview Questions](#advanced-interview-questions)

---

## What is Distributed Locking?

### Problem: Race Conditions in Distributed Systems

**Problem:** In a distributed system with multiple service instances, multiple requests can try to modify the same resource simultaneously, causing race conditions.

```
Without Distributed Locking:

Instance 1: Read inventory (100 units)
Instance 2: Read inventory (100 units)  ← Both read same value!
Instance 1: Update inventory (100 - 10 = 90)
Instance 2: Update inventory (100 - 5 = 95)  ← Lost update!

Result: Inventory should be 85, but it's 95 ❌
```

**Solution:** Distributed locking ensures only one process can access a resource at a time.

```
With Distributed Locking:

Instance 1: Acquire lock → Read (100) → Update (90) → Release lock ✅
Instance 2: Wait for lock → Acquire lock → Read (90) → Update (85) → Release lock ✅

Result: Inventory is correctly 85 ✅
```

### What is a Distributed Lock?

A **distributed lock** is a mechanism that allows only one process/thread across multiple machines to access a shared resource at a time.

```
┌─────────────────────────────────────────────────────────┐
│         DISTRIBUTED LOCK CONCEPT                        │
└─────────────────────────────────────────────────────────┘

Service Instance 1          Service Instance 2          Service Instance 3
     │                            │                            │
     │  Try to acquire lock       │  Try to acquire lock       │  Try to acquire lock
     │         │                   │         │                   │         │
     │         ▼                   │         ▼                   │         ▼
     └─────────┼───────────────────┴─────────┼───────────────────┴─────────┼
               │                             │                             │
               └─────────────────────────────┴─────────────────────────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │   Redis (Lock Store) │
                              │                      │
                              │  lock:inventory:123  │
                              │  Value: uuid-abc     │
                              │  TTL: 30 seconds     │
                              └──────────────────────┘
                                         │
                                         ▼
                              Only ONE instance gets lock
                              Others wait or fail
```

---

## How Distributed Locking is Used in Our Codebase

### Overview

In our e-commerce system, distributed locking is used for:

1. **Inventory Updates** - Prevent overselling when multiple orders process simultaneously
2. **Saga Steps** - Ensure atomic operations in distributed transactions
3. **Critical Sections** - Protect shared resources across service instances

### Real-World Scenarios

#### Scenario 1: Inventory Reservation (Saga Pattern)

```
When processing an order in Saga:

Step 2: Reserve Inventory
    │
    ├─→ Multiple service instances might try to reserve same SKU
    │
    ├─→ Without lock: Could oversell (sell more than available)
    │
    └─→ With lock: Only one instance can reserve at a time ✅
```

**Code Location:** `saga/step/ReserveInventoryStep.java`

#### Scenario 2: Inventory Update

```
When updating inventory:

Multiple requests for same SKU arrive simultaneously
    │
    ├─→ Without lock: Race condition, lost updates
    │
    └─→ With lock: Sequential updates, data consistency ✅
```

**Code Location:** `service/LockedInventoryService.java`

---

## Code Structure and Implementation

### File Structure

```
product-service/
└── src/main/java/com/ekart/product_service/
    ├── lock/
    │   └── DistributedLockService.java      ← Core lock service
    └── service/
        └── LockedInventoryService.java      ← Example usage
```

### 1. Core Service: DistributedLockService

**File:** `lock/DistributedLockService.java`

#### Key Components

```java
@Service
public class DistributedLockService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    private static final String LOCK_PREFIX = "lock:";
    private static final Duration DEFAULT_LOCK_TIMEOUT = Duration.ofSeconds(30);
    
    // Lua script for atomic lock release
    private static final String UNLOCK_SCRIPT = 
        "if redis.call('get', KEYS[1]) == ARGV[1] then " +
        "    return redis.call('del', KEYS[1]) " +
        "else " +
        "    return 0 " +
        "end";
}
```

#### Methods

1. **acquireLock()** - Acquire a lock with expiration
2. **releaseLock()** - Release lock atomically (only if we own it)
3. **renewLock()** - Extend lock expiration
4. **executeWithLock()** - Execute code with automatic lock management
5. **executeWithLockAndRenewal()** - Execute with auto-renewal for long operations

---

## When to Use Distributed Locking

### ✅ Use Distributed Locking When:

#### 1. **Preventing Race Conditions**

```
Multiple processes updating same resource:

✅ Inventory updates
✅ Account balance updates
✅ Counter increments
✅ Configuration changes
```

#### 2. **Ensuring Atomic Operations**

```
Operations that must be atomic:

✅ Reserve inventory → Process payment
✅ Deduct balance → Create transaction
✅ Update status → Send notification
```

#### 3. **Preventing Overselling**

```
E-commerce scenarios:

✅ Only one order can reserve last item
✅ Prevent selling more than available
✅ Ensure stock consistency
```

#### 4. **Critical Sections in Distributed Systems**

```
Operations that must be exclusive:

✅ Database migrations
✅ Cache warming
✅ Scheduled jobs
✅ Resource allocation
```

### ❌ Don't Use Distributed Locking When:

#### 1. **Single-Instance Applications**

```
If you have only one service instance:
❌ Use local locks (synchronized, ReentrantLock)
✅ Distributed locks add unnecessary overhead
```

#### 2. **Read-Only Operations**

```
Reading data doesn't need locks:
❌ Reading inventory (unless you need consistent reads)
✅ Only writes need locks
```

#### 3. **Operations That Can Be Idempotent**

```
If operations are idempotent:
❌ Locks might not be necessary
✅ Use idempotency keys instead
```

#### 4. **High-Contention Scenarios**

```
If many processes compete for same lock:
❌ Consider alternative approaches
✅ Use optimistic locking, versioning, or partitioning
```

---

## How Distributed Locking Works

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│         DISTRIBUTED LOCKING ARCHITECTURE                │
└─────────────────────────────────────────────────────────┘

Service Instance 1          Service Instance 2          Service Instance 3
     │                            │                            │
     │  executeWithLock()         │  executeWithLock()         │  executeWithLock()
     │         │                   │         │                   │         │
     │         ▼                   │         ▼                   │         ▼
     │  acquireLock()              │  acquireLock()              │  acquireLock()
     │         │                   │         │                   │         │
     └─────────┼───────────────────┴─────────┼───────────────────┴─────────┼
               │                             │                             │
               └─────────────────────────────┴─────────────────────────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │   Redis Server       │
                              │                      │
                              │  SETNX lock:key      │
                              │  Value: UUID         │
                              │  EXPIRE 30 seconds   │
                              └──────────────────────┘
                                         │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
            Instance 1 gets lock   Instance 2 waits    Instance 3 waits
                    │                   │                   │
                    ▼                   │                   │
            Execute critical section    │                   │
                    │                   │                   │
                    ▼                   │                   │
            Release lock (Lua script)   │                   │
                    │                   │                   │
                    └───────────────────┼───────────────────┘
                                        │
                                        ▼
                              Instance 2 gets lock
                                        │
                                        ▼
                              Execute critical section
```

### Step-by-Step Flow

#### Step 1: Acquire Lock

```java
public LockToken acquireLock(String lockKey, Duration timeout) {
    String fullLockKey = LOCK_PREFIX + lockKey;  // "lock:inventory:123"
    String lockValue = UUID.randomUUID().toString();  // Unique identifier
    
    // Redis SETNX: Set if not exists (atomic operation)
    Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(fullLockKey, lockValue, timeout);
    
    if (acquired) {
        return new LockToken(fullLockKey, lockValue, timeout);
    }
    return null;  // Lock already held by another process
}
```

**What happens:**
1. Generate unique lock value (UUID)
2. Try to set key in Redis (SETNX - Set if Not eXists)
3. Set expiration time (prevents deadlocks)
4. Return lock token if successful

**Redis Command:**
```
SET lock:inventory:123 "uuid-abc-123" EX 30 NX
```

#### Step 2: Execute Critical Section

```java
public <T> T executeWithLock(String lockKey, Duration timeout, 
                             LockedOperation<T> criticalSection) {
    LockToken lockToken = acquireLock(lockKey, timeout);
    if (lockToken == null) {
        throw new LockAcquisitionException("Could not acquire lock");
    }
    
    try {
        return criticalSection.execute();  // Your code here
    } finally {
        releaseLock(lockToken);  // Always release
    }
}
```

**What happens:**
1. Acquire lock
2. Execute critical section (your code)
3. Always release lock in finally block

#### Step 3: Release Lock

```java
public boolean releaseLock(LockToken lockToken) {
    // Lua script ensures atomic release
    // Only releases if we own the lock
    DefaultRedisScript<Long> script = new DefaultRedisScript<>();
    script.setScriptText(UNLOCK_SCRIPT);
    
    Long result = redisTemplate.execute(
        script,
        Collections.singletonList(lockToken.getLockKey()),
        lockToken.getLockValue()
    );
    
    return result != null && result > 0;
}
```

**Lua Script:**
```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```

**Why Lua Script?**
- Atomic operation (prevents releasing someone else's lock)
- Prevents race condition between GET and DEL

### Visual: Lock Lifecycle

```
┌─────────────────────────────────────────────────────────┐
│              LOCK LIFECYCLE                             │
└─────────────────────────────────────────────────────────┘

Time →
│
├─ 0ms:   Process 1: acquireLock("inventory:123")
│         │
│         ├─→ Redis: SETNX lock:inventory:123 "uuid-1" EX 30
│         │
│         └─→ ✅ Lock acquired
│
├─ 5ms:   Process 2: acquireLock("inventory:123")
│         │
│         ├─→ Redis: SETNX lock:inventory:123 "uuid-2" EX 30
│         │
│         └─→ ❌ Lock already exists, returns null
│
├─ 10ms:  Process 1: Execute critical section
│         │
│         └─→ Update inventory...
│
├─ 15ms:  Process 1: releaseLock()
│         │
│         ├─→ Redis: Execute Lua script
│         │   └─→ GET lock:inventory:123
│         │   └─→ If matches uuid-1, DELETE
│         │
│         └─→ ✅ Lock released
│
├─ 20ms:  Process 2: acquireLock("inventory:123") (retry)
│         │
│         ├─→ Redis: SETNX lock:inventory:123 "uuid-2" EX 30
│         │
│         └─→ ✅ Lock acquired (lock was released)
│
└─ 25ms:  Process 2: Execute critical section
```

### Lock Renewal for Long Operations

```java
public <T> T executeWithLockAndRenewal(String lockKey, Duration timeout, 
                                       Duration renewalInterval,
                                       LockedOperation<T> criticalSection) {
    LockToken lockToken = acquireLock(lockKey, timeout);
    
    // Start background thread to renew lock
    Thread renewalThread = new Thread(() -> {
        while (!Thread.currentThread().isInterrupted()) {
            Thread.sleep(renewalInterval.toMillis());
            renewLock(lockToken, timeout);  // Extend expiration
        }
    });
    renewalThread.setDaemon(true);
    renewalThread.start();
    
    try {
        return criticalSection.execute();
    } finally {
        renewalThread.interrupt();
        releaseLock(lockToken);
    }
}
```

**When to use:**
- Operations that might take longer than lock timeout
- Long-running database operations
- External API calls that might be slow

**How it works:**
- Background thread renews lock periodically
- Extends expiration time before it expires
- Stops when operation completes

---

## Best Practices

### 1. Always Use Try-Finally

```java
// ✅ GOOD: Always release lock
LockToken lock = acquireLock(key, timeout);
try {
    // Critical section
} finally {
    releaseLock(lock);
}

// ❌ BAD: Lock might not be released
LockToken lock = acquireLock(key, timeout);
// Critical section
releaseLock(lock);  // If exception occurs, lock not released!
```

### 2. Use executeWithLock() Helper

```java
// ✅ GOOD: Automatic lock management
lockService.executeWithLock(lockKey, timeout, () -> {
    // Critical section
    return result;
});

// ❌ BAD: Manual lock management (error-prone)
LockToken lock = acquireLock(key, timeout);
try {
    // Critical section
} finally {
    releaseLock(lock);
}
```

### 3. Set Appropriate Timeout

```java
// ✅ GOOD: Timeout based on operation duration
lockService.executeWithLock(
    lockKey, 
    Duration.ofSeconds(30),  // Enough time for operation
    () -> {
        // Operation that takes ~20 seconds
    }
);

// ❌ BAD: Too short timeout
lockService.executeWithLock(
    lockKey,
    Duration.ofSeconds(5),  // Operation takes 20 seconds!
    () -> {
        // Lock expires before operation completes
    }
);
```

### 4. Use Unique Lock Keys

```java
// ✅ GOOD: Specific lock keys
String lockKey = "inventory:reserve:" + skuId;  // Specific to SKU
String lockKey = "order:process:" + orderId;    // Specific to order

// ❌ BAD: Too broad lock keys
String lockKey = "inventory";  // Locks ALL inventory operations!
String lockKey = "order";      // Locks ALL order operations!
```

### 5. Handle Lock Acquisition Failures

```java
// ✅ GOOD: Handle failure gracefully
try {
    lockService.executeWithLock(lockKey, timeout, () -> {
        // Critical section
    });
} catch (LockAcquisitionException e) {
    // Retry with backoff, or return error to user
    logger.warn("Could not acquire lock, retrying...");
    // Retry logic
}

// ❌ BAD: Ignore failures
lockService.executeWithLock(lockKey, timeout, () -> {
    // If lock fails, exception is thrown but not handled
});
```

### 6. Use Lock Renewal for Long Operations

```java
// ✅ GOOD: Use renewal for long operations
lockService.executeWithLockAndRenewal(
    lockKey,
    Duration.ofSeconds(30),      // Initial timeout
    Duration.ofSeconds(10),      // Renew every 10 seconds
    () -> {
        // Long-running operation (might take minutes)
        processLargeBatch();
    }
);

// ❌ BAD: Long operation without renewal
lockService.executeWithLock(lockKey, Duration.ofSeconds(30), () -> {
    processLargeBatch();  // Takes 2 minutes, lock expires!
});
```

### 7. Avoid Nested Locks

```java
// ❌ BAD: Nested locks can cause deadlocks
lockService.executeWithLock("lock1", timeout, () -> {
    lockService.executeWithLock("lock2", timeout, () -> {
        // If another process does lock2 then lock1, deadlock!
    });
});

// ✅ GOOD: Acquire all locks at once, or redesign
// Option 1: Acquire in consistent order
lockService.executeWithLock("lock1", timeout, () -> {
    // Do work that needs lock1
});
lockService.executeWithLock("lock2", timeout, () -> {
    // Do work that needs lock2
});

// Option 2: Single lock for related operations
lockService.executeWithLock("operation:123", timeout, () -> {
    // All related work
});
```

### 8. Monitor Lock Contention

```java
// ✅ GOOD: Log lock acquisition attempts
public LockToken acquireLock(String lockKey, Duration timeout) {
    long startTime = System.currentTimeMillis();
    LockToken token = // ... acquire lock
    long duration = System.currentTimeMillis() - startTime;
    
    if (duration > 100) {  // Took more than 100ms
        logger.warn("Lock acquisition took {}ms for key: {}", duration, lockKey);
        // Alert if contention is high
    }
    
    return token;
}
```

### 9. Use Appropriate Lock Granularity

```java
// ✅ GOOD: Fine-grained locks (less contention)
String lockKey = "inventory:" + skuId;  // Lock per SKU
// Multiple SKUs can be updated in parallel

// ❌ BAD: Coarse-grained locks (high contention)
String lockKey = "inventory";  // Lock all inventory
// Only one SKU can be updated at a time
```

### 10. Clean Up Expired Locks

```java
// ✅ GOOD: Redis automatically expires locks
// But monitor for stuck locks

@Scheduled(fixedRate = 60000)  // Every minute
public void checkStuckLocks() {
    // Check for locks that have been held too long
    // Alert if found
}
```

---

## Advanced Interview Questions

### Q1: How does Redis distributed locking work?

**Answer:**
"Redis distributed locking uses the SETNX (Set if Not eXists) command to atomically acquire a lock. Here's how it works:

1. **Acquire Lock:** Use `SET key value EX timeout NX` where:
   - `NX` means set only if key doesn't exist (atomic)
   - `EX timeout` sets expiration to prevent deadlocks
   - `value` is a unique identifier (UUID) to verify ownership

2. **Release Lock:** Use a Lua script to atomically:
   - Check if we own the lock (compare value)
   - Delete the lock only if we own it
   - This prevents releasing someone else's lock

3. **Lock Renewal:** For long operations, periodically extend the expiration time before it expires.

The key advantages are:
- Atomic operations (SETNX, Lua scripts)
- Automatic expiration (prevents deadlocks)
- Unique lock values (prevents releasing wrong lock)"

### Q2: What are the problems with distributed locking?

**Answer:**
"Several challenges:

1. **Clock Skew:** If system clocks are out of sync, lock expiration times can be incorrect. Solution: Use Redis server time, not client time.

2. **Network Partitions:** If network splits, multiple processes might think they own the lock. Solution: Use Redlock algorithm with multiple Redis instances.

3. **Lock Expiration:** If a process holds a lock longer than expiration, another process can acquire it. Solution: Use lock renewal for long operations.

4. **Performance:** Locks serialize operations, reducing parallelism. Solution: Use fine-grained locks, minimize lock duration.

5. **Deadlocks:** If processes acquire multiple locks in different orders. Solution: Always acquire locks in consistent order, or use single lock for related operations.

6. **Lock Contention:** High contention can cause performance issues. Solution: Optimize lock granularity, use optimistic locking where possible."

### Q3: What is the Redlock algorithm?

**Answer:**
"Redlock is a distributed locking algorithm by Redis creator Salvatore Sanfilippo. It addresses the single point of failure problem:

**How it works:**
1. Client tries to acquire lock on N Redis instances (typically 5)
2. Lock is acquired if client gets lock on majority (N/2 + 1) instances
3. Lock expiration time includes network delay and clock drift
4. To release, client sends unlock to all instances

**Advantages:**
- Tolerates failures of minority of Redis instances
- More resilient than single Redis instance

**Challenges:**
- More complex implementation
- Requires multiple Redis instances
- Still has edge cases (network partitions, clock skew)

**In our system:** We use single Redis instance for simplicity, but can extend to Redlock for higher availability."

### Q4: How do you handle lock timeouts?

**Answer:**
"Lock timeouts are critical to prevent deadlocks. Here's our approach:

1. **Set Appropriate Timeout:**
   - Based on expected operation duration
   - Add buffer for network delays
   - Example: If operation takes 20 seconds, set timeout to 30 seconds

2. **Lock Renewal for Long Operations:**
   - Background thread renews lock before expiration
   - Renews at intervals (e.g., every 10 seconds for 30-second lock)
   - Stops when operation completes

3. **Monitor Lock Duration:**
   - Log how long locks are held
   - Alert if locks held longer than expected
   - Helps identify performance issues

4. **Handle Timeout Gracefully:**
   - If lock expires, operation should fail safely
   - Don't continue with stale lock
   - Retry if appropriate

**Code example:**
```java
// Auto-renewal for long operations
lockService.executeWithLockAndRenewal(
    lockKey,
    Duration.ofSeconds(30),      // Initial timeout
    Duration.ofSeconds(10),      // Renew every 10 seconds
    () -> {
        // Long operation
    }
);
```"

### Q5: What happens if Redis goes down?

**Answer:**
"Several scenarios:

1. **During Lock Acquisition:**
   - Operation fails immediately
   - Throw `LockAcquisitionException`
   - Application should handle gracefully (retry, fail fast, or degrade)

2. **During Lock Hold:**
   - If Redis goes down, lock might be lost
   - Other processes might acquire lock
   - Risk of concurrent execution

3. **During Lock Release:**
   - Lock might not be released
   - Will expire automatically when Redis comes back
   - No immediate issue

**Mitigation Strategies:**

1. **Redis High Availability:**
   - Use Redis Sentinel or Cluster
   - Automatic failover
   - Reduces downtime

2. **Circuit Breaker:**
   - If Redis is down, fail fast
   - Don't wait for timeout
   - Return error to user

3. **Fallback Strategy:**
   - Use database-level locking as fallback
   - Or degrade functionality
   - Log for manual intervention

4. **Monitoring:**
   - Alert when Redis is down
   - Monitor lock acquisition failures
   - Track Redis availability

**In our system:** We use Redis with proper error handling and monitoring."

### Q6: How do you prevent releasing someone else's lock?

**Answer:**
"This is a critical issue! Here's how we prevent it:

**Problem:**
```
Process 1: Acquires lock (value: uuid-1, expires in 30s)
Process 1: Takes 35 seconds (lock expires)
Process 2: Acquires lock (value: uuid-2)
Process 1: Tries to release lock
Without protection: Process 1 releases Process 2's lock! ❌
```

**Solution: Use Lua Script**

```lua
-- Atomic check-and-delete
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```

**How it works:**
1. Store unique value (UUID) when acquiring lock
2. When releasing, check if current value matches our UUID
3. Only delete if values match
4. All done atomically in Lua script

**Code:**
```java
public boolean releaseLock(LockToken lockToken) {
    // Lua script ensures atomic release
    Long result = redisTemplate.execute(
        script,
        Collections.singletonList(lockToken.getLockKey()),
        lockToken.getLockValue()  // Our unique UUID
    );
    
    // Returns 1 if deleted, 0 if not ours
    return result != null && result > 0;
}
```

**Why Lua Script?**
- Atomic operation (no race condition between GET and DEL)
- Executed on Redis server (single-threaded, no concurrency issues)
- Prevents releasing wrong lock"

### Q7: What's the difference between optimistic and pessimistic locking?

**Answer:**
"Two different approaches to concurrency control:

**Pessimistic Locking (What we use):**
- Assume conflicts will happen
- Lock resource before accessing
- Other processes wait
- Better for high-contention scenarios
- Example: Distributed locks

**Optimistic Locking:**
- Assume conflicts are rare
- Read resource, modify, write with version check
- If version changed, retry
- Better for low-contention scenarios
- Example: Version fields in database

**Comparison:**

| Aspect | Pessimistic | Optimistic |
|--------|------------|------------|
| Lock | Acquire before operation | No lock, check version |
| Conflicts | Prevented | Detected and retried |
| Performance | Slower (waits) | Faster (no waiting) |
| Use Case | High contention | Low contention |
| Implementation | Distributed locks | Version numbers |

**When to use each:**

**Pessimistic (Distributed Locks):**
- High contention (many processes competing)
- Critical operations (inventory, payments)
- Need guaranteed exclusivity

**Optimistic (Versioning):**
- Low contention (few conflicts)
- Read-heavy workloads
- Can tolerate retries

**In our system:** We use pessimistic locking (distributed locks) for inventory because:
- High contention (many orders)
- Critical (can't oversell)
- Need guaranteed exclusivity"

### Q8: How do you test distributed locking?

**Answer:**
"Multiple testing strategies:

**1. Unit Tests:**
```java
@Test
public void testLockAcquisition() {
    LockToken token = lockService.acquireLock("test", Duration.ofSeconds(10));
    assertNotNull(token);
    
    LockToken token2 = lockService.acquireLock("test", Duration.ofSeconds(10));
    assertNull(token2);  // Should fail, lock already held
    
    lockService.releaseLock(token);
    
    LockToken token3 = lockService.acquireLock("test", Duration.ofSeconds(10));
    assertNotNull(token3);  // Should succeed after release
}
```

**2. Integration Tests:**
```java
@Test
public void testConcurrentAccess() throws InterruptedException {
    AtomicInteger counter = new AtomicInteger(0);
    int threads = 10;
    CountDownLatch latch = new CountDownLatch(threads);
    
    for (int i = 0; i < threads; i++) {
        new Thread(() -> {
            lockService.executeWithLock("test", Duration.ofSeconds(5), () -> {
                int value = counter.get();
                Thread.sleep(100);  // Simulate work
                counter.set(value + 1);
                latch.countDown();
            });
        }).start();
    }
    
    latch.await();
    assertEquals(threads, counter.get());  // All increments should succeed
}
```

**3. Load Tests:**
- Simulate high contention
- Measure lock acquisition time
- Check for deadlocks
- Monitor Redis performance

**4. Chaos Testing:**
- Kill Redis during lock hold
- Network partitions
- Clock skew scenarios
- Verify system handles gracefully"

### Q9: What is lock granularity and why does it matter?

**Answer:**
"Lock granularity refers to the scope of what a lock protects.

**Fine-Grained Locks:**
```java
// Lock per SKU
String lockKey = "inventory:" + skuId;
// Multiple SKUs can be updated in parallel
```

**Coarse-Grained Locks:**
```java
// Lock all inventory
String lockKey = "inventory";
// Only one SKU can be updated at a time
```

**Impact:**

| Granularity | Contention | Parallelism | Complexity |
|------------|-----------|-------------|------------|
| Fine | Low | High | Higher |
| Coarse | High | Low | Lower |

**Example:**

**Fine-grained (Better):**
```
Order 1: Lock inventory:123 → Update SKU 123
Order 2: Lock inventory:456 → Update SKU 456  ← Can run in parallel!
Order 3: Lock inventory:789 → Update SKU 789  ← Can run in parallel!
```

**Coarse-grained (Worse):**
```
Order 1: Lock inventory → Update SKU 123
Order 2: Wait...        ← Blocked!
Order 3: Wait...        ← Blocked!
```

**Best Practice:**
- Use finest granularity possible
- Lock only what you need
- Balance between parallelism and complexity

**In our system:** We use fine-grained locks per SKU for maximum parallelism."

### Q10: How do you handle distributed locks in a microservices architecture?

**Answer:**
"Several considerations:

**1. Lock Service:**
- Centralized lock service (Redis)
- All services use same Redis instance
- Consistent lock semantics

**2. Lock Keys:**
- Use consistent naming convention
- Include service/domain in key
- Example: `inventory:reserve:123`, `order:process:456`

**3. Cross-Service Operations:**
- Use distributed locks for cross-service transactions
- Example: Saga pattern with locks
- Ensure atomicity across services

**4. Service Boundaries:**
- Each service manages its own locks
- Don't share lock keys across services
- Use events for cross-service coordination

**5. Monitoring:**
- Track lock acquisition across services
- Monitor contention
- Alert on lock timeouts

**6. Failure Handling:**
- If lock service down, services should degrade gracefully
- Use circuit breakers
- Have fallback strategies

**Example in our system:**
```java
// Product Service uses locks for inventory
lockService.executeWithLock("inventory:" + skuId, timeout, () -> {
    // Update inventory
});

// Order Service uses locks for order processing
lockService.executeWithLock("order:" + orderId, timeout, () -> {
    // Process order
});
```

**Best Practices:**
- Keep lock scope within service boundaries
- Use events for cross-service coordination
- Monitor lock usage across services
- Document lock key conventions"

---

---

## Real-World Examples: Cinema & Flight Ticket Booking

### Overview

Ticket booking scenarios (cinema, flights) differ from inventory because **each seat is unique**, not multiple units of the same SKU. This requires different lock granularity.

### Key Difference: Inventory vs Tickets

| Scenario | Resource Type | Lock Granularity |
|----------|--------------|------------------|
| **Inventory** | Multiple units of same SKU | Per SKU |
| **Cinema Ticket** | Unique seats per show | Per seat per show |
| **Flight Ticket** | Unique seats per flight | Per seat per flight per date |

---

### Cinema Ticket Booking

#### Lock Granularity Options

##### Option 1: Lock Per Seat Per Show (✅ BEST)

```java
// ✅ BEST: Lock per seat per show
String lockKey = "cinema:show:" + showId + ":seat:" + seatId;

// Example:
// "cinema:show:12345:seat:A12"
// "cinema:show:12345:seat:B05"
```

**Why this works:**
- ✅ Maximum parallelism: Different seats can be booked simultaneously
- ✅ Minimal contention: Only blocks when same seat is selected
- ✅ Prevents double booking

**Example:**
```
User 1: Booking seat A12 for show 12345
User 2: Booking seat B05 for show 12345  ← Can run in parallel! ✅
User 3: Booking seat A12 for show 12345  ← Waits (correctly) ✅
```

##### Option 2: Lock Per Show (❌ NOT RECOMMENDED)

```java
// ❌ BAD: Lock entire show
String lockKey = "cinema:show:" + showId;

// Problem: Only one seat can be booked at a time for entire show!
// User 1: Booking seat A12 → Locks entire show
// User 2: Booking seat B05 → BLOCKED! (unnecessarily)
```

##### Option 3: Lock Per Seat (❌ WRONG)

```java
// ❌ WRONG: Lock per seat (without show)
String lockKey = "cinema:seat:" + seatId;

// Problem: Can't book same seat in different shows simultaneously!
// User 1: Booking seat A12 for show 12345
// User 2: Booking seat A12 for show 67890 → BLOCKED! (wrong!)
```

#### Implementation Example

```java
@Service
public class CinemaBookingService {
    
    @Autowired
    private DistributedLockService lockService;
    
    /**
     * Book a cinema ticket with distributed lock
     */
    public BookingResult bookTicket(Long showId, String seatId, Long userId) {
        // ✅ Lock per seat per show
        String lockKey = "cinema:show:" + showId + ":seat:" + seatId;
        
        return lockService.executeWithLock(lockKey, Duration.ofSeconds(5), () -> {
            // 1. Check if seat is available
            Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
            if (seat == null || seat.isBooked()) {
                return new BookingResult(false, "Seat already booked");
            }
            
            // 2. Reserve seat
            seat.setBooked(true);
            seat.setBookedBy(userId);
            seat.setBookedAt(LocalDateTime.now());
            seatRepository.save(seat);
            
            // 3. Create booking
            Booking booking = new Booking(showId, seatId, userId);
            bookingRepository.save(booking);
            
            return new BookingResult(true, "Seat booked successfully", booking);
        });
    }
    
    /**
     * Book multiple seats (same show)
     */
    public BookingResult bookMultipleSeats(Long showId, List<String> seatIds, Long userId) {
        // Sort seats to prevent deadlocks
        List<String> sortedSeats = seatIds.stream()
            .sorted()
            .collect(Collectors.toList());
        
        // Acquire all locks first
        List<LockToken> locks = new ArrayList<>();
        try {
            // Try to acquire all locks
            for (String seatId : sortedSeats) {
                String lockKey = "cinema:show:" + showId + ":seat:" + seatId;
                LockToken lock = lockService.acquireLock(lockKey, Duration.ofSeconds(5));
                
                if (lock == null) {
                    // Release all acquired locks
                    locks.forEach(lockService::releaseLock);
                    return new BookingResult(false, "Could not acquire all seats");
                }
                locks.add(lock);
            }
            
            // All locks acquired, now book all seats
            List<Booking> bookings = new ArrayList<>();
            for (String seatId : sortedSeats) {
                Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
                if (seat.isBooked()) {
                    // Release all locks and bookings
                    // ... cleanup code
                    return new BookingResult(false, "Seat " + seatId + " already booked");
                }
                
                seat.setBooked(true);
                seatRepository.save(seat);
                
                Booking booking = new Booking(showId, seatId, userId);
                bookingRepository.save(booking);
                bookings.add(booking);
            }
            
            return new BookingResult(true, "All seats booked", bookings);
            
        } finally {
            // Release all locks
            locks.forEach(lockService::releaseLock);
        }
    }
}
```

---

### Flight Ticket Booking

#### Lock Granularity Options

##### Option 1: Lock Per Seat Per Flight Per Date (✅ BEST)

```java
// ✅ BEST: Lock per seat per flight per date
String lockKey = "flight:" + flightNumber + ":date:" + date + ":seat:" + seatId;

// Example:
// "flight:AA123:date:2025-11-15:seat:12A"
// "flight:AA123:date:2025-11-15:seat:12B"
```

**Why this works:**
- ✅ Maximum parallelism
- ✅ Minimal contention
- ✅ Prevents double booking

##### Option 2: Lock Per Flight Per Date (❌ NOT RECOMMENDED)

```java
// ❌ BAD: Lock entire flight
String lockKey = "flight:" + flightNumber + ":date:" + date;

// Problem: Only one seat can be booked at a time!
```

#### Implementation Example

```java
@Service
public class FlightBookingService {
    
    @Autowired
    private DistributedLockService lockService;
    
    /**
     * Book a flight ticket with distributed lock
     */
    public BookingResult bookTicket(String flightNumber, LocalDate date, 
                                    String seatId, Long userId) {
        // ✅ Lock per seat per flight per date
        String lockKey = String.format("flight:%s:date:%s:seat:%s", 
                                      flightNumber, date, seatId);
        
        return lockService.executeWithLock(lockKey, Duration.ofSeconds(5), () -> {
            // 1. Check if seat is available
            Seat seat = seatRepository.findByFlightAndDateAndSeat(
                flightNumber, date, seatId);
            
            if (seat == null || seat.isBooked()) {
                return new BookingResult(false, "Seat already booked");
            }
            
            // 2. Reserve seat
            seat.setBooked(true);
            seat.setBookedBy(userId);
            seat.setBookedAt(LocalDateTime.now());
            seatRepository.save(seat);
            
            // 3. Create booking
            Booking booking = new Booking(flightNumber, date, seatId, userId);
            bookingRepository.save(booking);
            
            return new BookingResult(true, "Seat booked successfully", booking);
        });
    }
}
```

---

### Comparison: Inventory vs Tickets

#### Inventory Locking (Your Current Code)

```java
// Multiple units of same SKU
String lockKey = "inventory:reserve:" + skuId;

// Example: 100 units of SKU-100
// User 1: Reserve 10 units of SKU-100
// User 2: Reserve 5 units of SKU-100  ← Can run in parallel (if enough stock)
// User 3: Reserve 100 units of SKU-100 ← Waits (correctly)
```

**Lock Granularity:** Per SKU (not per unit)

#### Cinema/Flight Ticket Locking

```java
// Each seat is unique
String lockKey = "cinema:show:" + showId + ":seat:" + seatId;

// Example: Seat A12 for show 12345
// User 1: Book seat A12
// User 2: Book seat B05  ← Can run in parallel ✅
// User 3: Book seat A12  ← Waits (correctly) ✅
```

**Lock Granularity:** Per seat per show/flight

---

### Best Practices for Ticket Booking

#### 1. Lock Granularity: Per Seat Per Show/Flight

```java
// ✅ BEST
String lockKey = "cinema:show:" + showId + ":seat:" + seatId;
String lockKey = "flight:" + flightNumber + ":date:" + date + ":seat:" + seatId;
```

#### 2. Short Lock Duration

```java
// ✅ GOOD: Short timeout (booking should be fast)
lockService.executeWithLock(lockKey, Duration.ofSeconds(5), () -> {
    // Check availability + Book seat
    // Should complete in < 1 second
});

// ❌ BAD: Long timeout
lockService.executeWithLock(lockKey, Duration.ofSeconds(30), () -> {
    // Unnecessary long lock
});
```

#### 3. Sort Seats When Booking Multiple

```java
// ✅ GOOD: Sort to prevent deadlocks
List<String> sortedSeats = seatIds.stream()
    .sorted()  // A12, A13, B05, B06
    .collect(Collectors.toList());

// ❌ BAD: Random order (can cause deadlocks)
// User 1: Locks A12 then B05
// User 2: Locks B05 then A12
// → DEADLOCK!
```

#### 4. Acquire All Locks Before Processing

```java
// ✅ GOOD: Acquire all locks first
List<LockToken> locks = new ArrayList<>();
try {
    // Acquire all locks
    for (String seatId : sortedSeats) {
        LockToken lock = acquireLock(seatId);
        if (lock == null) {
            // Release all and fail
            locks.forEach(this::releaseLock);
            throw new BookingException("Could not book all seats");
        }
        locks.add(lock);
    }
    
    // Now process all seats (all locks held)
    bookAllSeats(seats);
    
} finally {
    // Release all locks
    locks.forEach(this::releaseLock);
}
```

#### 5. Handle Partial Failures

```java
// ✅ GOOD: Handle case where some seats are already booked
for (String seatId : sortedSeats) {
    Seat seat = checkAvailability(seatId);
    if (seat.isBooked()) {
        // Rollback already booked seats
        rollbackBookings(alreadyBookedSeats);
        return new BookingResult(false, "Seat " + seatId + " unavailable");
    }
    bookSeat(seatId);
}
```

---

### Visual Comparison

#### Inventory Locking

```
SKU-100 (100 units available)
│
├─ Lock: "inventory:reserve:100"
│
├─ User 1: Reserve 10 units ✅
├─ User 2: Reserve 5 units ✅ (if enough stock)
└─ User 3: Reserve 100 units ⏳ (waits if not enough)
```

#### Ticket Locking

```
Show 12345 (100 seats)
│
├─ Lock: "cinema:show:12345:seat:A12"
│   └─ User 1: Book A12 ✅
│
├─ Lock: "cinema:show:12345:seat:B05"
│   └─ User 2: Book B05 ✅ (parallel)
│
└─ Lock: "cinema:show:12345:seat:A12"
    └─ User 3: Book A12 ⏳ (waits - same seat)
```

---

### Summary: Where to Put Locks

| Scenario | Lock Key | Granularity | Example |
|----------|----------|-------------|---------|
| **Inventory** | `inventory:reserve:{skuId}` | Per SKU | `inventory:reserve:100` |
| **Cinema Ticket** | `cinema:show:{showId}:seat:{seatId}` | Per seat per show | `cinema:show:12345:seat:A12` |
| **Flight Ticket** | `flight:{flightNumber}:date:{date}:seat:{seatId}` | Per seat per flight per date | `flight:AA123:date:2025-11-15:seat:12A` |
| **Payment Processing** | `payment:order:{orderId}` | Per order | `payment:order:ORD-12345` |
| **Payment Refund** | `payment:refund:{orderId}` | Per order | `payment:refund:ORD-12345` |
| **Payment Callback** | `payment:callback:{transactionId}` | Per transaction | `payment:callback:TXN-ABC123` |

**Key Differences:**
- **Inventory:** Lock per SKU (multiple units)
- **Tickets:** Lock per seat per show/flight (unique items)

**Best Practice:** Use the finest granularity that prevents conflicts while maximizing parallelism.

For tickets: **Lock per seat per show/flight**. This allows parallel bookings of different seats while preventing double booking of the same seat.

---

## Handling Multiple Locks: Cinema Booking with Payment

### Overview

In real-world scenarios, you often need to lock **multiple resources** simultaneously. For example, cinema booking requires:
1. **Seat lock** - Prevent double booking
2. **Payment lock** - Prevent duplicate charges

### The Challenge: Multiple Locks

When you need to lock multiple resources, you must:
- Acquire locks in **consistent order** (prevent deadlocks)
- Handle **lock duration conflicts** (different timeouts)
- Release locks in **reverse order**
- Handle **partial failures** gracefully

---

### Lock Duration Conflict Problem

#### The Problem

```java
// ❌ PROBLEM: Different timeouts
LockToken seatLock = acquireLock(seatKey, Duration.ofSeconds(10));  // 10 seconds
LockToken paymentLock = acquireLock(paymentKey, Duration.ofSeconds(60));  // 60 seconds

// What happens:
// 0s:  Acquire seat lock (expires in 10s)
// 1s:  Acquire payment lock (expires in 60s)
// 5s:  Start payment processing (takes 15 seconds)
// 10s: ⚠️ SEAT LOCK EXPIRES! (but payment still processing)
// 15s: Another user can now acquire seat lock for same seat!
// 20s: Payment completes, but seat might be booked by someone else!
```

**Problem:** Seat lock expires while payment is still processing, allowing another user to book the same seat.

---

### Solution 1: Use Same Timeout (✅ RECOMMENDED)

Use the **longer timeout** for both locks to ensure they expire at the same time.

```java
@Service
public class CinemaBookingService {
    
    @Autowired
    private DistributedLockService lockService;
    
    public BookingResult bookTicketWithPayment(Long showId, String seatId, 
                                               Long userId, BigDecimal amount) {
        String seatLockKey = "cinema:show:" + showId + ":seat:" + seatId;
        String orderId = generateOrderId(showId, seatId);
        String paymentLockKey = "payment:order:" + orderId;
        
        // ✅ Use same timeout for both (60 seconds - enough for payment)
        Duration lockTimeout = Duration.ofSeconds(60);
        
        LockToken seatLock = null;
        LockToken paymentLock = null;
        
        try {
            // Acquire seat lock (60 seconds)
            seatLock = lockService.acquireLock(seatLockKey, lockTimeout);
            if (seatLock == null) {
                return new BookingResult(false, "Seat not available");
            }
            
            // Check seat availability
            Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
            if (seat.isBooked()) {
                return new BookingResult(false, "Seat already booked");
            }
            
            // Acquire payment lock (60 seconds - same timeout)
            paymentLock = lockService.acquireLock(paymentLockKey, lockTimeout);
            if (paymentLock == null) {
                return new BookingResult(false, "Payment system busy");
            }
            
            // Both locks held with same timeout - process booking
            return executeBooking(showId, seatId, userId, amount, orderId);
            
        } catch (Exception e) {
            logger.error("Booking failed", e);
            return new BookingResult(false, "Booking failed: " + e.getMessage());
            
        } finally {
            // Release locks in REVERSE ORDER
            if (paymentLock != null) lockService.releaseLock(paymentLock);
            if (seatLock != null) lockService.releaseLock(seatLock);
        }
    }
    
    private BookingResult executeBooking(Long showId, String seatId, 
                                        Long userId, BigDecimal amount, String orderId) {
        try {
            // 1. Reserve seat
            Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
            seat.setBooked(true);
            seat.setBookedBy(userId);
            seatRepository.save(seat);
            
            // 2. Process payment
            PaymentResult paymentResult = paymentGateway.charge(amount, orderId);
            if (!paymentResult.isSuccess()) {
                // Rollback seat reservation
                seat.setBooked(false);
                seat.setBookedBy(null);
                seatRepository.save(seat);
                throw new PaymentException("Payment failed: " + paymentResult.getErrorMessage());
            }
            
            // 3. Create booking
            Booking booking = new Booking();
            booking.setShowId(showId);
            booking.setSeatId(seatId);
            booking.setUserId(userId);
            booking.setOrderId(orderId);
            booking.setTransactionId(paymentResult.getTransactionId());
            booking.setAmount(amount);
            booking.setStatus(BookingStatus.CONFIRMED);
            bookingRepository.save(booking);
            
            return new BookingResult(true, "Booking confirmed", booking);
            
        } catch (PaymentException e) {
            throw e;
        } catch (Exception e) {
            rollbackBooking(showId, seatId);
            throw new BookingException("Booking failed", e);
        }
    }
}
```

**Why this works:**
- ✅ Both locks expire at the same time
- ✅ No conflict between durations
- ✅ Seat lock won't expire before payment completes
- ✅ Simple to implement

---

### Solution 2: Use Lock Renewal for Seat Lock

Use lock renewal to keep the seat lock alive during long payment processing.

```java
public BookingResult bookTicketWithPayment(Long showId, String seatId, 
                                           Long userId, BigDecimal amount) {
    String seatLockKey = "cinema:show:" + showId + ":seat:" + seatId;
    String paymentLockKey = "payment:order:" + generateOrderId(showId, seatId);
    
    // Acquire seat lock with renewal
    return lockService.executeWithLockAndRenewal(
        seatLockKey,
        Duration.ofSeconds(10),      // Initial timeout
        Duration.ofSeconds(5),       // Renew every 5 seconds
        () -> {
            // Inside seat lock (auto-renewed)
            
            // Acquire payment lock (no renewal needed - shorter operation)
            return lockService.executeWithLock(
                paymentLockKey,
                Duration.ofSeconds(60),
                () -> {
                    // Both locks held - process booking
                    return executeBooking(showId, seatId, userId, amount);
                }
            );
        }
    );
}
```

**How it works:**
- Seat lock starts at 10 seconds
- Renews every 5 seconds automatically
- Payment lock is 60 seconds (no renewal needed)
- Seat lock stays alive during payment processing

**When to use:**
- Long payment processing times
- Need flexibility in lock durations
- Payment gateway can be slow

---

### Solution 3: Two-Phase Approach (Best for Long Operations)

Reserve seat first (short lock), then process payment separately (long lock).

```java
public BookingResult bookTicketWithPayment(Long showId, String seatId, 
                                           Long userId, BigDecimal amount) {
    String seatLockKey = "cinema:show:" + showId + ":seat:" + seatId;
    
    // Phase 1: Reserve seat (short lock)
    return lockService.executeWithLock(seatLockKey, Duration.ofSeconds(10), () -> {
        // Check and reserve seat
        Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
        if (seat.isBooked()) {
            throw new BookingException("Seat already booked");
        }
        
        // Reserve seat (mark as reserved, not booked yet)
        seat.setStatus(SeatStatus.RESERVED);
        seat.setReservedBy(userId);
        seat.setReservedUntil(LocalDateTime.now().plusMinutes(5));  // 5 min reservation
        seatRepository.save(seat);
        
        // Create pending booking
        String orderId = generateOrderId(showId, seatId);
        Booking booking = new Booking();
        booking.setOrderId(orderId);
        booking.setStatus(BookingStatus.PENDING_PAYMENT);
        bookingRepository.save(booking);
        
        // Release seat lock (seat is reserved, not booked)
        // Now process payment (separate lock)
        return processPayment(showId, seatId, orderId, amount);
    });
}

private BookingResult processPayment(Long showId, String seatId, 
                                    String orderId, BigDecimal amount) {
    String paymentLockKey = "payment:order:" + orderId;
    
    // Phase 2: Process payment (long lock)
    return lockService.executeWithLock(paymentLockKey, Duration.ofSeconds(60), () -> {
        // Process payment
        PaymentResult result = paymentGateway.charge(amount, orderId);
        
        if (result.isSuccess()) {
            // Confirm booking
            Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
            seat.setStatus(SeatStatus.BOOKED);
            seat.setReservedBy(null);
            seatRepository.save(seat);
            
            Booking booking = bookingRepository.findByOrderId(orderId);
            booking.setStatus(BookingStatus.CONFIRMED);
            booking.setTransactionId(result.getTransactionId());
            bookingRepository.save(booking);
            
            return new BookingResult(true, "Booking confirmed", booking);
        } else {
            // Payment failed - release reservation
            Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
            seat.setStatus(SeatStatus.AVAILABLE);
            seat.setReservedBy(null);
            seatRepository.save(seat);
            
            throw new PaymentException("Payment failed");
        }
    });
}
```

**Benefits:**
- ✅ Seat lock is short (just for reservation)
- ✅ Payment lock is separate (longer timeout)
- ✅ Seat is reserved, not fully booked
- ✅ Reservation expires if payment doesn't complete

**When to use:**
- Very long payment processing times
- Need to minimize lock duration
- Want to allow reservation expiration

---

### Lock Ordering: Prevent Deadlocks

#### The Problem

```
User 1: Locks seat A12 → Tries to lock payment
User 2: Locks payment → Tries to lock seat A12
→ DEADLOCK! ❌
```

#### The Solution: Always Lock in Same Order

```java
// ✅ GOOD: Always lock seat first, then payment
// User 1: Lock seat → Lock payment ✅
// User 2: Lock seat → Lock payment ✅ (waits for seat lock)

// ❌ BAD: Different order
// User 1: Lock seat → Lock payment
// User 2: Lock payment → Lock seat (different order = deadlock!)
```

**Best Practice:**
- Always acquire locks in **consistent order** (e.g., seat first, then payment)
- Always release locks in **reverse order** (payment first, then seat)
- Use sorted keys if locking multiple items

---

### Complete Example: Cinema Booking with Payment

```java
@Service
public class CinemaBookingService {
    
    @Autowired
    private DistributedLockService lockService;
    
    public BookingResult bookTicketWithPayment(Long showId, String seatId, 
                                               Long userId, BigDecimal amount) {
        String seatLockKey = "cinema:show:" + showId + ":seat:" + seatId;
        String orderId = "ORD-" + showId + "-" + seatId + "-" + System.currentTimeMillis();
        String paymentLockKey = "payment:order:" + orderId;
        
        // ✅ Use same timeout for both (60 seconds)
        Duration lockTimeout = Duration.ofSeconds(60);
        
        LockToken seatLock = null;
        LockToken paymentLock = null;
        
        try {
            // Step 1: Acquire seat lock (ALWAYS FIRST - prevents deadlocks)
            seatLock = lockService.acquireLock(seatLockKey, lockTimeout);
            if (seatLock == null) {
                return new BookingResult(false, "Seat not available, please try another seat");
            }
            
            // Step 2: Check seat availability
            Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
            if (seat.isBooked()) {
                return new BookingResult(false, "Seat already booked");
            }
            
            // Step 3: Acquire payment lock (ALWAYS SECOND - same timeout)
            paymentLock = lockService.acquireLock(paymentLockKey, lockTimeout);
            if (paymentLock == null) {
                return new BookingResult(false, "Payment system busy, please try again");
            }
            
            // Step 4: Both locks acquired with same timeout - process booking
            return executeBooking(showId, seatId, userId, amount, orderId);
            
        } catch (Exception e) {
            logger.error("Booking failed", e);
            return new BookingResult(false, "Booking failed: " + e.getMessage());
            
        } finally {
            // Step 5: Release locks in REVERSE ORDER
            if (paymentLock != null) {
                lockService.releaseLock(paymentLock);
            }
            if (seatLock != null) {
                lockService.releaseLock(seatLock);
            }
        }
    }
    
    private BookingResult executeBooking(Long showId, String seatId, 
                                        Long userId, BigDecimal amount, String orderId) {
        try {
            // 1. Reserve seat
            Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
            seat.setBooked(true);
            seat.setBookedBy(userId);
            seatRepository.save(seat);
            
            // 2. Process payment
            PaymentResult paymentResult = paymentGateway.charge(amount, orderId);
            if (!paymentResult.isSuccess()) {
                // Rollback seat reservation
                seat.setBooked(false);
                seat.setBookedBy(null);
                seatRepository.save(seat);
                throw new PaymentException("Payment failed: " + paymentResult.getErrorMessage());
            }
            
            // 3. Create booking
            Booking booking = new Booking();
            booking.setShowId(showId);
            booking.setSeatId(seatId);
            booking.setUserId(userId);
            booking.setOrderId(orderId);
            booking.setTransactionId(paymentResult.getTransactionId());
            booking.setAmount(amount);
            booking.setStatus(BookingStatus.CONFIRMED);
            bookingRepository.save(booking);
            
            return new BookingResult(true, "Booking confirmed", booking);
            
        } catch (PaymentException e) {
            throw e;
        } catch (Exception e) {
            rollbackBooking(showId, seatId);
            throw new BookingException("Booking failed", e);
        }
    }
    
    private void rollbackBooking(Long showId, String seatId) {
        try {
            Seat seat = seatRepository.findByShowIdAndSeatId(showId, seatId);
            if (seat != null && seat.isBooked()) {
                seat.setBooked(false);
                seat.setBookedBy(null);
                seatRepository.save(seat);
            }
        } catch (Exception e) {
            logger.error("Failed to rollback seat", e);
        }
    }
}
```

---

### Visual Flow: Multiple Locks

```
User clicks "Book & Pay"
    │
    ▼
1. Acquire seat lock: "cinema:show:123:seat:A12" (60s timeout)
    │
    ├─→ Success ✅
    │
    └─→ Fail → Return "Seat not available"
    │
    ▼
2. Check seat availability
    │
    ├─→ Already booked → Release lock → Return error
    │
    └─→ Available ✅
    │
    ▼
3. Acquire payment lock: "payment:order:ORD-123-A12-..." (60s timeout)
    │
    ├─→ Success ✅ (Both locks now held, same timeout)
    │
    └─→ Fail → Release seat lock → Return "Payment system busy"
    │
    ▼
4. Both locks held! Process booking:
    │
    ├─→ Reserve seat ✅
    ├─→ Charge payment ✅ (takes 5-30 seconds)
    └─→ Create booking ✅
    │
    ▼
5. Release locks (reverse order):
    │
    ├─→ Release payment lock ✅
    └─→ Release seat lock ✅
    │
    ▼
Booking complete! ✅
```

---

### Comparison of Solutions

| Solution | Pros | Cons | When to Use |
|----------|------|------|-------------|
| **Same Timeout** | Simple, no conflicts | Longer lock on seat | Fast payment processing (< 60s) |
| **Lock Renewal** | Flexible, seat lock stays alive | More complex | Long payment processing |
| **Two-Phase** | Clean separation, shorter locks | More complex logic | Very long payment processing |

---

### Best Practices for Multiple Locks

1. **Consistent Lock Order**
   ```java
   // ✅ GOOD: Always same order
   // Lock seat first, then payment
   
   // ❌ BAD: Different orders cause deadlocks
   ```

2. **Same Timeout for Related Locks**
   ```java
   // ✅ GOOD: Same timeout prevents conflicts
   Duration timeout = Duration.ofSeconds(60);
   lockService.acquireLock(seatKey, timeout);
   lockService.acquireLock(paymentKey, timeout);
   ```

3. **Release in Reverse Order**
   ```java
   // ✅ GOOD: Release in reverse order
   releaseLock(paymentLock);
   releaseLock(seatLock);
   ```

4. **Handle Partial Failures**
   ```java
   // ✅ GOOD: Release acquired locks if later lock fails
   if (paymentLock == null) {
       releaseLock(seatLock);  // Release if payment lock fails
       return error;
   }
   ```

5. **Use Try-Finally**
   ```java
   // ✅ GOOD: Always release in finally
   try {
       // Acquire locks and process
   } finally {
       // Always release locks
   }
   ```

---

### Summary: Multiple Locks

**Key Points:**
- ✅ Use **same timeout** for related locks (prevents conflicts)
- ✅ Always acquire locks in **consistent order** (prevents deadlocks)
- ✅ Always release locks in **reverse order**
- ✅ Handle **partial failures** gracefully
- ✅ Use **lock renewal** for long operations if needed

**Recommended Approach:**
For cinema booking with payment, use **same timeout (60 seconds)** for both seat and payment locks. This ensures:
- No lock duration conflicts
- Simple implementation
- Seat lock won't expire before payment completes

---

## Summary

**Distributed Locking:**
- ✅ Prevents race conditions in distributed systems
- ✅ Ensures atomic operations
- ✅ Uses Redis with SETNX and Lua scripts
- ✅ Automatic expiration prevents deadlocks
- ✅ Lock renewal for long operations

**Key Points:**
- Always use try-finally or executeWithLock()
- Set appropriate timeouts
- Use fine-grained locks
- Handle lock acquisition failures
- Monitor lock contention
- Use lock renewal for long operations

**When to Use:**
- ✅ Preventing race conditions
- ✅ Critical sections
- ✅ High-contention scenarios
- ✅ Ensuring atomicity

**When NOT to Use:**
- ❌ Single-instance applications
- ❌ Read-only operations
- ❌ Low-contention scenarios (use optimistic locking)
- ❌ Operations that can be idempotent

**Lock Granularity Guidelines:**
- **Inventory:** Lock per SKU (multiple units of same product)
- **Tickets:** Lock per seat per show/flight (unique items)
- **Payment Gateway:** Lock per order ID or transaction ID
- **Accounts:** Lock per account ID
- **Orders:** Lock per order ID (if needed)
- **Counters:** Lock per counter name

---

## Payment Gateway Locking

### Overview

Payment gateway operations are **critical** and must be protected with distributed locks to prevent:
- **Duplicate charges** (charging same order twice)
- **Race conditions** (multiple payment attempts for same order)
- **Idempotency issues** (same payment processed multiple times)

### Key Considerations

1. **Idempotency:** Payment operations must be idempotent
2. **Long-running:** Payment gateway calls can take time (5-30 seconds)
3. **Critical:** Financial operations must be accurate
4. **Retries:** Payment gateways may retry, need to handle gracefully

---

### Lock Granularity Options

#### Option 1: Lock Per Order ID (✅ BEST - Most Common)

```java
// ✅ BEST: Lock per order ID
String lockKey = "payment:order:" + orderId;

// Example:
// "payment:order:ORD-12345"
// "payment:order:ORD-67890"
```

**Why this works:**
- ✅ Prevents duplicate payment processing for same order
- ✅ Ensures only one payment attempt per order
- ✅ Works well with Saga pattern
- ✅ Simple and effective

**Use Case:** When processing payment for an order

#### Option 2: Lock Per Transaction ID (✅ GOOD - For Idempotency)

```java
// ✅ GOOD: Lock per transaction ID
String lockKey = "payment:transaction:" + transactionId;

// Example:
// "payment:transaction:TXN-ABC123"
// "payment:transaction:TXN-XYZ789"
```

**Why this works:**
- ✅ Prevents duplicate processing of same transaction
- ✅ Handles payment gateway retries
- ✅ Ensures idempotency

**Use Case:** When handling payment gateway callbacks/webhooks

#### Option 3: Lock Per User Account (⚠️ USE CAREFULLY)

```java
// ⚠️ CAREFUL: Lock per user account
String lockKey = "payment:account:" + userId;

// Problem: Locks ALL payments for a user
// Only use if you need to prevent concurrent payments
```

**When to use:**
- Preventing multiple simultaneous payments from same account
- Balance operations (deducting from account balance)
- Rate limiting per user

**When NOT to use:**
- Regular order payments (too coarse-grained)
- Multiple orders from same user (blocks unnecessarily)

---

### Implementation Examples

#### Example 1: Process Payment with Lock (Order ID)

```java
@Service
public class PaymentService {
    
    @Autowired
    private DistributedLockService lockService;
    
    @Autowired
    private PaymentGatewayClient paymentGateway;
    
    /**
     * Process payment for an order with distributed lock
     */
    public PaymentResult processPayment(String orderId, BigDecimal amount, 
                                       PaymentMethod paymentMethod) {
        // ✅ Lock per order ID
        String lockKey = "payment:order:" + orderId;
        
        return lockService.executeWithLock(lockKey, Duration.ofSeconds(60), () -> {
            // 1. Check if payment already processed (idempotency check)
            Payment existingPayment = paymentRepository.findByOrderId(orderId);
            if (existingPayment != null && existingPayment.isSuccessful()) {
                logger.info("Payment already processed for order: {}", orderId);
                return new PaymentResult(true, "Payment already processed", existingPayment);
            }
            
            // 2. Create payment record (PENDING status)
            Payment payment = new Payment(orderId, amount, paymentMethod);
            payment.setStatus(PaymentStatus.PENDING);
            paymentRepository.save(payment);
            
            try {
                // 3. Call payment gateway
                PaymentGatewayResponse response = paymentGateway.charge(
                    amount, 
                    paymentMethod
                );
                
                // 4. Update payment status
                if (response.isSuccess()) {
                    payment.setStatus(PaymentStatus.SUCCESS);
                    payment.setTransactionId(response.getTransactionId());
                    payment.setProcessedAt(LocalDateTime.now());
                    paymentRepository.save(payment);
                    
                    return new PaymentResult(true, "Payment successful", payment);
                } else {
                    payment.setStatus(PaymentStatus.FAILED);
                    payment.setFailureReason(response.getErrorMessage());
                    paymentRepository.save(payment);
                    
                    return new PaymentResult(false, response.getErrorMessage(), payment);
                }
                
            } catch (PaymentGatewayException e) {
                // 5. Handle payment gateway errors
                payment.setStatus(PaymentStatus.FAILED);
                payment.setFailureReason(e.getMessage());
                paymentRepository.save(payment);
                
                logger.error("Payment gateway error for order: {}", orderId, e);
                throw new PaymentProcessingException("Payment gateway error", e);
            }
        });
    }
}
```

#### Example 2: Handle Payment Gateway Callback with Lock (Transaction ID)

```java
@Service
public class PaymentCallbackService {
    
    @Autowired
    private DistributedLockService lockService;
    
    /**
     * Handle payment gateway callback/webhook with lock
     */
    public void handlePaymentCallback(String transactionId, PaymentStatus status) {
        // ✅ Lock per transaction ID (for idempotency)
        String lockKey = "payment:callback:" + transactionId;
        
        lockService.executeWithLock(lockKey, Duration.ofSeconds(30), () -> {
            // 1. Check if callback already processed
            PaymentCallback callback = callbackRepository.findByTransactionId(transactionId);
            if (callback != null && callback.isProcessed()) {
                logger.info("Callback already processed for transaction: {}", transactionId);
                return; // Idempotent - already processed
            }
            
            // 2. Find payment by transaction ID
            Payment payment = paymentRepository.findByTransactionId(transactionId);
            if (payment == null) {
                logger.warn("Payment not found for transaction: {}", transactionId);
                throw new PaymentNotFoundException("Payment not found");
            }
            
            // 3. Update payment status
            payment.setStatus(status);
            payment.setUpdatedAt(LocalDateTime.now());
            paymentRepository.save(payment);
            
            // 4. Record callback
            PaymentCallback newCallback = new PaymentCallback(transactionId, status);
            newCallback.setProcessed(true);
            callbackRepository.save(newCallback);
            
            // 5. Publish event
            if (status == PaymentStatus.SUCCESS) {
                eventPublisher.publishPaymentSuccess(payment.getOrderId());
            } else {
                eventPublisher.publishPaymentFailure(payment.getOrderId());
            }
        });
    }
}
```

#### Example 3: Refund Payment with Lock

```java
@Service
public class RefundService {
    
    @Autowired
    private DistributedLockService lockService;
    
    /**
     * Refund payment with distributed lock
     */
    public RefundResult refundPayment(String orderId, BigDecimal amount) {
        // ✅ Lock per order ID (prevent duplicate refunds)
        String lockKey = "payment:refund:" + orderId;
        
        return lockService.executeWithLock(lockKey, Duration.ofSeconds(60), () -> {
            // 1. Check if refund already processed
            Refund existingRefund = refundRepository.findByOrderId(orderId);
            if (existingRefund != null && existingRefund.isSuccessful()) {
                return new RefundResult(true, "Refund already processed", existingRefund);
            }
            
            // 2. Find original payment
            Payment payment = paymentRepository.findByOrderId(orderId);
            if (payment == null || !payment.isSuccessful()) {
                throw new RefundException("Payment not found or not successful");
            }
            
            // 3. Create refund record
            Refund refund = new Refund(orderId, amount, payment.getTransactionId());
            refund.setStatus(RefundStatus.PENDING);
            refundRepository.save(refund);
            
            try {
                // 4. Call payment gateway to refund
                RefundGatewayResponse response = paymentGateway.refund(
                    payment.getTransactionId(),
                    amount
                );
                
                // 5. Update refund status
                if (response.isSuccess()) {
                    refund.setStatus(RefundStatus.SUCCESS);
                    refund.setRefundTransactionId(response.getRefundId());
                    refundRepository.save(refund);
                    
                    return new RefundResult(true, "Refund successful", refund);
                } else {
                    refund.setStatus(RefundStatus.FAILED);
                    refund.setFailureReason(response.getErrorMessage());
                    refundRepository.save(refund);
                    
                    return new RefundResult(false, response.getErrorMessage(), refund);
                }
                
            } catch (PaymentGatewayException e) {
                refund.setStatus(RefundStatus.FAILED);
                refund.setFailureReason(e.getMessage());
                refundRepository.save(refund);
                
                throw new RefundException("Refund failed", e);
            }
        });
    }
}
```

#### Example 4: Payment with Lock Renewal (Long Operations)

```java
@Service
public class PaymentService {
    
    /**
     * Process payment with lock renewal (for long-running operations)
     */
    public PaymentResult processPaymentWithRenewal(String orderId, BigDecimal amount) {
        String lockKey = "payment:order:" + orderId;
        
        // Use lock renewal for long operations
        return lockService.executeWithLockAndRenewal(
            lockKey,
            Duration.ofSeconds(60),      // Initial timeout
            Duration.ofSeconds(20),      // Renew every 20 seconds
            () -> {
                // Long-running payment processing
                // Lock will be automatically renewed
                
                // 1. Validate payment
                validatePayment(orderId, amount);
                
                // 2. Call payment gateway (might take 30+ seconds)
                PaymentGatewayResponse response = paymentGateway.charge(amount);
                
                // 3. Process response
                processPaymentResponse(orderId, response);
                
                return new PaymentResult(true, "Payment successful");
            }
        );
    }
}
```

---

### Best Practices for Payment Gateway Locking

#### 1. Lock Per Order ID (Most Common)

```java
// ✅ BEST: Lock per order ID
String lockKey = "payment:order:" + orderId;

// Prevents duplicate payment processing
// Works well with Saga pattern
```

#### 2. Implement Idempotency Checks

```java
// ✅ GOOD: Check if already processed
Payment existingPayment = paymentRepository.findByOrderId(orderId);
if (existingPayment != null && existingPayment.isSuccessful()) {
    return existingPayment; // Idempotent - return existing result
}
```

#### 3. Longer Timeout for Payment Operations

```java
// ✅ GOOD: Longer timeout (payment gateways can be slow)
lockService.executeWithLock(lockKey, Duration.ofSeconds(60), () -> {
    // Payment gateway call (can take 5-30 seconds)
});

// ❌ BAD: Too short timeout
lockService.executeWithLock(lockKey, Duration.ofSeconds(5), () -> {
    // Payment might take longer!
});
```

#### 4. Use Lock Renewal for Long Operations

```java
// ✅ GOOD: Use renewal for long operations
lockService.executeWithLockAndRenewal(
    lockKey,
    Duration.ofSeconds(60),      // Initial timeout
    Duration.ofSeconds(20),      // Renew every 20 seconds
    () -> {
        // Long payment processing
    }
);
```

#### 5. Separate Locks for Different Operations

```java
// ✅ GOOD: Different locks for different operations
String paymentLock = "payment:order:" + orderId;      // For charging
String refundLock = "payment:refund:" + orderId;      // For refunding
String callbackLock = "payment:callback:" + transactionId;  // For callbacks

// Allows: Can process refund while handling callback
// But: Prevents duplicate charges/refunds
```

#### 6. Handle Payment Gateway Timeouts

```java
// ✅ GOOD: Handle timeouts gracefully
try {
    PaymentGatewayResponse response = paymentGateway.charge(amount);
    // Process response
} catch (PaymentGatewayTimeoutException e) {
    // Payment might have succeeded, check status
    PaymentStatus status = paymentGateway.checkStatus(transactionId);
    if (status == PaymentStatus.SUCCESS) {
        // Payment succeeded, update database
    } else {
        // Payment failed or pending
        throw new PaymentProcessingException("Payment timeout");
    }
}
```

#### 7. Store Payment Status Before Gateway Call

```java
// ✅ GOOD: Store status before calling gateway
Payment payment = new Payment(orderId, amount);
payment.setStatus(PaymentStatus.PENDING);  // Set before gateway call
paymentRepository.save(payment);

try {
    // Call payment gateway
    PaymentGatewayResponse response = paymentGateway.charge(amount);
    payment.setStatus(PaymentStatus.SUCCESS);
} catch (Exception e) {
    payment.setStatus(PaymentStatus.FAILED);
} finally {
    paymentRepository.save(payment);
}
```

---

### Integration with Saga Pattern

#### Updated ProcessPaymentStep with Lock

```java
@Component
public class ProcessPaymentStep implements SagaStep {
    
    @Autowired
    private DistributedLockService lockService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Override
    public void execute(SagaContext context) throws Exception {
        String orderId = context.getOrderId();
        BigDecimal amount = calculateAmount(context);
        
        // ✅ Lock per order ID
        String lockKey = "payment:order:" + orderId;
        
        lockService.executeWithLock(lockKey, Duration.ofSeconds(60), () -> {
            // 1. Check idempotency
            Payment existingPayment = paymentRepository.findByOrderId(orderId);
            if (existingPayment != null && existingPayment.isSuccessful()) {
                context.setPaymentProcessed(true);
                return; // Already processed
            }
            
            // 2. Process payment
            PaymentResult result = paymentService.processPayment(orderId, amount);
            
            if (!result.isSuccess()) {
                throw new PaymentProcessingException(result.getErrorMessage());
            }
            
            context.setPaymentProcessed(true);
            context.setTransactionId(result.getTransactionId());
        });
    }
    
    @Override
    public void compensate(SagaContext context) throws Exception {
        if (context.isPaymentProcessed() && context.getTransactionId() != null) {
            // Refund with lock
            String lockKey = "payment:refund:" + context.getOrderId();
            
            lockService.executeWithLock(lockKey, Duration.ofSeconds(60), () -> {
                refundService.refundPayment(context.getOrderId(), context.getTransactionId());
            });
        }
    }
}
```

---

### Visual: Payment Locking Flow

```
┌─────────────────────────────────────────────────────────┐
│         PAYMENT PROCESSING WITH LOCK                    │
└─────────────────────────────────────────────────────────┘

User clicks "Pay"
    │
    ▼
OrderService.checkout()
    │
    ▼
Saga Orchestrator → ProcessPaymentStep
    │
    ├─→ Acquire lock: "payment:order:ORD-123"
    │
    ├─→ Check idempotency (already processed?)
    │   └─→ If yes, return existing payment ✅
    │
    ├─→ Create payment record (PENDING)
    │
    ├─→ Call payment gateway
    │   └─→ Might take 5-30 seconds
    │
    ├─→ Update payment status (SUCCESS/FAILED)
    │
    └─→ Release lock
        │
        ▼
Payment processed ✅
```

---

### Summary: Payment Gateway Locking

| Operation | Lock Key | Granularity | Timeout |
|-----------|----------|-------------|---------|
| **Process Payment** | `payment:order:{orderId}` | Per order | 60 seconds |
| **Refund Payment** | `payment:refund:{orderId}` | Per order | 60 seconds |
| **Payment Callback** | `payment:callback:{transactionId}` | Per transaction | 30 seconds |
| **Account Balance** | `payment:account:{userId}` | Per user | 30 seconds |

**Key Points:**
- ✅ Lock per order ID for payment processing
- ✅ Lock per transaction ID for callbacks (idempotency)
- ✅ Longer timeouts (payment gateways are slow)
- ✅ Use lock renewal for long operations
- ✅ Always check idempotency before processing
- ✅ Store payment status before gateway call
- ✅ Handle timeouts gracefully

**Best Practice:** Use **lock per order ID** for payment processing. This prevents duplicate charges while allowing parallel processing of different orders.

