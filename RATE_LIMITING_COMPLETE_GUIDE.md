# Complete Rate Limiting Guide: RateLimitingFilter & RedisRateLimitingFilter

## Table of Contents
1. [Overview](#overview)
2. [RateLimitingFilter (In-Memory)](#ratelimitingfilter-in-memory)
3. [RedisRateLimitingFilter (Distributed)](#redisratelimitingfilter-distributed)
4. [Flow Diagrams](#flow-diagrams)
5. [All Cases & Scenarios](#all-cases--scenarios)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Code Examples](#code-examples)
8. [Comparison Table](#comparison-table)

---

## Overview

### What is Rate Limiting?
Rate limiting is a technique to control the number of requests a client can make to an API within a specific time window. It prevents:
- **DoS attacks** (Denial of Service)
- **API abuse** and resource exhaustion
- **Cost overruns** in cloud environments
- **Fair usage** enforcement

### Token Bucket Algorithm
Both filters use the **Token Bucket Algorithm**:
- **Bucket**: Contains tokens (capacity = max requests)
- **Refill Rate**: Tokens are added at a fixed rate
- **Consumption**: Each request consumes 1 token
- **Result**: If bucket is empty, request is rejected (HTTP 429)

**Example:**
- Bucket capacity: 60 tokens
- Refill rate: 10 tokens/second
- If 60 requests come instantly, all pass
- After that, only 10 requests/second are allowed

---

## RateLimitingFilter (In-Memory)

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway Instance                      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         RateLimitingFilter (GlobalFilter)            │  │
│  │                                                       │  │
│  │  ┌──────────────────────────────────────────────┐   │  │
│  │  │  ConcurrentHashMap<String, Bucket> buckets   │   │  │
│  │  │  Key: Client IP (e.g., "192.168.1.1")        │   │  │
│  │  │  Value: Bucket4j Bucket instance             │   │  │
│  │  └──────────────────────────────────────────────┘   │  │
│  │                                                       │  │
│  │  Configuration:                                      │  │
│  │  - REQUESTS_PER_MINUTE = 60                         │  │
│  │  - REQUESTS_PER_SECOND = 10                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              GatewayFilterChain                       │  │
│  │         (JWT Filter, Routing, etc.)                   │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

1. **Storage**: `ConcurrentHashMap<String, Bucket>`
   - Key: Client IP address
   - Value: Bucket4j Bucket instance
   - Thread-safe for concurrent requests

2. **Bucket Configuration**:
   ```java
   Refill refill = Refill.intervally(10, Duration.ofSeconds(1));
   Bandwidth limit = Bandwidth.classic(60, refill);
   ```
   - Capacity: 60 tokens
   - Refill: 10 tokens every 1 second

3. **Filter Order**: `-200` (runs before JWT filter)

### Complete Flow Diagram

```
┌─────────────┐
│   Request   │
│   Arrives   │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│  RateLimitingFilter.filter()        │
│  - Extract Client IP                │
│  - Extract Path                     │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Get or Create Bucket               │
│  buckets.computeIfAbsent(ip, ...)   │
│                                     │
│  If IP exists: Return existing      │
│  If new IP: Create new bucket       │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Try to Consume Token               │
│  bucket.tryConsume(1)               │
└──────┬──────────────────────────────┘
       │
       ├──────────────┬──────────────┐
       │              │              │
       ▼              ▼              ▼
   Token Available  Token Not Available
       │              │              │
       │              │              │
       ▼              ▼              ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Add Headers│  │  Log Warning│  │  Return 429 │
│  - X-Rate   │  │  Rate limit │  │  Response   │
│    Limit-   │  │  exceeded   │  │  with error │
│    Limit    │  │             │  │  message    │
│  - X-Rate   │  │             │  │             │
│    Limit-   │  │             │  │             │
│    Remain   │  │             │  │             │
│  - X-Rate   │  │             │  │             │
│    Limit-   │  │             │  │             │
│    Reset    │  │             │  │             │
└──────┬──────┘  └─────────────┘  └─────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Continue Filter Chain              │
│  chain.filter(exchange)             │
└─────────────────────────────────────┘
```

### Code Walkthrough

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // Step 1: Extract request information
    ServerHttpRequest request = exchange.getRequest();
    String clientIp = getClientIp(request);  // e.g., "192.168.1.1"
    String path = request.getURI().getPath(); // e.g., "/api/products"
    
    // Step 2: Get or create bucket for this IP
    // computeIfAbsent: Thread-safe, creates bucket only if IP doesn't exist
    Bucket bucket = buckets.computeIfAbsent(clientIp, k -> createBucket());
    
    // Step 3: Try to consume a token
    if (bucket.tryConsume(1)) {
        // Token available - Request allowed
        long remainingTokens = bucket.getAvailableTokens();
        
        // Step 4: Add rate limit headers to response
        ServerHttpResponse response = exchange.getResponse();
        response.getHeaders().add("X-RateLimit-Limit", "60");
        response.getHeaders().add("X-RateLimit-Remaining", String.valueOf(remainingTokens));
        response.getHeaders().add("X-RateLimit-Reset", String.valueOf(System.currentTimeMillis() / 1000 + 60));
        
        // Step 5: Continue to next filter
        return chain.filter(exchange);
    } else {
        // No token available - Rate limit exceeded
        logger.warn("Rate limit exceeded for IP: {} on path: {}", clientIp, path);
        return onRateLimitExceeded(exchange, clientIp);
    }
}
```

### IP Extraction Logic

```java
private String getClientIp(ServerHttpRequest request) {
    // Priority 1: X-Forwarded-For (when behind proxy/load balancer)
    String xForwardedFor = request.getHeaders().getFirst("X-Forwarded-For");
    if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
        // X-Forwarded-For can contain multiple IPs: "client, proxy1, proxy2"
        // Take the first one (original client)
        return xForwardedFor.split(",")[0].trim();
    }
    
    // Priority 2: X-Real-IP (Nginx/HAProxy)
    String xRealIp = request.getHeaders().getFirst("X-Real-IP");
    if (xRealIp != null && !xRealIp.isEmpty()) {
        return xRealIp;
    }
    
    // Priority 3: Direct connection
    return request.getRemoteAddress() != null 
            ? request.getRemoteAddress().getAddress().getHostAddress() 
            : "unknown";
}
```

### Bucket Creation

```java
private Bucket createBucket() {
    // Refill: Add 10 tokens every 1 second
    Refill refill = Refill.intervally(REQUESTS_PER_SECOND, Duration.ofSeconds(1));
    
    // Bandwidth: Maximum 60 tokens, refill at specified rate
    Bandwidth limit = Bandwidth.classic(REQUESTS_PER_MINUTE, refill);
    
    return Bucket.builder()
            .addLimit(limit)
            .build();
}
```

### Rate Limit Exceeded Response

```java
private Mono<Void> onRateLimitExceeded(ServerWebExchange exchange, String clientIp) {
    ServerHttpResponse response = exchange.getResponse();
    
    // HTTP 429 Too Many Requests
    response.setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
    response.getHeaders().add("Content-Type", "application/json");
    response.getHeaders().add("X-RateLimit-Limit", "60");
    response.getHeaders().add("X-RateLimit-Remaining", "0");
    response.getHeaders().add("Retry-After", "60"); // Retry after 60 seconds
    
    String errorBody = String.format(
        "{\"success\":false,\"message\":\"Rate limit exceeded. Maximum %d requests per minute allowed. Please try again later.\",\"retryAfter\":60}",
        REQUESTS_PER_MINUTE
    );
    
    DataBuffer buffer = response.bufferFactory().wrap(errorBody.getBytes());
    return response.writeWith(Mono.just(buffer));
}
```

---

## RedisRateLimitingFilter (Distributed)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Multiple API Gateway Instances                    │
│                                                                      │
│  ┌──────────────────┐      ┌──────────────────┐                    │
│  │  Gateway-1       │      │  Gateway-2       │                    │
│  │                  │      │                  │                    │
│  │ RedisRateLimit   │      │ RedisRateLimit   │                    │
│  │ Filter           │      │ Filter           │                    │
│  └────────┬─────────┘      └────────┬─────────┘                    │
│           │                          │                              │
│           └──────────┬───────────────┘                              │
│                      │                                              │
│                      ▼                                              │
│           ┌──────────────────────┐                                 │
│           │   Redis Server       │                                 │
│           │                      │                                 │
│           │  Key: ratelimit:ip:  │                                 │
│           │       192.168.1.1:   │                                 │
│           │       /api/products  │                                 │
│           │                      │                                 │
│           │  Value: Bucket State │                                 │
│           │  (Serialized)        │                                 │
│           └──────────────────────┘                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Components

1. **LettuceBasedProxyManager**: 
   - Manages distributed buckets in Redis
   - Uses CAS (Compare-And-Swap) for atomic operations
   - Handles bucket serialization/deserialization

2. **Rate Limit Key Generation**:
   ```java
   // Per-user: "ratelimit:user:123:/api/products"
   // Per-IP: "ratelimit:ip:192.168.1.1:/api/products"
   ```

3. **Tiered Rate Limiting**:
   - Premium users: 120 requests/minute
   - Free users: 60 requests/minute

4. **Expiration Strategy**: Buckets expire after 1 minute of inactivity

### Complete Flow Diagram

```
┌─────────────┐
│   Request   │
│   Arrives   │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│  RedisRateLimitingFilter.filter()   │
│  - Extract Client IP                │
│  - Extract Path                     │
│  - Extract User ID (if authenticated)│
│  - Extract User Tier (premium/free) │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Generate Rate Limit Key            │
│  Priority:                          │
│  1. User ID (if exists)             │
│  2. IP Address                      │
│                                     │
│  Key Format:                        │
│  "ratelimit:user:123:/api/products" │
│  OR                                 │
│  "ratelimit:ip:192.168.1.1:..."    │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Determine Rate Limit               │
│  Based on User Tier:                │
│  - Premium: 120 req/min             │
│  - Free: 60 req/min                 │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Create Bucket Configuration        │
│  BucketConfiguration.builder()      │
│  - Capacity: Based on tier          │
│  - Refill: 10 tokens/second         │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Get/Create Bucket from Redis       │
│  proxyManager.builder()             │
│  .build(key, config)                │
│                                     │
│  Redis Operations:                  │
│  - GET bucket state                 │
│  - If not exists: CREATE            │
│  - If exists: DESERIALIZE           │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Try to Consume Token               │
│  bucket.tryConsume(1)               │
│                                     │
│  Redis Operations:                  │
│  - Atomic CAS operation             │
│  - Update bucket state              │
│  - Return success/failure           │
└──────┬──────────────────────────────┘
       │
       ├──────────────┬──────────────┐
       │              │              │
       ▼              ▼              ▼
   Token Available  Token Not Available
       │              │              │
       │              │              │
       ▼              ▼              ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Add Headers│  │  Log Warning│  │  Return 429 │
│  - X-Rate   │  │  Rate limit │  │  Response   │
│    Limit-   │  │  exceeded   │  │  with error │
│    Limit    │  │             │  │  message    │
│  - X-Rate   │  │             │  │             │
│    Limit-   │  │             │  │             │
│    Remain   │  │             │  │             │
│  - X-Rate   │  │             │  │             │
│    Limit-   │  │             │  │             │
│    Reset    │  │             │  │             │
└──────┬──────┘  └─────────────┘  └─────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│  Continue Filter Chain              │
│  chain.filter(exchange)             │
└─────────────────────────────────────┘
```

### Code Walkthrough

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // Step 1: Extract all request information
    ServerHttpRequest request = exchange.getRequest();
    String clientIp = getClientIp(request);
    String path = request.getURI().getPath();
    String userId = getUserId(request);        // From JWT token
    String userTier = getUserTier(request);    // premium or free
    
    // Step 2: Generate rate limit key
    // Priority: User ID > IP Address
    String rateLimitKeyStr = generateRateLimitKey(clientIp, userId, userTier, path);
    byte[] rateLimitKey = rateLimitKeyStr.getBytes();
    
    // Step 3: Determine rate limit based on user tier
    int limit = "premium".equals(userTier) 
                ? premiumRequestsPerMinute  // 120
                : freeRequestsPerMinute;    // 60
    
    // Step 4: Create bucket configuration
    BucketConfiguration bucketConfig = createBucketConfiguration(limit);
    
    // Step 5: Get or create bucket from Redis
    // This operation is distributed - all gateway instances share the same bucket
    Bucket bucket = proxyManager.builder().build(rateLimitKey, bucketConfig);
    
    // Step 6: Try to consume token (atomic operation in Redis)
    if (bucket.tryConsume(1)) {
        long remainingTokens = bucket.getAvailableTokens();
        
        // Step 7: Add headers
        ServerHttpResponse response = exchange.getResponse();
        response.getHeaders().add("X-RateLimit-Limit", String.valueOf(limit));
        response.getHeaders().add("X-RateLimit-Remaining", String.valueOf(remainingTokens));
        response.getHeaders().add("X-RateLimit-Reset", String.valueOf(System.currentTimeMillis() / 1000 + 60));
        
        return chain.filter(exchange);
    } else {
        return onRateLimitExceeded(exchange, limit);
    }
}
```

### Key Generation Strategy

```java
private String generateRateLimitKey(String clientIp, String userId, String userTier, String path) {
    // Priority 1: Per-user rate limiting (if authenticated)
    if (userId != null) {
        // Format: "ratelimit:user:123:/api/products"
        return String.format("ratelimit:user:%s:%s", userId, path);
    } else {
        // Priority 2: Per-IP rate limiting (anonymous users)
        // Format: "ratelimit:ip:192.168.1.1:/api/products"
        return String.format("ratelimit:ip:%s:%s", clientIp, path);
    }
}
```

**Key Examples:**
- Authenticated user: `ratelimit:user:123:/api/products`
- Anonymous user: `ratelimit:ip:192.168.1.1:/api/products`
- Different endpoints: `ratelimit:ip:192.168.1.1:/api/cart` (separate bucket)

### Tiered Rate Limiting

```java
// Configuration
@Value("${rate.limit.premium.requests.per-minute:120}")
private int premiumRequestsPerMinute;  // Premium: 120 req/min

@Value("${rate.limit.free.requests.per-minute:60}")
private int freeRequestsPerMinute;     // Free: 60 req/min

// Usage
int limit = "premium".equals(userTier) 
            ? premiumRequestsPerMinute 
            : freeRequestsPerMinute;
```

### Redis Proxy Manager Initialization

```java
public RedisRateLimitingFilter(StatefulRedisConnection<byte[], byte[]> redisConnection) {
    this.proxyManager = LettuceBasedProxyManager.builderFor(redisConnection)
            .withExpirationStrategy(
                ExpirationAfterWriteStrategy.basedOnTimeForRefillingBucketUpToMax(
                    Duration.ofMinutes(1)
                )
            )
            .build();
}
```

**Expiration Strategy**: Buckets expire after 1 minute of inactivity to prevent Redis memory bloat.

---

## All Cases & Scenarios

### Case 1: Normal Request Flow (Token Available)

**Scenario**: Client makes request, bucket has tokens

```
Request: GET /api/products
Client IP: 192.168.1.1
Bucket State: 45 tokens remaining

Flow:
1. Filter extracts IP: "192.168.1.1"
2. Gets existing bucket (45 tokens)
3. bucket.tryConsume(1) → true
4. New bucket state: 44 tokens
5. Adds headers:
   - X-RateLimit-Limit: 60
   - X-RateLimit-Remaining: 44
   - X-RateLimit-Reset: <timestamp>
6. Request proceeds to next filter
7. Response: 200 OK
```

### Case 2: Rate Limit Exceeded

**Scenario**: Client exceeds rate limit

```
Request: GET /api/products (Request #61 in 1 minute)
Client IP: 192.168.1.1
Bucket State: 0 tokens remaining

Flow:
1. Filter extracts IP: "192.168.1.1"
2. Gets existing bucket (0 tokens)
3. bucket.tryConsume(1) → false
4. Logs warning: "Rate limit exceeded"
5. Returns 429 response:
   - Status: 429 Too Many Requests
   - Headers:
     * X-RateLimit-Limit: 60
     * X-RateLimit-Remaining: 0
     * Retry-After: 60
   - Body: {"success":false,"message":"Rate limit exceeded..."}
6. Request does NOT proceed to next filter
```

### Case 3: New Client (First Request)

**Scenario**: New IP address makes first request

```
Request: GET /api/products
Client IP: 192.168.1.100 (new IP)
Bucket State: No bucket exists

Flow:
1. Filter extracts IP: "192.168.1.100"
2. buckets.computeIfAbsent() → creates new bucket
3. New bucket: 60 tokens (full capacity)
4. bucket.tryConsume(1) → true
5. New bucket state: 59 tokens
6. Adds headers with remaining: 59
7. Request proceeds
```

### Case 4: Token Refill (Time-Based)

**Scenario**: Tokens refill over time

```
Time: T=0s
Bucket: 0 tokens
Request #1: tryConsume(1) → false (429)

Time: T=1s (refill happens)
Bucket: 10 tokens (refilled)
Request #2: tryConsume(1) → true (200 OK)
Bucket: 9 tokens

Time: T=2s
Bucket: 19 tokens (9 + 10 refilled)
Request #3: tryConsume(1) → true (200 OK)
Bucket: 18 tokens
```

### Case 5: Burst Traffic (Burst Capacity)

**Scenario**: Client sends 60 requests instantly

```
Time: T=0s
Bucket: 60 tokens (full)

Request #1-60: All tryConsume(1) → true
Bucket: 0 tokens

Request #61: tryConsume(1) → false (429)

Time: T=1s
Bucket: 10 tokens (refilled)
Request #62: tryConsume(1) → true (200 OK)
```

### Case 6: Behind Load Balancer (X-Forwarded-For)

**Scenario**: Client behind proxy/load balancer

```
Request Headers:
X-Forwarded-For: 192.168.1.1, 10.0.0.1, 10.0.0.2

Flow:
1. getClientIp() checks X-Forwarded-For
2. Extracts first IP: "192.168.1.1" (original client)
3. Uses this IP for rate limiting
4. Rate limit applies to original client, not proxy
```

### Case 7: Multiple Gateway Instances (Redis)

**Scenario**: Distributed rate limiting with Redis

```
Gateway-1 receives Request #30 from IP 192.168.1.1
Gateway-2 receives Request #31 from IP 192.168.1.1

Flow:
1. Gateway-1: Gets bucket from Redis (30 tokens used)
2. Gateway-1: tryConsume(1) → true, updates Redis (31 tokens used)
3. Gateway-2: Gets bucket from Redis (31 tokens used)
4. Gateway-2: tryConsume(1) → true, updates Redis (32 tokens used)

Both gateways share the same bucket state in Redis!
```

### Case 8: Per-User Rate Limiting (Redis)

**Scenario**: Authenticated user with user ID

```
Request Headers:
Authorization: Bearer <JWT>
X-User-Id: 123
X-User-Tier: premium

Flow:
1. Extract userId: "123"
2. Extract userTier: "premium"
3. Generate key: "ratelimit:user:123:/api/products"
4. Get limit: 120 (premium tier)
5. Get/create bucket from Redis with key
6. Rate limit is per-user, not per-IP
```

### Case 9: Different Endpoints (Separate Buckets)

**Scenario**: Same IP, different endpoints

```
Request 1: GET /api/products
Request 2: GET /api/cart

Flow:
1. Request 1: Key = "ratelimit:ip:192.168.1.1:/api/products"
   - Bucket 1: 60 tokens
   
2. Request 2: Key = "ratelimit:ip:192.168.1.1:/api/cart"
   - Bucket 2: 60 tokens (separate bucket!)

Each endpoint has its own rate limit bucket.
```

### Case 10: Redis Connection Failure (Error Handling)

**Scenario**: Redis is down (RedisRateLimitingFilter)

```
Request arrives
Redis connection fails

Flow:
1. proxyManager.builder().build() throws exception
2. Filter chain may fail or fallback needed
3. In production: Should have fallback mechanism
   - Option 1: Allow all requests (fail-open)
   - Option 2: Block all requests (fail-closed)
   - Option 3: Fallback to in-memory rate limiting
```

### Case 11: Concurrent Requests (Thread Safety)

**Scenario**: Multiple requests from same IP simultaneously

```
Time: T=0s
Bucket: 5 tokens

Request A (Thread 1): tryConsume(1)
Request B (Thread 2): tryConsume(1)
Request C (Thread 3): tryConsume(1)

Flow:
- ConcurrentHashMap ensures thread-safe bucket access
- Bucket4j uses atomic operations for token consumption
- All three requests may succeed if tokens available
- If only 2 tokens: 2 succeed, 1 fails (429)
```

### Case 12: Memory Management (In-Memory Filter)

**Scenario**: Many unique IPs over time

```
Problem: ConcurrentHashMap grows indefinitely
- IP 1: Creates bucket
- IP 2: Creates bucket
- ...
- IP 10000: Creates bucket

Solution Needed:
- Implement bucket expiration (TTL)
- Use Caffeine cache with expiration
- Periodic cleanup of unused buckets
```

---

## Interview Questions & Answers

### Q1: What is the difference between RateLimitingFilter and RedisRateLimitingFilter?

**Answer:**
- **RateLimitingFilter**: In-memory rate limiting using `ConcurrentHashMap`. Works for single gateway instance. Fast but not distributed.
- **RedisRateLimitingFilter**: Distributed rate limiting using Redis. Works across multiple gateway instances. Slower but shared state.

**Use Cases:**
- Single instance: Use RateLimitingFilter
- Multiple instances: Use RedisRateLimitingFilter (or Spring Cloud Gateway's built-in Redis rate limiting)

### Q2: How does the token bucket algorithm work?

**Answer:**
1. **Bucket**: Container with maximum capacity (e.g., 60 tokens)
2. **Refill**: Tokens added at fixed rate (e.g., 10 tokens/second)
3. **Consumption**: Each request consumes 1 token
4. **Result**: If bucket empty → reject (429), else → allow

**Example:**
- Capacity: 60, Refill: 10/sec
- 60 instant requests → all pass
- Request #61 → 429 (bucket empty)
- After 1 second → 10 tokens refilled → 10 requests allowed

### Q3: How do you handle rate limiting in a distributed system?

**Answer:**
1. **Shared State**: Use Redis to store bucket state
2. **Atomic Operations**: Use CAS (Compare-And-Swap) for token consumption
3. **Key Strategy**: Generate unique keys per client (IP/user)
4. **Consistency**: All gateway instances read/write same Redis bucket

**Alternative Approaches:**
- Spring Cloud Gateway's built-in Redis rate limiting
- Load balancer level (Nginx, Kong)
- Separate rate limiting service

### Q4: What happens if Redis is down in RedisRateLimitingFilter?

**Answer:**
- **Current Implementation**: Filter will fail/throw exception
- **Production Solution**: Implement fallback mechanism
  - **Fail-open**: Allow all requests (risky)
  - **Fail-closed**: Block all requests (safe but may cause outage)
  - **Fallback**: Use in-memory rate limiting as backup

### Q5: How do you extract client IP behind a load balancer?

**Answer:**
Priority order:
1. **X-Forwarded-For**: First IP in comma-separated list
2. **X-Real-IP**: Direct client IP (Nginx/HAProxy)
3. **Remote Address**: Direct connection IP

```java
String xForwardedFor = request.getHeaders().getFirst("X-Forwarded-For");
if (xForwardedFor != null) {
    return xForwardedFor.split(",")[0].trim(); // First IP
}
```

### Q6: How do you implement per-user rate limiting?

**Answer:**
1. Extract user ID from JWT token
2. Generate key: `ratelimit:user:{userId}:{path}`
3. Use user-specific bucket in Redis
4. Different limits per user tier (premium/free)

**Key Generation:**
```java
if (userId != null) {
    return "ratelimit:user:" + userId + ":" + path;
} else {
    return "ratelimit:ip:" + clientIp + ":" + path;
}
```

### Q7: What is the filter order and why?

**Answer:**
- **Order: -200** (runs before JWT filter)
- **Reason**: Rate limiting should happen early to:
  1. Block malicious requests before authentication
  2. Save resources (don't process JWT for rate-limited requests)
  3. Protect against DoS attacks

**Filter Chain Order:**
```
RateLimitingFilter (-200) → JwtAuthenticationFilter (-100) → Routing Filter (0)
```

### Q8: How do you handle burst traffic?

**Answer:**
- **Burst Capacity**: Set bucket capacity higher than refill rate
- **Example**: Capacity 60, Refill 10/sec
  - Allows 60 instant requests (burst)
  - Then limits to 10/sec (sustained rate)
- **Trade-off**: Higher capacity = more memory, better burst handling

### Q9: What are the limitations of in-memory rate limiting?

**Answer:**
1. **Not Distributed**: Each gateway instance has separate buckets
2. **Memory Growth**: Buckets never expire (memory leak risk)
3. **No Persistence**: Lost on restart
4. **Scaling Issues**: Rate limits don't aggregate across instances

**Solution**: Use Redis for distributed rate limiting

### Q10: How do you test rate limiting?

**Answer:**
```bash
# Test with curl
for i in {1..70}; do
  curl -v http://localhost:8080/api/products
done

# Expected:
# Requests 1-60: 200 OK
# Requests 61-70: 429 Too Many Requests

# Check headers
curl -v http://localhost:8080/api/products | grep -i "rate"
```

### Q11: What is the difference between sliding window and token bucket?

**Answer:**
- **Token Bucket**: Allows bursts, refills over time
  - Example: 60 capacity, 10/sec refill
  - 60 instant requests → all pass
  
- **Sliding Window**: Fixed window that slides
  - Example: 60 requests per 60-second window
  - More predictable, no burst capacity

**Token Bucket Advantages:**
- Handles burst traffic better
- More flexible refill rates

### Q12: How do you implement tiered rate limiting?

**Answer:**
1. Extract user tier from request (JWT or database)
2. Configure different limits per tier:
   ```java
   int limit = "premium".equals(userTier) 
               ? 120  // Premium: 120 req/min
               : 60;  // Free: 60 req/min
   ```
3. Use tier-specific bucket configuration
4. Apply limit when creating bucket

---

## Code Examples

### Example 1: Testing Rate Limiting

```bash
#!/bin/bash
# Test rate limiting

echo "Testing rate limiting with 70 requests..."

for i in {1..70}; do
  response=$(curl -s -w "\n%{http_code}" http://localhost:8080/api/products)
  http_code=$(echo "$response" | tail -n1)
  
  if [ "$http_code" -eq 429 ]; then
    echo "Request #$i: Rate limit exceeded (429)"
    break
  else
    echo "Request #$i: Success (200)"
  fi
  
  sleep 0.1
done
```

### Example 2: Custom Rate Limit Configuration

```java
// Different limits for different endpoints
private int getRateLimitForPath(String path) {
    if (path.startsWith("/api/admin")) {
        return 10; // Stricter limit for admin endpoints
    } else if (path.startsWith("/api/public")) {
        return 100; // More lenient for public endpoints
    } else {
        return 60; // Default
    }
}
```

### Example 3: Rate Limit Headers Response

```java
// Successful request headers
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1704067200

// Rate limit exceeded headers
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
Retry-After: 60
Content-Type: application/json

{
  "success": false,
  "message": "Rate limit exceeded. Maximum 60 requests per minute allowed.",
  "retryAfter": 60
}
```

### Example 4: Redis Key Patterns

```java
// Per-user, per-endpoint
"ratelimit:user:123:/api/products"
"ratelimit:user:123:/api/cart"

// Per-IP, per-endpoint
"ratelimit:ip:192.168.1.1:/api/products"
"ratelimit:ip:192.168.1.1:/api/cart"

// Global per-user
"ratelimit:user:123:global"

// Global per-IP
"ratelimit:ip:192.168.1.1:global"
```

---

## Comparison Table

| Feature | RateLimitingFilter | RedisRateLimitingFilter |
|---------|-------------------|------------------------|
| **Storage** | In-memory (ConcurrentHashMap) | Redis (distributed) |
| **Scalability** | Single instance only | Multiple instances |
| **Performance** | Very fast (no network) | Slower (Redis network calls) |
| **Consistency** | Per-instance | Shared across instances |
| **Memory** | Grows with unique IPs | Redis manages memory |
| **Persistence** | Lost on restart | Persists in Redis |
| **User-based** | No | Yes (per-user, per-tier) |
| **Endpoint-based** | No | Yes |
| **Thread Safety** | Yes (ConcurrentHashMap) | Yes (Redis atomic ops) |
| **Expiration** | No (memory leak risk) | Yes (configurable TTL) |
| **Use Case** | Development, single instance | Production, distributed |
| **Complexity** | Simple | More complex |

---

## Best Practices

1. **Choose the Right Filter**:
   - Single instance → RateLimitingFilter
   - Multiple instances → RedisRateLimitingFilter or Spring Cloud Gateway's built-in

2. **Set Appropriate Limits**:
   - Too strict → Legitimate users blocked
   - Too lenient → Vulnerable to abuse

3. **Monitor Rate Limits**:
   - Track 429 responses
   - Alert on high rate limit hits
   - Adjust limits based on usage

4. **Handle Edge Cases**:
   - Redis failures (fallback mechanism)
   - Memory growth (bucket expiration)
   - Load balancer IP extraction

5. **Provide Clear Error Messages**:
   - Include retry-after time
   - Explain rate limit exceeded
   - Return proper HTTP 429 status

---

## Summary

### RateLimitingFilter (In-Memory)
- ✅ Fast and simple
- ✅ Good for single instance
- ❌ Not distributed
- ❌ Memory growth issue

### RedisRateLimitingFilter (Distributed)
- ✅ Works across multiple instances
- ✅ Per-user and tiered rate limiting
- ✅ Shared state in Redis
- ❌ Slower (network calls)
- ❌ Requires Redis infrastructure

### Key Takeaways for Interviews
1. **Token Bucket Algorithm**: Capacity + refill rate
2. **Distributed Rate Limiting**: Use Redis for shared state
3. **IP Extraction**: Handle load balancers (X-Forwarded-For)
4. **Filter Order**: Rate limiting before authentication
5. **Tiered Limits**: Different limits for different user types
6. **Error Handling**: Proper 429 responses with retry-after

---

## Additional Resources

- [Spring Cloud Gateway Rate Limiting](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#the-requestratelimiter-gatewayfilter-factory)
- [Bucket4j Documentation](https://bucket4j.com/)
- [Redis Rate Limiting Patterns](https://redis.io/docs/manual/patterns/rate-limiting/)
- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket)

