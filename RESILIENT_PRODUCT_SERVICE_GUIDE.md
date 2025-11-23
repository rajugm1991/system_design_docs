# ResilientProductService - Complete Guide with Scenarios

## Overview

`ResilientProductService` demonstrates how to make service calls resilient using Resilience4j patterns:
- **Circuit Breaker** - Stops calling failing services
- **Retry** - Retries failed requests with exponential backoff
- **Timeout** - Fails fast for slow services
- **Bulkhead** - Isolates resources to prevent cascading failures

## Architecture

```
Product Service (ResilientProductService)
    │
    ├─ Circuit Breaker (sellerService)
    │   └─ Opens after 50% failure rate
    │   └─ Waits 10s before half-open
    │   └─ Allows 3 calls in half-open state
    │
    ├─ Retry (sellerService)
    │   └─ Max 3 attempts
    │   └─ Exponential backoff (1s, 2s, 4s)
    │   └─ Retries on HttpServerErrorException
    │
    ├─ Timeout (sellerService)
    │   └─ 5 seconds timeout
    │   └─ Cancels running future
    │
    └─ Bulkhead (sellerService)
        └─ Max 10 concurrent calls
        └─ Thread pool isolation
        │
        └─ Calls Seller Service
            └─ http://localhost:8083/api/sellers/{sellerId}/catalog/spus
            └─ http://localhost:8083/api/sellers/{sellerId}/catalog/skus
```

## Configuration

### Resilience4j Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      sellerService:
        registerHealthIndicator: true
        slidingWindowSize: 10              # Last 10 calls
        minimumNumberOfCalls: 5            # Need 5 calls before opening
        permittedNumberOfCallsInHalfOpenState: 3  # Allow 3 calls in half-open
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 10s       # Wait 10s before half-open
        failureRateThreshold: 50           # Open at 50% failure rate
        slowCallRateThreshold: 100         # Open at 100% slow calls
        slowCallDurationThreshold: 5s      # Calls > 5s are slow
        recordExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException  # Don't count 4xx as failures

  retry:
    instances:
      sellerService:
        maxAttempts: 3                     # Retry up to 3 times
        waitDuration: 1000                 # Initial wait 1 second
        enableExponentialBackoff: true     # Exponential backoff
        exponentialBackoffMultiplier: 2    # Double wait time each retry
        retryExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.net.SocketTimeoutException

  timelimiter:
    instances:
      sellerService:
        timeoutDuration: 5s                # Timeout after 5 seconds
        cancelRunningFuture: true          # Cancel running future

  bulkhead:
    instances:
      sellerService:
        maxConcurrentCalls: 10             # Max 10 concurrent calls
        maxWaitDuration: 0                 # Don't wait, fail fast

  thread-pool-bulkhead:
    instances:
      sellerService:
        coreThreadPoolSize: 5              # Core thread pool size
        maxThreadPoolSize: 10              # Max thread pool size
        queueCapacity: 25                  # Queue capacity
        keepAliveDuration: 20ms            # Thread keep-alive
