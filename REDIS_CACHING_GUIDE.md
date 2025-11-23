# Redis Caching - Complete Implementation Guide

## Table of Contents
1. [What is Redis Caching?](#what-is-redis-caching)
2. [How Redis Caching is Used in Our Codebase](#how-redis-caching-is-used-in-our-codebase)
3. [Code Structure and Implementation](#code-structure-and-implementation)
4. [When to Use Redis Caching](#when-to-use-redis-caching)
5. [How Redis Caching Works](#how-redis-caching-works)
6. [Caching Strategies](#caching-strategies)
7. [Best Practices](#best-practices)
8. [Interview Questions & Answers](#interview-questions--answers)

---

## What is Redis Caching?

### The Problem: Slow Database Queries

**Problem:** Database queries are slow, especially for frequently accessed data.

```
Without Cache:

User Request → Application → Database (slow - 100ms)
User Request → Application → Database (slow - 100ms)
User Request → Application → Database (slow - 100ms)

Every request hits database! ❌
```

**Solution:** Cache frequently accessed data in Redis (fast - 1ms)

```
With Cache:

User Request → Application → Redis Cache (fast - 1ms) ✅
User Request → Application → Redis Cache (fast - 1ms) ✅
User Request → Application → Redis Cache (fast - 1ms) ✅

Only first request hits database, rest from cache! ✅
```

### What is Redis?

**Redis** (Remote Dictionary Server) is an in-memory data store used for:
- **Caching** - Fast data access
- **Distributed Locking** - Coordination
- **Rate Limiting** - Request throttling
- **Session Storage** - User sessions
- **Pub/Sub** - Messaging

**Key Features:**
- In-memory (very fast)
- Key-value store
- Supports TTL (Time To Live)
- Persistence options
- High availability

---

## How Redis Caching is Used in Our Codebase

### Overview

In our e-commerce system, Redis caching is used for:

1. **Product Caching** - Cache product details
2. **Product List Caching** - Cache product lists/search results
3. **Distributed Locking** - Coordinate operations
4. **Rate Limiting** - Limit API requests

### Real-World Scenarios

#### Scenario 1: Product Details Caching

```
User requests product details:
    │
    ├─→ First request: Cache miss
    │   └─→ Fetch from database → Store in cache → Return
    │
    └─→ Subsequent requests: Cache hit
        └─→ Return from cache (fast!) ✅
```

**Code Location:** `cache/ProductCacheService.java`

#### Scenario 2: Product List Caching

```
User searches for products:
    │
    ├─→ First search: Cache miss
    │   └─→ Query database → Cache results → Return
    │
    └─→ Same search: Cache hit
        └─→ Return from cache ✅
```

---

## Code Structure and Implementation

### File Structure

```
product-service/
└── src/main/java/com/ekart/product_service/
    ├── config/
    │   └── RedisConfig.java              ← Redis configuration
    └── cache/
        └── ProductCacheService.java      ← Cache service
```

### 1. Redis Configuration

**File:** `config/RedisConfig.java`

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory();
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate(
            RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new StringRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }

    @Bean
    public RedisTemplate<String, Object> redisObjectTemplate(
            RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }
}
```

**What it does:**
- Configures Redis connection
- Sets up serializers (String for keys/values)
- Creates templates for String and Object operations

### 2. Product Cache Service

**File:** `cache/ProductCacheService.java`

```java
@Service
public class ProductCacheService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    private static final String CACHE_PREFIX = "product:";
    private static final String PRODUCT_LIST_PREFIX = "products:list:";
    private static final Duration PRODUCT_TTL = Duration.ofMinutes(5);
    private static final Duration PRODUCT_LIST_TTL = Duration.ofMinutes(2);

    /**
     * Get product from cache
     */
    public ProductDTO getProduct(Long skuId) {
        String cacheKey = CACHE_PREFIX + skuId;
        String cached = redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            logger.debug("Product found in cache: {}", skuId);
            // Deserialize JSON to ProductDTO
            return deserialize(cached);
        }
        
        logger.debug("Product not found in cache: {}", skuId);
        return null;
    }

    /**
     * Put product in cache
     */
    public void putProduct(Long skuId, ProductDTO product) {
        String cacheKey = CACHE_PREFIX + skuId;
        String json = serialize(product);
        redisTemplate.opsForValue().set(
            cacheKey, 
            json, 
            PRODUCT_TTL.toMinutes(), 
            TimeUnit.MINUTES
        );
        logger.debug("Product cached: {}", skuId);
    }

    /**
     * Invalidate product cache
     */
    public void invalidateProduct(Long skuId) {
        String cacheKey = CACHE_PREFIX + skuId;
        redisTemplate.delete(cacheKey);
        logger.debug("Product cache invalidated: {}", skuId);
    }
}
```

**What it does:**
- Implements cache-aside pattern
- Stores products with TTL
- Invalidates cache on updates
- Uses consistent key naming

### 3. Application Configuration

**File:** `application.yaml`

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms
```

---

## When to Use Redis Caching

### ✅ Use Redis Caching When:

#### 1. **Frequently Accessed Data**

```
Data accessed many times:

✅ Product details (read-heavy)
✅ User profiles
✅ Configuration data
✅ Search results
```

#### 2. **Expensive Database Queries**

```
Queries that are slow:

✅ Complex joins
✅ Aggregations
✅ Full-text search
✅ Computed values
```

#### 3. **Data That Changes Infrequently**

```
Data that doesn't change often:

✅ Product catalog (changes rarely)
✅ User preferences
✅ Static configuration
✅ Reference data
```

#### 4. **High Read-to-Write Ratio**

```
Many reads, few writes:

✅ Product listings (many reads, few updates)
✅ User sessions (many reads, few writes)
✅ Search results (many reads, few writes)
```

### ❌ Don't Use Redis Caching When:

#### 1. **Frequently Changing Data**

```
Data that changes constantly:

❌ Real-time inventory (changes every second)
❌ Stock prices (changes continuously)
❌ Live chat messages
```

#### 2. **Large Objects**

```
Very large objects:

❌ Large files (> 100MB)
❌ Video content
❌ Binary data
```

#### 3. **Critical Data That Must Be Accurate**

```
Data that must be 100% accurate:

❌ Payment transactions
❌ Financial balances
❌ Medical records
```

#### 4. **Simple, Fast Queries**

```
Queries that are already fast:

❌ Simple primary key lookups (< 1ms)
❌ Indexed queries that are fast
```

---

## How Redis Caching Works

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│         REDIS CACHING ARCHITECTURE                      │
└─────────────────────────────────────────────────────────┘

Application
    │
    ├─→ Request: Get Product (SKU-100)
    │
    ├─→ Check Redis Cache
    │   │
    │   ├─→ Cache Hit ✅
    │   │   └─→ Return from Redis (1ms)
    │   │
    │   └─→ Cache Miss ❌
    │       ├─→ Query Database (100ms)
    │       ├─→ Store in Redis
    │       └─→ Return to user
    │
    └─→ Update Product
        ├─→ Update Database
        └─→ Invalidate Cache
```

### Cache-Aside Pattern (What We Use)

```
┌─────────────────────────────────────────────────────────┐
│         CACHE-ASIDE PATTERN                            │
└─────────────────────────────────────────────────────────┘

Read Flow:
    │
    ├─→ 1. Check cache
    │   │
    │   ├─→ Cache Hit: Return from cache ✅
    │   │
    │   └─→ Cache Miss:
    │       ├─→ 2. Query database
    │       ├─→ 3. Store in cache
    │       └─→ 4. Return to user

Write Flow:
    │
    ├─→ 1. Update database
    └─→ 2. Invalidate cache (delete from cache)
```

**Code Example:**
```java
public ProductDTO getProduct(Long skuId) {
    // 1. Check cache
    ProductDTO cached = cacheService.getProduct(skuId);
    if (cached != null) {
        return cached;  // Cache hit
    }
    
    // 2. Cache miss - query database
    ProductDTO product = productRepository.findById(skuId);
    
    // 3. Store in cache
    if (product != null) {
        cacheService.putProduct(skuId, product);
    }
    
    // 4. Return
    return product;
}

public void updateProduct(Long skuId, ProductDTO product) {
    // 1. Update database
    productRepository.save(product);
    
    // 2. Invalidate cache
    cacheService.invalidateProduct(skuId);
}
```

### Visual: Cache Hit vs Cache Miss

#### Cache Hit

```
Request: Get Product SKU-100
    │
    ▼
Check Redis: "product:100"
    │
    ├─→ Found! ✅
    │
    └─→ Return (1ms) ✅
```

#### Cache Miss

```
Request: Get Product SKU-100
    │
    ▼
Check Redis: "product:100"
    │
    ├─→ Not found ❌
    │
    ├─→ Query Database (100ms)
    │
    ├─→ Store in Redis: "product:100" = {...}
    │
    └─→ Return (101ms total)
```

---

## Caching Strategies

### 1. Cache-Aside (Lazy Loading) - What We Use

**How it works:**
- Application checks cache first
- If miss, loads from database
- Stores in cache for future requests
- On update, invalidates cache

**Pros:**
- ✅ Simple to implement
- ✅ Cache only what's needed
- ✅ Works with any database

**Cons:**
- ❌ Cache miss penalty (2 round trips)
- ❌ Possible stale data (if invalidation fails)

**Code:**
```java
// Read
ProductDTO product = cache.get(key);
if (product == null) {
    product = database.get(key);
    cache.put(key, product);
}

// Write
database.update(key, product);
cache.delete(key);  // Invalidate
```

### 2. Write-Through

**How it works:**
- Write to cache and database simultaneously
- Cache always has latest data

**Pros:**
- ✅ Cache always consistent
- ✅ No stale data

**Cons:**
- ❌ Slower writes (2 writes)
- ❌ Writes data that might not be read

**Code:**
```java
// Write
cache.put(key, product);  // Write to cache
database.update(key, product);  // Write to database
```

### 3. Write-Back (Write-Behind)

**How it works:**
- Write to cache first
- Write to database asynchronously (later)

**Pros:**
- ✅ Very fast writes
- ✅ Reduces database load

**Cons:**
- ❌ Risk of data loss (if cache fails)
- ❌ Complex to implement

**Code:**
```java
// Write
cache.put(key, product);  // Write to cache immediately
asyncDatabase.update(key, product);  // Write to database later
```

### 4. Refresh-Ahead

**How it works:**
- Refresh cache before expiration
- Background thread refreshes popular items

**Pros:**
- ✅ Reduces cache misses
- ✅ Better user experience

**Cons:**
- ❌ Wastes resources on unused data
- ❌ Complex to implement

### Comparison Table

| Strategy | Read Performance | Write Performance | Consistency | Complexity |
|----------|-----------------|-------------------|-------------|------------|
| **Cache-Aside** | Fast (if hit) | Fast | Eventual | Low |
| **Write-Through** | Fast | Slow | Strong | Medium |
| **Write-Back** | Fast | Very Fast | Eventual | High |
| **Refresh-Ahead** | Very Fast | Fast | Eventual | High |

**Our Choice:** Cache-Aside (simple, effective, works well for read-heavy workloads)

---

## Best Practices

### 1. Use Consistent Key Naming

```java
// ✅ GOOD: Consistent naming
private static final String CACHE_PREFIX = "product:";
String key = CACHE_PREFIX + skuId;  // "product:100"

// ❌ BAD: Inconsistent naming
String key1 = "prod:" + skuId;
String key2 = "product_" + skuId;
String key3 = "PRODUCT-" + skuId;
```

### 2. Set Appropriate TTL

```java
// ✅ GOOD: TTL based on data volatility
private static final Duration PRODUCT_TTL = Duration.ofMinutes(5);  // Products change rarely
private static final Duration INVENTORY_TTL = Duration.ofMinutes(1);  // Inventory changes often
private static final Duration USER_SESSION_TTL = Duration.ofHours(24);  // Sessions last long

// ❌ BAD: Same TTL for everything
private static final Duration TTL = Duration.ofMinutes(5);  // Too generic
```

**TTL Guidelines:**
- **Static data:** Hours or days
- **Product catalog:** 5-15 minutes
- **Inventory:** 1-2 minutes
- **User sessions:** 24 hours
- **Search results:** 2-5 minutes

### 3. Handle Cache Failures Gracefully

```java
// ✅ GOOD: Handle failures gracefully
public ProductDTO getProduct(Long skuId) {
    try {
        ProductDTO cached = cacheService.getProduct(skuId);
        if (cached != null) {
            return cached;
        }
    } catch (Exception e) {
        logger.warn("Cache read failed, falling back to database", e);
        // Continue to database
    }
    
    // Fallback to database
    return productRepository.findById(skuId);
}

// ❌ BAD: Fail if cache fails
public ProductDTO getProduct(Long skuId) {
    ProductDTO cached = cacheService.getProduct(skuId);  // Throws exception if Redis down
    if (cached != null) {
        return cached;
    }
    return productRepository.findById(skuId);
}
```

### 4. Invalidate Cache on Updates

```java
// ✅ GOOD: Invalidate on update
public void updateProduct(Long skuId, ProductDTO product) {
    productRepository.save(product);
    cacheService.invalidateProduct(skuId);  // Invalidate cache
}

// ❌ BAD: Forget to invalidate
public void updateProduct(Long skuId, ProductDTO product) {
    productRepository.save(product);
    // Cache still has old data! ❌
}
```

### 5. Use SCAN Instead of KEYS

```java
// ✅ GOOD: Use SCAN for pattern matching
public void invalidateAllProducts() {
    Set<String> keys = new HashSet<>();
    String pattern = CACHE_PREFIX + "*";
    
    ScanOptions options = ScanOptions.scanOptions()
        .match(pattern)
        .count(100)
        .build();
    
    try (Cursor<String> cursor = redisTemplate.scan(options)) {
        while (cursor.hasNext()) {
            keys.add(cursor.next());
        }
    }
    
    if (!keys.isEmpty()) {
        redisTemplate.delete(keys);
    }
}

// ❌ BAD: Use KEYS (blocks Redis)
public void invalidateAllProducts() {
    redisTemplate.delete(redisTemplate.keys(CACHE_PREFIX + "*"));  // Blocks!
}
```

### 6. Serialize Objects Properly

```java
// ✅ GOOD: Use JSON serialization
@Autowired
private ObjectMapper objectMapper;

public void putProduct(Long skuId, ProductDTO product) {
    try {
        String json = objectMapper.writeValueAsString(product);
        redisTemplate.opsForValue().set(cacheKey, json, TTL);
    } catch (Exception e) {
        logger.error("Failed to serialize product", e);
    }
}

public ProductDTO getProduct(Long skuId) {
    String json = redisTemplate.opsForValue().get(cacheKey);
    if (json != null) {
        try {
            return objectMapper.readValue(json, ProductDTO.class);
        } catch (Exception e) {
            logger.error("Failed to deserialize product", e);
        }
    }
    return null;
}
```

### 7. Monitor Cache Hit Rate

```java
// ✅ GOOD: Monitor cache performance
@Service
public class ProductCacheService {
    
    private final AtomicLong cacheHits = new AtomicLong(0);
    private final AtomicLong cacheMisses = new AtomicLong(0);
    
    public ProductDTO getProduct(Long skuId) {
        ProductDTO cached = getFromCache(skuId);
        if (cached != null) {
            cacheHits.incrementAndGet();
            return cached;
        }
        
        cacheMisses.incrementAndGet();
        // ... fetch from database
    }
    
    public double getHitRate() {
        long total = cacheHits.get() + cacheMisses.get();
        return total > 0 ? (double) cacheHits.get() / total : 0.0;
    }
}
```

### 8. Cache Warming

```java
// ✅ GOOD: Pre-load frequently accessed data
@PostConstruct
public void warmCache() {
    List<Long> popularProducts = getPopularProductIds();
    cacheService.warmCache(popularProducts);
}

public void warmCache(List<Long> productIds) {
    productIds.forEach(skuId -> {
        ProductDTO product = productRepository.findById(skuId);
        if (product != null) {
            cacheService.putProduct(skuId, product);
        }
    });
}
```

### 9. Use Hash for Related Data

```java
// ✅ GOOD: Use Hash for product with multiple fields
public void cacheProduct(Long skuId, ProductDTO product) {
    String key = "product:" + skuId;
    redisTemplate.opsForHash().put(key, "name", product.getName());
    redisTemplate.opsForHash().put(key, "price", product.getPrice());
    redisTemplate.opsForHash().put(key, "quantity", product.getQuantity());
    redisTemplate.expire(key, PRODUCT_TTL);
}

// Allows partial updates
public void updateProductPrice(Long skuId, BigDecimal price) {
    String key = "product:" + skuId;
    redisTemplate.opsForHash().put(key, "price", price);
}
```

### 10. Set Memory Limits

```yaml
# Redis configuration
maxmemory 2gb
maxmemory-policy allkeys-lru  # Evict least recently used
```

**Eviction Policies:**
- `allkeys-lru`: Evict least recently used (recommended)
- `allkeys-lfu`: Evict least frequently used
- `volatile-lru`: Evict LRU from keys with TTL
- `noeviction`: Don't evict (can cause OOM)

---

## Interview Questions & Answers

### Q1: What is Redis and why use it for caching?

**Answer:**
"Redis is an in-memory data store that's extremely fast (sub-millisecond latency). I use it for caching because:

1. **Performance:** Much faster than database (1ms vs 100ms)
2. **Reduces Database Load:** Fewer queries to database
3. **Scalability:** Can handle millions of requests
4. **TTL Support:** Automatic expiration of cached data
5. **Data Structures:** Supports strings, hashes, lists, sets

In our system, I use Redis to cache product details and search results, reducing database load by 80% and improving response times from 100ms to 1ms for cached data."

### Q2: What caching strategies do you know?

**Answer:**
"There are several caching strategies:

1. **Cache-Aside (Lazy Loading):** Application manages cache. Check cache, if miss load from DB, then cache. On update, invalidate cache. This is what we use.

2. **Write-Through:** Write to cache and database simultaneously. Cache always has latest data but slower writes.

3. **Write-Back (Write-Behind):** Write to cache first, write to database asynchronously. Very fast but risk of data loss.

4. **Refresh-Ahead:** Refresh cache before expiration. Reduces misses but wastes resources.

I use Cache-Aside because it's simple, effective, and works well for read-heavy workloads like e-commerce."

### Q3: How do you handle cache invalidation?

**Answer:**
"I handle cache invalidation in multiple ways:

1. **TTL (Time To Live):** Automatic expiration after time period
   - Products: 5 minutes
   - Inventory: 1 minute
   - Search results: 2 minutes

2. **Explicit Invalidation:** Delete cache on updates
   ```java
   public void updateProduct(Long skuId, ProductDTO product) {
       productRepository.save(product);
       cacheService.invalidateProduct(skuId);  // Delete from cache
   }
   ```

3. **Pattern-Based Invalidation:** Invalidate related caches
   ```java
   public void updateProduct(Long skuId) {
       cacheService.invalidateProduct(skuId);
       cacheService.invalidateProductList("seller:" + sellerId);  // Invalidate list
   }
   ```

4. **Event-Based Invalidation:** Use events to invalidate
   - Product updated event → Invalidate cache
   - Inventory changed event → Invalidate inventory cache

I also use SCAN instead of KEYS for pattern matching to avoid blocking Redis."

### Q4: What happens when Redis goes down?

**Answer:**
"When Redis goes down, I handle it gracefully:

1. **Cache Failures Don't Break Application:**
   ```java
   try {
       ProductDTO cached = cacheService.getProduct(skuId);
       if (cached != null) return cached;
   } catch (Exception e) {
       logger.warn("Cache unavailable, using database", e);
   }
   // Fallback to database
   return productRepository.findById(skuId);
   ```

2. **Circuit Breaker Pattern:**
   - Detect Redis failures
   - Stop trying after threshold
   - Fallback to database
   - Retry after timeout

3. **High Availability:**
   - Use Redis Sentinel or Cluster
   - Automatic failover
   - Reduces downtime

4. **Monitoring:**
   - Alert when Redis is down
   - Track cache hit rate
   - Monitor fallback usage

**Result:** Application continues to function, just slower (no cache)."

### Q5: How do you prevent cache stampede/thundering herd?

**Answer:**
"Cache stampede happens when cache expires and many requests hit database simultaneously.

**Solutions:**

1. **Lock-Based Approach:**
   ```java
   public ProductDTO getProduct(Long skuId) {
       ProductDTO cached = cacheService.getProduct(skuId);
       if (cached != null) return cached;
       
       // Acquire lock
       String lockKey = "lock:product:" + skuId;
       if (distributedLock.acquire(lockKey)) {
           try {
               // Double-check cache
               cached = cacheService.getProduct(skuId);
               if (cached != null) return cached;
               
               // Only one thread loads from DB
               ProductDTO product = productRepository.findById(skuId);
               cacheService.putProduct(skuId, product);
               return product;
           } finally {
               distributedLock.release(lockKey);
           }
       } else {
           // Wait and retry
           Thread.sleep(100);
           return getProduct(skuId);
       }
   }
   ```

2. **Probabilistic Early Expiration:**
   - Refresh cache before expiration
   - Reduces simultaneous expirations

3. **Background Refresh:**
   - Background thread refreshes popular items
   - Users always get cached data"

### Q6: How do you handle cache consistency?

**Answer:**
"Cache consistency is challenging. I use multiple approaches:

1. **TTL-Based Expiration:**
   - Accept eventual consistency
   - Data becomes consistent after TTL
   - Good for read-heavy workloads

2. **Invalidation on Updates:**
   ```java
   public void updateProduct(Long skuId, ProductDTO product) {
       productRepository.save(product);
       cacheService.invalidateProduct(skuId);  // Delete cache
   }
   ```

3. **Write-Through for Critical Data:**
   - Update cache and database together
   - Stronger consistency
   - Slower writes

4. **Version-Based Invalidation:**
   - Store version with data
   - Compare versions
   - Invalidate if version mismatch

5. **Event-Driven Invalidation:**
   - Publish events on updates
   - Consumers invalidate cache
   - Works across services

**Trade-off:** Stronger consistency = slower performance. I choose eventual consistency for most data (products, lists) and strong consistency only for critical data (payments)."

### Q7: What is cache warming and when do you use it?

**Answer:**
"Cache warming is pre-loading frequently accessed data into cache.

**When to use:**
- Application startup
- After cache flush
- Popular products
- Frequently searched items

**Implementation:**
```java
@PostConstruct
public void warmCache() {
    // Load top 100 products
    List<Long> popularProducts = productRepository.findTop100ByPopularity();
    popularProducts.forEach(skuId -> {
        ProductDTO product = productRepository.findById(skuId);
        cacheService.putProduct(skuId, product);
    });
}
```

**Benefits:**
- Reduces cache misses
- Better user experience
- Lower database load

**Considerations:**
- Don't warm too much (wastes memory)
- Focus on frequently accessed data
- Monitor hit rate to adjust"

### Q8: How do you choose TTL values?

**Answer:**
"I choose TTL based on:

1. **Data Volatility:**
   - Static data (config): Hours/days
   - Product catalog: 5-15 minutes
   - Inventory: 1-2 minutes
   - User sessions: 24 hours

2. **Business Requirements:**
   - How stale can data be?
   - Real-time requirements?
   - Acceptable inconsistency window?

3. **Update Frequency:**
   - Frequently updated: Shorter TTL
   - Rarely updated: Longer TTL

4. **Cache Size:**
   - Limited memory: Shorter TTL
   - Plenty of memory: Longer TTL

**Example:**
```java
// Products change rarely
private static final Duration PRODUCT_TTL = Duration.ofMinutes(5);

// Inventory changes often
private static final Duration INVENTORY_TTL = Duration.ofMinutes(1);

// Configuration changes very rarely
private static final Duration CONFIG_TTL = Duration.ofHours(24);
```

I monitor cache hit rate and adjust TTLs based on performance."

### Q9: What's the difference between cache and database?

**Answer:**
"Key differences:

| Aspect | Cache (Redis) | Database |
|--------|---------------|----------|
| **Storage** | In-memory | Disk |
| **Speed** | Very fast (1ms) | Slower (100ms) |
| **Persistence** | Optional | Always |
| **Size** | Limited (RAM) | Large (disk) |
| **Cost** | Expensive | Cheaper |
| **Use Case** | Temporary, fast access | Permanent storage |

**When to use each:**
- **Cache:** Frequently accessed, can be regenerated, temporary
- **Database:** Permanent storage, source of truth, transactional

**Best Practice:** Use both - cache for performance, database for persistence."

### Q10: How do you monitor Redis cache?

**Answer:**
"I monitor Redis in multiple ways:

1. **Cache Hit Rate:**
   ```java
   double hitRate = (double) cacheHits / (cacheHits + cacheMisses);
   // Target: > 80%
   ```

2. **Redis Metrics:**
   - Memory usage
   - Number of keys
   - Evictions
   - Commands per second

3. **Application Metrics:**
   - Response time (with vs without cache)
   - Database query reduction
   - Cache operation latency

4. **Monitoring Tools:**
   - Redis CLI: `INFO stats`
   - Prometheus + Grafana
   - Application logs

5. **Alerts:**
   - Low hit rate (< 70%)
   - High memory usage (> 80%)
   - Redis unavailable
   - High eviction rate

**Example:**
```java
@Scheduled(fixedRate = 60000)
public void monitorCache() {
    double hitRate = getHitRate();
    if (hitRate < 0.7) {
        logger.warn("Low cache hit rate: {}", hitRate);
        alertService.sendAlert("Cache hit rate below threshold");
    }
}
```"

---

## Summary

**Redis Caching:**
- ✅ Improves performance (1ms vs 100ms)
- ✅ Reduces database load
- ✅ Uses cache-aside pattern
- ✅ TTL-based expiration
- ✅ Graceful degradation on failures

**Key Points:**
- Use consistent key naming
- Set appropriate TTLs
- Handle cache failures gracefully
- Invalidate on updates
- Monitor cache hit rate
- Use SCAN instead of KEYS
- Serialize objects properly

**When to Use:**
- ✅ Frequently accessed data
- ✅ Expensive queries
- ✅ High read-to-write ratio
- ✅ Data that changes infrequently

**When NOT to Use:**
- ❌ Frequently changing data
- ❌ Critical data requiring strong consistency
- ❌ Very large objects
- ❌ Simple, fast queries

**In Our System:**
- Product caching with 5-minute TTL
- Product list caching with 2-minute TTL
- Cache-aside pattern
- Invalidation on updates
- Graceful fallback to database