```

## How It Works

### Method: `getProductsFromSeller()`

```java
@CircuitBreaker(name = "sellerService", fallbackMethod = "getProductsFromSellerFallback")
@Retry(name = "sellerService", fallbackMethod = "getProductsFromSellerFallback")
@TimeLimiter(name = "sellerService")
@Bulkhead(name = "sellerService", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<List<ProductDTO>> getProductsFromSeller(Long sellerId) {
    // Calls seller-service
}
```

**Execution Flow:**
1. **Bulkhead Check:** Check if thread pool has capacity (max 10 concurrent calls)
2. **Timeout Check:** Set 5-second timeout
3. **Circuit Breaker Check:** Check circuit state (CLOSED/OPEN/HALF_OPEN)
4. **Retry Logic:** Retry on failure (up to 3 times with exponential backoff)
5. **Service Call:** Call seller-service
6. **Fallback:** If all retries fail or circuit is open, call fallback method

## Scenarios

### Scenario 1: Normal Operation (Success)

**Situation:** Seller-service is healthy and responding quickly.

**Flow:**
```
1. Request arrives → getProductsFromSeller(sellerId=1)
2. Bulkhead: Thread pool available ✓
3. Circuit Breaker: CLOSED state ✓
4. Timeout: 5s timeout set ✓
5. Service Call: GET /api/sellers/1/catalog/spus
6. Response: 200 OK with data
7. Result: Products returned successfully
```

**Logs:**
```
INFO  - Fetching products from seller: 1
INFO  - Successfully fetched products for seller: 1
```

**Circuit Breaker State:** CLOSED (normal operation)

---

### Scenario 2: Transient Failure (Retry Success)

**Situation:** Seller-service temporarily fails but recovers.

**Flow:**
```
1. Request arrives → getProductsFromSeller(sellerId=1)
2. Bulkhead: Thread pool available ✓
3. Circuit Breaker: CLOSED state ✓
4. Timeout: 5s timeout set ✓
5. Service Call: GET /api/sellers/1/catalog/spus
6. Response: 500 Internal Server Error ❌
7. Retry 1: Wait 1 second, retry
8. Response: 500 Internal Server Error ❌
9. Retry 2: Wait 2 seconds, retry
10. Response: 200 OK with data ✓
11. Result: Products returned successfully after retry
```

**Logs:**
```
INFO  - Fetching products from seller: 1
ERROR - Error fetching products from seller: 1 (Attempt 1)
INFO  - Retrying... (Wait 1s)
ERROR - Error fetching products from seller: 1 (Attempt 2)
INFO  - Retrying... (Wait 2s)
INFO  - Successfully fetched products for seller: 1 (Attempt 3)
```

**Circuit Breaker State:** CLOSED (failure rate < 50%)

---

### Scenario 3: Persistent Failure (Circuit Breaker Opens)

**Situation:** Seller-service is down or consistently failing.

**Flow:**
```
1. Request 1-5: All fail (500 errors)
2. Circuit Breaker: Calculates failure rate = 100% (> 50% threshold)
3. Circuit Breaker: Opens circuit (state = OPEN)
4. Request 6: Circuit is OPEN
   - Immediately returns fallback (no service call)
   - Response: Empty list from fallback
5. Request 7-10: All return fallback immediately
6. After 10 seconds: Circuit moves to HALF_OPEN
7. Request 11: Circuit is HALF_OPEN
   - Allows 1 call to test service
   - Service call fails
   - Circuit opens again
8. Request 12+: Return fallback immediately
```

**Logs:**
```
INFO  - Fetching products from seller: 1 (Request 1)
ERROR - Error fetching products from seller: 1
INFO  - Fetching products from seller: 1 (Request 2)
ERROR - Error fetching products from seller: 1
... (Requests 3-5 fail)
INFO  - Circuit breaker opened for sellerService
WARN  - Fallback triggered for seller: 1. Reason: Circuit breaker is OPEN
WARN  - Fallback triggered for seller: 1. Reason: Circuit breaker is OPEN
... (Requests 6-10 return fallback immediately)
INFO  - Circuit breaker moved to HALF_OPEN for sellerService
INFO  - Testing service recovery (Request 11)
ERROR - Error fetching products from seller: 1
INFO  - Circuit breaker opened again for sellerService
WARN  - Fallback triggered for seller: 1. Reason: Circuit breaker is OPEN
```

**Circuit Breaker State:** OPEN (stopping calls to failing service)

---

### Scenario 4: Slow Service (Timeout)

**Situation:** Seller-service is slow (takes > 5 seconds to respond).

**Flow:**
```
1. Request arrives → getProductsFromSeller(sellerId=1)
2. Bulkhead: Thread pool available ✓
3. Circuit Breaker: CLOSED state ✓
4. Timeout: 5s timeout set ✓
5. Service Call: GET /api/sellers/1/catalog/spus
6. Service takes 6 seconds to respond
7. Timeout: Request times out after 5 seconds ❌
8. TimeoutException thrown
9. Retry: Retries with exponential backoff
10. Timeout again after 5 seconds ❌
11. Retry: Final retry
12. Timeout again after 5 seconds ❌
13. Fallback: Returns empty list
14. Circuit Breaker: Records slow call (6s > 5s threshold)
15. After multiple slow calls: Circuit opens
```

**Logs:**
```
INFO  - Fetching products from seller: 1
WARN  - Request timed out after 5 seconds
INFO  - Retrying... (Wait 1s)
WARN  - Request timed out after 5 seconds
INFO  - Retrying... (Wait 2s)
WARN  - Request timed out after 5 seconds
WARN  - Fallback triggered for seller: 1. Reason: TimeoutException
INFO  - Circuit breaker recorded slow call (6s > 5s threshold)
```

**Circuit Breaker State:** Opens when slow call rate > 100%

---

### Scenario 5: Service Recovery (Circuit Breaker Closes)

**Situation:** Seller-service recovers after being down.

**Flow:**
```
1. Circuit is OPEN (service was down)
2. After 10 seconds: Circuit moves to HALF_OPEN
3. Request arrives: Circuit is HALF_OPEN
4. Service Call: GET /api/sellers/1/catalog/spus
5. Response: 200 OK with data ✓
6. Circuit Breaker: Records success
7. Circuit Breaker: Closes circuit (state = CLOSED)
8. Subsequent requests: Normal operation resumes
```

**Logs:**
```
INFO  - Circuit breaker moved to HALF_OPEN for sellerService
INFO  - Testing service recovery
INFO  - Successfully fetched products for seller: 1
INFO  - Circuit breaker closed for sellerService (service recovered)
INFO  - Normal operation resumed
```

**Circuit Breaker State:** CLOSED (service recovered)

---

### Scenario 6: Bulkhead Isolation (Concurrent Requests)

**Situation:** Multiple concurrent requests to seller-service.

**Flow:**
```
1. Request 1-10: All accepted (within bulkhead limit)
2. Request 11: Bulkhead is full (10 concurrent calls)
   - If maxWaitDuration > 0: Waits for slot
   - If maxWaitDuration = 0: Fails immediately
3. Request 1 completes: Frees up slot
4. Request 11: Now accepted
5. Request 12-20: Processed as slots become available
```

**Logs:**
```
INFO  - Fetching products from seller: 1 (Request 1)
INFO  - Fetching products from seller: 2 (Request 2)
... (Requests 1-10 accepted)
WARN  - Bulkhead is full, request rejected (Request 11)
INFO  - Request 1 completed, slot freed
INFO  - Fetching products from seller: 11 (Request 11, now accepted)
```

**Bulkhead State:** Isolates concurrent calls (max 10)

---

### Scenario 7: Combined Patterns (Real-World Scenario)

**Situation:** High traffic, service is slow and sometimes fails.

**Flow:**
```
1. High traffic: 100 concurrent requests
2. Bulkhead: Only 10 requests processed concurrently
3. Timeout: Slow requests timeout after 5s
4. Retry: Failed requests retry with exponential backoff
5. Circuit Breaker: Opens after 50% failure rate
6. Fallback: Returns cached/empty data when circuit is open
7. Service Recovery: Circuit closes when service recovers
```

**Logs:**
```
INFO  - 100 requests received
INFO  - 10 requests accepted (bulkhead limit)
WARN  - 90 requests queued/waiting
INFO  - Request 1-5: Success
WARN  - Request 6-10: Timeout after 5s
INFO  - Retrying requests 6-10...
WARN  - Request 6-10: Timeout again
INFO  - Circuit breaker opened (failure rate > 50%)
WARN  - Fallback triggered for requests 11-100
INFO  - Service recovered after 30 seconds
INFO  - Circuit breaker closed
INFO  - Normal operation resumed
```

**Circuit Breaker State:** Opens during failures, closes on recovery

---

## Code Walkthrough

### Method Signature

```java
@CircuitBreaker(name = "sellerService", fallbackMethod = "getProductsFromSellerFallback")
@Retry(name = "sellerService", fallbackMethod = "getProductsFromSellerFallback")
@TimeLimiter(name = "sellerService")
@Bulkhead(name = "sellerService", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<List<ProductDTO>> getProductsFromSeller(Long sellerId)
```

**Annotations Explained:**
1. `@CircuitBreaker` - Opens circuit when failures exceed threshold
2. `@Retry` - Retries failed requests with exponential backoff
3. `@TimeLimiter` - Sets 5-second timeout
4. `@Bulkhead` - Limits concurrent calls to 10

### Execution Order

```
1. Bulkhead Check (Thread Pool)
   └─ If thread pool is full → Reject or wait
   
2. Timeout Set (5 seconds)
   └─ If request takes > 5s → TimeoutException
   
3. Circuit Breaker Check
   └─ If OPEN → Immediately return fallback
   └─ If HALF_OPEN → Allow limited calls
   └─ If CLOSED → Proceed to service call
   
4. Retry Logic
   └─ If failure → Retry with exponential backoff
   └─ Max 3 attempts
   
5. Service Call
   └─ Call seller-service
   └─ Handle response/exception
   
6. Fallback (if all retries fail or circuit is open)
   └─ Return cached data or empty list
```

### Fallback Method

```java
public CompletableFuture<List<ProductDTO>> getProductsFromSellerFallback(Long sellerId, Exception ex) {
    logger.warn("Fallback triggered for seller: {}. Reason: {}", sellerId, ex.getMessage());
    
    // Return empty list or cached data
    return CompletableFuture.completedFuture(new ArrayList<>());
}
```

**Fallback Scenarios:**
- Circuit breaker is OPEN
- All retries exhausted
- Timeout occurred
- Service unavailable

---

## Testing Scenarios

### Test 1: Normal Operation

```bash
# Seller-service is running and healthy
curl http://localhost:8080/api/products

# Expected: Products returned successfully
# Circuit Breaker: CLOSED
# Logs: "Successfully fetched products for seller: X"
```

### Test 2: Service Down (Circuit Breaker)

```bash
# Stop seller-service
# Make requests
curl http://localhost:8080/api/products

# Expected: Fallback triggered, empty list returned
# Circuit Breaker: OPEN after 5 failures
# Logs: "Fallback triggered for seller: X. Reason: ..."
```

### Test 3: Slow Service (Timeout)

```bash
# Simulate slow service (add delay in seller-service)
# Make requests
curl http://localhost:8080/api/products

# Expected: Timeout after 5 seconds, fallback triggered
# Logs: "Request timed out after 5 seconds"
```

### Test 4: High Traffic (Bulkhead)

```bash
# Make 20 concurrent requests
for i in {1..20}; do
  curl http://localhost:8080/api/products &
done

# Expected: Only 10 requests processed concurrently
# Logs: "Bulkhead is full, request rejected"
```

### Test 5: Service Recovery

```bash
# 1. Stop seller-service (circuit opens)
# 2. Wait 10 seconds (circuit moves to half-open)
# 3. Start seller-service
# 4. Make requests

curl http://localhost:8080/api/products

# Expected: Circuit closes, normal operation resumes
# Logs: "Circuit breaker closed for sellerService"
```

---

## Monitoring

### Check Circuit Breaker Status

```bash
# Check circuit breaker health
curl http://localhost:8082/actuator/health

# Check circuit breaker metrics
curl http://localhost:8082/actuator/circuitbreakers

# Check circuit breaker events
curl http://localhost:8082/actuator/circuitbreakerevents
```

### Check Retry Metrics

```bash
# Check retry metrics
curl http://localhost:8082/actuator/retries

# Check retry events
curl http://localhost:8082/actuator/retryevents
```

### Check Bulkhead Metrics

```bash
# Check bulkhead metrics
curl http://localhost:8082/actuator/bulkheads

# Check bulkhead events
curl http://localhost:8082/actuator/bulkheadevents
```

### Check All Metrics

```bash
# Check all resilience metrics
curl http://localhost:8082/actuator/metrics

# Check specific metric
curl http://localhost:8082/actuator/metrics/resilience4j.circuitbreaker.calls
```

---

## Interview Answers

### Q1: How does Resilience4j work?

**Answer:**
"Resilience4j provides resilience patterns through annotations. I use `@CircuitBreaker`, `@Retry`, `@TimeLimiter`, and `@Bulkhead` to make service calls resilient. The circuit breaker opens when failures exceed 50%, stopping calls to the failing service. Retry handles transient failures with exponential backoff. Timeout fails fast for slow services. Bulkhead isolates resources to prevent cascading failures. When the circuit is open or all retries fail, fallback methods provide graceful degradation."

### Q2: Explain circuit breaker states

**Answer:**
"Circuit breaker has three states: CLOSED (normal operation), OPEN (stopping calls), and HALF_OPEN (testing recovery). When failure rate exceeds 50%, it opens and stops calling the service. After 10 seconds, it moves to half-open and allows 3 test calls. If successful, it closes. If failures continue, it opens again. This prevents cascading failures and allows the service to recover."

### Q3: How do you handle slow services?

**Answer:**
"I use timeout to fail fast for slow services. I set a 5-second timeout, so if a service doesn't respond within 5 seconds, the request fails immediately. I combine this with circuit breaker, so if a service is consistently slow (slow call rate > 100%), the circuit opens. I also use bulkhead to isolate slow operations so they don't block other requests."

### Q4: What is the difference between retry and circuit breaker?

**Answer:**
"Retry handles transient failures by retrying the request with exponential backoff. Circuit breaker stops calling a failing service when failures are persistent. I use retry for transient failures (network issues, temporary errors) and circuit breaker for persistent failures (service down, consistent errors). Retry happens before circuit breaker opens, and once the circuit is open, retry is bypassed."

---

## Best Practices

1. **Configure Thresholds:** Set appropriate failure rate thresholds (50% is common)
2. **Set Timeouts:** Always set timeouts to fail fast (5s is reasonable)
3. **Use Fallbacks:** Always provide fallback methods for graceful degradation
4. **Monitor Metrics:** Monitor circuit breaker, retry, and bulkhead metrics
5. **Test Scenarios:** Test all failure scenarios (service down, slow, transient failures)
6. **Combine Patterns:** Use multiple patterns together for comprehensive resilience
7. **Isolate Resources:** Use bulkhead to isolate resources and prevent cascading failures

---

## Summary

ResilientProductService demonstrates:
- ✅ **Circuit Breaker:** Stops calling failing services
- ✅ **Retry:** Handles transient failures
- ✅ **Timeout:** Fails fast for slow services
- ✅ **Bulkhead:** Isolates resources
- ✅ **Fallback:** Provides graceful degradation

All patterns work together to make the service resilient to failures, slow responses, and high traffic.

