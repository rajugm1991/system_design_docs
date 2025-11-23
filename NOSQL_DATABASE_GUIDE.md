# NoSQL Database - Complete Implementation Guide

## Table of Contents
1. [What is NoSQL?](#what-is-nosql)
2. [NoSQL Databases Used in Our Codebase](#nosql-databases-used-in-our-codebase)
3. [Redis - In-Memory Data Store](#redis---in-memory-data-store)
4. [Redis Implementation in Codebase](#redis-implementation-in-codebase)
5. [Other NoSQL Databases](#other-nosql-databases)
6. [When to Use NoSQL](#when-to-use-nosql)
7. [Best Practices](#best-practices)
8. [Real-Time Examples](#real-time-examples)
9. [Strategies and Patterns](#strategies-and-patterns)
10. [Interview Questions](#interview-questions)

---

## What is NoSQL?

**NoSQL (Not Only SQL)** databases are non-relational databases designed for:
- **Horizontal scaling**: Scale across multiple servers
- **Flexible schemas**: No fixed schema, adapts to data
- **High performance**: Optimized for specific use cases
- **Large volumes**: Handle big data efficiently

### Types of NoSQL Databases:

1. **Key-Value Stores**: Redis, DynamoDB
2. **Document Databases**: MongoDB, CouchDB
3. **Column-Family Stores**: Cassandra, HBase
4. **Graph Databases**: Neo4j, Amazon Neptune

### Key Differences from SQL:

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| Schema | Fixed, predefined | Flexible, dynamic |
| Scaling | Vertical (bigger server) | Horizontal (more servers) |
| ACID | Full ACID support | Eventual consistency (usually) |
| Relationships | Foreign keys, joins | Embedded documents or references |
| Query Language | SQL | API-based or query languages |
| Use Case | Structured data, transactions | Unstructured data, high throughput |

---

## NoSQL Databases Used in Our Codebase

### Redis (In-Memory Key-Value Store)

**Location:** `product-service/src/main/resources/application.yaml`

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms
```

**Why Redis?**
- **Caching**: Fast access to frequently used data
- **Distributed Locking**: Coordinate across multiple instances
- **Rate Limiting**: Control request rates
- **Session Storage**: Store user sessions
- **Pub/Sub**: Real-time messaging

**Use Cases in Our Codebase:**
1. **Product Caching** (`ProductCacheService`)
2. **Distributed Locking** (`DistributedLockService`)
3. **Rate Limiting** (API Gateway)

---

## Redis - In-Memory Data Store

### What is Redis?

**Redis (Remote Dictionary Server)** is an in-memory data structure store that can be used as:
- Database
- Cache
- Message broker
- Session store

### Key Features:

1. **In-Memory**: Data stored in RAM (extremely fast)
2. **Persistence**: Optional disk persistence (RDB, AOF)
3. **Data Structures**: Strings, Lists, Sets, Hashes, Sorted Sets
4. **Atomic Operations**: All operations are atomic
5. **Pub/Sub**: Publish-subscribe messaging
6. **Transactions**: Multi-command transactions
7. **Lua Scripting**: Server-side scripting

### Redis Data Structures:

#### 1. Strings
```redis
SET user:100:name "John Doe"
GET user:100:name
SET user:100:name "John Doe" EX 3600  # With expiration
```

#### 2. Hashes
```redis
HSET product:100 name "Laptop" price 999.99
HGET product:100 name
HGETALL product:100
```

#### 3. Lists
```redis
LPUSH orders:user:100 "order:1"
RPUSH orders:user:100 "order:2"
LRANGE orders:user:100 0 -1
```

#### 4. Sets
```redis
SADD tags:product:100 "electronics" "laptop"
SMEMBERS tags:product:100
SISMEMBER tags:product:100 "electronics"
```

#### 5. Sorted Sets
```redis
ZADD leaderboard 100 "user:1"
ZADD leaderboard 200 "user:2"
ZRANGE leaderboard 0 -1 WITHSCORES
```

---

## Redis Implementation in Codebase

### 1. Redis Configuration

**File:** `product-service/src/main/java/com/ekart/product_service/config/RedisConfig.java`

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
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new StringRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }
}
```

**Key Points:**
- **LettuceConnectionFactory**: Async, thread-safe connection pool
- **StringRedisSerializer**: Serializes keys/values as strings
- **Two Templates**: One for strings, one for objects

### 2. Redis Caching Service

**File:** `product-service/src/main/java/com/ekart/product_service/cache/ProductCacheService.java`

```java
@Service
public class ProductCacheService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    private static final String CACHE_PREFIX = "product:";
    private static final String PRODUCT_LIST_PREFIX = "products:list:";
    private static final Duration PRODUCT_TTL = Duration.ofMinutes(5);
    private static final Duration PRODUCT_LIST_TTL = Duration.ofMinutes(2);

    // Get product from cache
    public ProductDTO getProduct(Long skuId) {
        String cacheKey = CACHE_PREFIX + skuId;
        String cached = redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            logger.debug("Product found in cache: {}", skuId);
            // Deserialize JSON to ProductDTO
            return deserialize(cached);
        }
        
        return null;
    }

    // Put product in cache
    public void putProduct(Long skuId, ProductDTO product) {
        String cacheKey = CACHE_PREFIX + skuId;
        String json = serialize(product);
        redisTemplate.opsForValue().set(
            cacheKey, 
            json, 
            PRODUCT_TTL.toMinutes(), 
            TimeUnit.MINUTES
        );
    }

    // Invalidate cache
    public void invalidateProduct(Long skuId) {
        String cacheKey = CACHE_PREFIX + skuId;
        redisTemplate.delete(cacheKey);
    }
}
```

**Cache-Aside Pattern:**
1. Check cache first
2. If miss, get from database
3. Store in cache
4. On update, invalidate cache

### 3. Distributed Locking Service

**File:** `product-service/src/main/java/com/ekart/product_service/lock/DistributedLockService.java`

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

    // Acquire lock
    public LockToken acquireLock(String lockKey, Duration timeout) {
        String fullLockKey = LOCK_PREFIX + lockKey;
        String lockValue = UUID.randomUUID().toString();
        
        Boolean acquired = redisTemplate.opsForValue()
                .setIfAbsent(fullLockKey, lockValue, timeout);
        
        if (Boolean.TRUE.equals(acquired)) {
            return new LockToken(fullLockKey, lockValue, timeout);
        }
        return null;
    }

    // Release lock atomically
    public boolean releaseLock(LockToken lockToken) {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>();
        script.setScriptText(UNLOCK_SCRIPT);
        script.setResultType(Long.class);

        Long result = redisTemplate.execute(
                script,
                Collections.singletonList(lockToken.getLockKey()),
                lockToken.getLockValue()
        );

        return result != null && result > 0;
    }

    // Execute with lock
    public <T> T executeWithLock(String lockKey, Duration timeout, 
                                LockedOperation<T> criticalSection) {
        LockToken lockToken = acquireLock(lockKey, timeout);
        if (lockToken == null) {
            throw new LockAcquisitionException("Could not acquire lock: " + lockKey);
        }

        try {
            return criticalSection.execute();
        } finally {
            releaseLock(lockToken);
        }
    }
}
```

**Key Points:**
- **SETNX (SET if Not eXists)**: Atomic lock acquisition
- **Lua Script**: Atomic lock release (only if we own it)
- **Expiration**: Locks expire automatically (prevents deadlocks)
- **UUID**: Unique lock value prevents releasing others' locks

### 4. Redis Operations

#### Basic Operations:

```java
// String operations
redisTemplate.opsForValue().set("key", "value");
redisTemplate.opsForValue().get("key");
redisTemplate.opsForValue().set("key", "value", 60, TimeUnit.SECONDS); // With TTL
redisTemplate.delete("key");

// Hash operations
redisTemplate.opsForHash().put("product:100", "name", "Laptop");
redisTemplate.opsForHash().get("product:100", "name");
redisTemplate.opsForHash().entries("product:100"); // Get all
redisTemplate.opsForHash().delete("product:100", "name");

// List operations
redisTemplate.opsForList().leftPush("orders", "order:1");
redisTemplate.opsForList().rightPush("orders", "order:2");
redisTemplate.opsForList().range("orders", 0, -1);

// Set operations
redisTemplate.opsForSet().add("tags:product:100", "electronics", "laptop");
redisTemplate.opsForSet().members("tags:product:100");
redisTemplate.opsForSet().isMember("tags:product:100", "electronics");

// Sorted Set operations
redisTemplate.opsForZSet().add("leaderboard", "user:1", 100.0);
redisTemplate.opsForZSet().range("leaderboard", 0, -1);
redisTemplate.opsForZSet().rangeWithScores("leaderboard", 0, -1);
```

#### Advanced Operations:

```java
// Check if key exists
Boolean exists = redisTemplate.hasKey("key");

// Set expiration
redisTemplate.expire("key", 60, TimeUnit.SECONDS);

// Increment/Decrement
redisTemplate.opsForValue().increment("counter");
redisTemplate.opsForValue().increment("counter", 5);
redisTemplate.opsForValue().decrement("counter");

// Pattern matching (use SCAN, not KEYS)
Set<String> keys = redisTemplate.keys("product:*"); // ❌ Blocks Redis
// ✅ Use SCAN instead
ScanOptions options = ScanOptions.scanOptions().match("product:*").build();
try (Cursor<String> cursor = redisTemplate.scan(options)) {
    cursor.forEachRemaining(key -> {
        // Process key
    });
}
```

---

## Other NoSQL Databases

### 1. MongoDB (Document Database) - Complete Guide

**What is MongoDB?**
- Document-oriented database
- Stores data as BSON (Binary JSON)
- Flexible schema (no fixed structure)
- Horizontal scaling with sharding
- Rich query language
- Indexing support

**When to Use MongoDB:**
- ✅ Product catalogs with varying attributes
- ✅ User profiles and preferences
- ✅ Content management systems
- ✅ Real-time analytics
- ✅ Logging and event storage
- ✅ Flexible schema requirements
- ✅ Embedded documents (reviews, specifications)

**When NOT to Use MongoDB:**
- ❌ Complex transactions across documents
- ❌ Strong ACID requirements
- ❌ Complex joins (use embedded documents instead)
- ❌ Small datasets (SQL is simpler)

---

#### MongoDB Setup and Configuration

**Step 1: Add Dependencies**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

**Step 2: Configuration**

```yaml
# application.yaml
spring:
  data:
    mongodb:
      host: localhost
      port: 27017
      database: ecommerce
      # Connection string alternative:
      # uri: mongodb://localhost:27017/ecommerce
      # With authentication:
      # uri: mongodb://username:password@localhost:27017/ecommerce
```

**Step 3: MongoDB Configuration Class**

```java
@Configuration
@EnableMongoRepositories(basePackages = "com.ekart.product_service.mongodb")
public class MongoConfig {
    
    @Bean
    public MongoTemplate mongoTemplate(MongoDatabaseFactory factory) {
        return new MongoTemplate(factory);
    }
    
    @Bean
    public MongoCustomConversions customConversions() {
        return new MongoCustomConversions(Arrays.asList(
            new BigDecimalToDecimal128Converter(),
            new Decimal128ToBigDecimalConverter()
        ));
    }
}
```

---

#### Real-World Example: E-Commerce Product Catalog

**Use Case:** Store product catalogs with flexible attributes, embedded reviews, and specifications.

**Why MongoDB?**
- Products have different attributes (electronics vs clothing)
- Reviews are embedded (read together with product)
- Specifications vary by category
- Need flexible schema for new product types

**1. MongoDB Document Structure**

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "skuId": 1001,
  "spuId": 100,
  "sellerId": 50,
  "sellerName": "TechStore Inc",
  "productName": "MacBook Pro 16-inch",
  "displayName": "Apple MacBook Pro 16-inch M2 Pro",
  "description": "Powerful laptop for professionals",
  "brand": "Apple",
  "category": "Electronics",
  "categoryId": 1,
  "skuCode": "MBP-16-M2-512",
  "variantName": "Space Gray, 512GB",
  "attributes": {
    "color": "Space Gray",
    "storage": "512GB SSD",
    "ram": "16GB",
    "processor": "M2 Pro",
    "screen": "16.2-inch Liquid Retina XDR"
  },
  "pricing": {
    "sellingPrice": 2499.99,
    "mrp": 2799.99,
    "currency": "USD"
  },
  "inventory": {
    "availableQuantity": 25,
    "reservedQuantity": 5,
    "warehouseId": 1
  },
  "images": [
    {
      "url": "https://example.com/mbp-front.jpg",
      "type": "front",
      "order": 1
    },
    {
      "url": "https://example.com/mbp-back.jpg",
      "type": "back",
      "order": 2
    }
  ],
  "specifications": {
    "display": {
      "size": "16.2 inches",
      "resolution": "3456 x 2234",
      "technology": "Liquid Retina XDR"
    },
    "processor": {
      "chip": "Apple M2 Pro",
      "cores": 12,
      "speed": "3.5 GHz"
    },
    "memory": {
      "ram": "16GB",
      "type": "Unified Memory"
    },
    "storage": {
      "capacity": "512GB",
      "type": "SSD"
    }
  },
  "reviews": [
    {
      "reviewId": "rev-001",
      "userId": 1001,
      "userName": "John Doe",
      "rating": 5,
      "title": "Excellent laptop!",
      "comment": "Fast performance, great display",
      "verifiedPurchase": true,
      "helpfulCount": 45,
      "createdAt": ISODate("2024-01-15T10:30:00Z")
    },
    {
      "reviewId": "rev-002",
      "userId": 1002,
      "userName": "Jane Smith",
      "rating": 4,
      "title": "Good but expensive",
      "comment": "Works well but pricey",
      "verifiedPurchase": true,
      "helpfulCount": 12,
      "createdAt": ISODate("2024-01-20T14:20:00Z")
    }
  ],
  "tags": ["laptop", "apple", "macbook", "professional", "m2"],
  "rating": {
    "average": 4.5,
    "count": 234,
    "distribution": {
      "5": 150,
      "4": 60,
      "3": 15,
      "2": 5,
      "1": 4
    }
  },
  "isActive": true,
  "createdAt": ISODate("2024-01-01T00:00:00Z"),
  "updatedAt": ISODate("2024-01-25T12:00:00Z")
}
```

**2. Java Entity Classes**

```java
// Product Document
@Document(collection = "products")
@CompoundIndex(name = "seller_category_idx", def = "{'sellerId': 1, 'category': 1}")
@CompoundIndex(name = "price_range_idx", def = "{'pricing.sellingPrice': 1}")
public class ProductDocument {
    
    @Id
    private String id;
    
    private Long skuId;
    private Long spuId;
    private Long sellerId;
    private String sellerName;
    private String productName;
    private String displayName;
    private String description;
    private String brand;
    private String category;
    private Long categoryId;
    private String skuCode;
    private String variantName;
    
    // Embedded document for attributes
    private Map<String, Object> attributes;
    
    // Embedded document for pricing
    @Field("pricing")
    private Pricing pricing;
    
    // Embedded document for inventory
    @Field("inventory")
    private Inventory inventory;
    
    // Array of embedded documents for images
    private List<ProductImage> images;
    
    // Embedded document for specifications
    @Field("specifications")
    private ProductSpecifications specifications;
    
    // Array of embedded documents for reviews
    private List<Review> reviews;
    
    private List<String> tags;
    
    // Embedded document for rating
    @Field("rating")
    private ProductRating rating;
    
    private Boolean isActive;
    
    @CreatedDate
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
    
    // Getters and setters
}

// Embedded Pricing Document
public class Pricing {
    private BigDecimal sellingPrice;
    private BigDecimal mrp;
    private String currency;
    
    // Getters and setters
}

// Embedded Inventory Document
public class Inventory {
    private Integer availableQuantity;
    private Integer reservedQuantity;
    private Long warehouseId;
    
    // Getters and setters
}

// Embedded Product Image Document
public class ProductImage {
    private String url;
    private String type;
    private Integer order;
    
    // Getters and setters
}

// Embedded Specifications Document
public class ProductSpecifications {
    private Map<String, Object> display;
    private Map<String, Object> processor;
    private Map<String, Object> memory;
    private Map<String, Object> storage;
    
    // Getters and setters
}

// Embedded Review Document
public class Review {
    private String reviewId;
    private Long userId;
    private String userName;
    private Integer rating;
    private String title;
    private String comment;
    private Boolean verifiedPurchase;
    private Integer helpfulCount;
    private Instant createdAt;
    
    // Getters and setters
}

// Embedded Rating Document
public class ProductRating {
    private Double average;
    private Integer count;
    private Map<String, Integer> distribution;
    
    // Getters and setters
}
```

**3. Repository Interface**

```java
@Repository
public interface ProductMongoRepository extends MongoRepository<ProductDocument, String> {
    
    // Find by SKU ID
    Optional<ProductDocument> findBySkuId(Long skuId);
    
    // Find by seller
    List<ProductDocument> findBySellerId(Long sellerId);
    
    // Find active products
    List<ProductDocument> findByIsActiveTrue();
    
    // Find by category
    List<ProductDocument> findByCategory(String category);
    
    // Find by tags (array contains)
    List<ProductDocument> findByTagsContaining(String tag);
    
    // Find by multiple tags (all tags present)
    List<ProductDocument> findByTagsContainingAll(List<String> tags);
    
    // Find by price range
    List<ProductDocument> findByPricingSellingPriceBetween(
        BigDecimal minPrice, BigDecimal maxPrice);
    
    // Find by brand
    List<ProductDocument> findByBrand(String brand);
    
    // Find by rating
    List<ProductDocument> findByRatingAverageGreaterThanEqual(Double minRating);
    
    // Find products with verified reviews
    @Query("{ 'reviews.verifiedPurchase': true }")
    List<ProductDocument> findProductsWithVerifiedReviews();
    
    // Find by specification attribute
    @Query("{ 'specifications.processor.chip': ?0 }")
    List<ProductDocument> findByProcessorChip(String chip);
    
    // Find by attribute value
    @Query("{ 'attributes.color': ?0 }")
    List<ProductDocument> findByColor(String color);
    
    // Complex query: Find products by category and price range
    @Query("{ 'category': ?0, 'pricing.sellingPrice': { $gte: ?1, $lte: ?2 } }")
    List<ProductDocument> findByCategoryAndPriceRange(
        String category, BigDecimal minPrice, BigDecimal maxPrice);
    
    // Text search
    @Query("{ $text: { $search: ?0 } }")
    List<ProductDocument> searchProducts(String searchText);
    
    // Pagination
    Page<ProductDocument> findByCategory(String category, Pageable pageable);
    
    // Sort by price
    List<ProductDocument> findByCategoryOrderByPricingSellingPriceAsc(String category);
    
    // Count by seller
    long countBySellerId(Long sellerId);
    
    // Exists check
    boolean existsBySkuId(Long skuId);
}
```

**4. Service Implementation**

```java
@Service
public class ProductMongoService {
    
    @Autowired
    private ProductMongoRepository productRepository;
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    // Create product
    public ProductDocument createProduct(ProductDocument product) {
        product.setCreatedAt(Instant.now());
        product.setUpdatedAt(Instant.now());
        return productRepository.save(product);
    }
    
    // Get product by SKU ID
    public Optional<ProductDocument> getProductBySkuId(Long skuId) {
        return productRepository.findBySkuId(skuId);
    }
    
    // Search products
    public List<ProductDocument> searchProducts(String query) {
        // Text search (requires text index)
        return productRepository.searchProducts(query);
    }
    
    // Find products by category with pagination
    public Page<ProductDocument> getProductsByCategory(
            String category, int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return productRepository.findByCategory(category, pageable);
    }
    
    // Find products by price range
    public List<ProductDocument> getProductsByPriceRange(
            BigDecimal minPrice, BigDecimal maxPrice) {
        return productRepository.findByPricingSellingPriceBetween(minPrice, maxPrice);
    }
    
    // Add review to product
    public void addReview(Long skuId, Review review) {
        ProductDocument product = productRepository.findBySkuId(skuId)
            .orElseThrow(() -> new ProductNotFoundException(skuId));
        
        if (product.getReviews() == null) {
            product.setReviews(new ArrayList<>());
        }
        
        review.setReviewId(UUID.randomUUID().toString());
        review.setCreatedAt(Instant.now());
        product.getReviews().add(review);
        
        // Update rating
        updateRating(product);
        
        product.setUpdatedAt(Instant.now());
        productRepository.save(product);
    }
    
    // Update rating after review
    private void updateRating(ProductDocument product) {
        List<Review> reviews = product.getReviews();
        if (reviews == null || reviews.isEmpty()) {
            return;
        }
        
        double average = reviews.stream()
            .mapToInt(Review::getRating)
            .average()
            .orElse(0.0);
        
        Map<String, Integer> distribution = reviews.stream()
            .collect(Collectors.groupingBy(
                r -> String.valueOf(r.getRating()),
                Collectors.collectingAndThen(Collectors.counting(), Long::intValue)
            ));
        
        ProductRating rating = new ProductRating();
        rating.setAverage(average);
        rating.setCount(reviews.size());
        rating.setDistribution(distribution);
        
        product.setRating(rating);
    }
    
    // Update inventory
    public void updateInventory(Long skuId, Integer quantity) {
        ProductDocument product = productRepository.findBySkuId(skuId)
            .orElseThrow(() -> new ProductNotFoundException(skuId));
        
        if (product.getInventory() == null) {
            product.setInventory(new Inventory());
        }
        
        product.getInventory().setAvailableQuantity(quantity);
        product.setUpdatedAt(Instant.now());
        productRepository.save(product);
    }
    
    // Update product price
    public void updatePrice(Long skuId, BigDecimal newPrice) {
        Update update = new Update();
        update.set("pricing.sellingPrice", newPrice);
        update.set("updatedAt", Instant.now());
        
        Query query = new Query(Criteria.where("skuId").is(skuId));
        mongoTemplate.updateFirst(query, update, ProductDocument.class);
    }
    
    // Bulk update: Update prices for multiple products
    public void bulkUpdatePrices(Map<Long, BigDecimal> priceUpdates) {
        BulkOperations bulkOps = mongoTemplate.bulkOps(
            BulkOperations.BulkMode.ORDERED, ProductDocument.class);
        
        priceUpdates.forEach((skuId, price) -> {
            Query query = new Query(Criteria.where("skuId").is(skuId));
            Update update = new Update()
                .set("pricing.sellingPrice", price)
                .set("updatedAt", Instant.now());
            bulkOps.updateOne(query, update);
        });
        
        bulkOps.execute();
    }
    
    // Aggregation: Get average price by category
    public Map<String, Double> getAveragePriceByCategory() {
        Aggregation aggregation = Aggregation.newAggregation(
            Aggregation.match(Criteria.where("isActive").is(true)),
            Aggregation.group("category")
                .avg("pricing.sellingPrice").as("avgPrice"),
            Aggregation.sort(Sort.Direction.ASC, "avgPrice")
        );
        
        AggregationResults<Map> results = mongoTemplate.aggregate(
            aggregation, ProductDocument.class, Map.class);
        
        return results.getMappedResults().stream()
            .collect(Collectors.toMap(
                m -> (String) m.get("_id"),
                m -> ((Number) m.get("avgPrice")).doubleValue()
            ));
    }
    
    // Aggregation: Get top rated products
    public List<ProductDocument> getTopRatedProducts(int limit) {
        Aggregation aggregation = Aggregation.newAggregation(
            Aggregation.match(Criteria.where("isActive").is(true)),
            Aggregation.sort(Sort.Direction.DESC, "rating.average"),
            Aggregation.limit(limit)
        );
        
        AggregationResults<ProductDocument> results = mongoTemplate.aggregate(
            aggregation, ProductDocument.class, ProductDocument.class);
        
        return results.getMappedResults();
    }
    
    // Find products with specific attribute
    public List<ProductDocument> findProductsByAttribute(
            String attributeKey, Object attributeValue) {
        Query query = new Query();
        query.addCriteria(Criteria.where("attributes." + attributeKey)
            .is(attributeValue));
        return mongoTemplate.find(query, ProductDocument.class);
    }
}
```

**5. Controller Example**

```java
@RestController
@RequestMapping("/api/mongo/products")
public class ProductMongoController {
    
    @Autowired
    private ProductMongoService productService;
    
    @PostMapping
    public ResponseEntity<ProductDocument> createProduct(
            @RequestBody ProductDocument product) {
        ProductDocument created = productService.createProduct(product);
        return ResponseEntity.ok(created);
    }
    
    @GetMapping("/{skuId}")
    public ResponseEntity<ProductDocument> getProduct(@PathVariable Long skuId) {
        return productService.getProductBySkuId(skuId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @GetMapping("/search")
    public ResponseEntity<List<ProductDocument>> searchProducts(
            @RequestParam String q) {
        List<ProductDocument> products = productService.searchProducts(q);
        return ResponseEntity.ok(products);
    }
    
    @GetMapping("/category/{category}")
    public ResponseEntity<Page<ProductDocument>> getProductsByCategory(
            @PathVariable String category,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Page<ProductDocument> products = productService.getProductsByCategory(
            category, page, size);
        return ResponseEntity.ok(products);
    }
    
    @PostMapping("/{skuId}/reviews")
    public ResponseEntity<Void> addReview(
            @PathVariable Long skuId,
            @RequestBody Review review) {
        productService.addReview(skuId, review);
        return ResponseEntity.ok().build();
    }
}
```

**6. Indexes for Performance**

```java
@Configuration
public class MongoIndexConfig {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @PostConstruct
    public void createIndexes() {
        // Text index for search
        Index textIndex = new Index().on("productName", Sort.Direction.ASC)
            .on("description", Sort.Direction.ASC)
            .on("tags", Sort.Direction.ASC)
            .named("product_text_idx")
            .background();
        mongoTemplate.indexOps(ProductDocument.class).ensureIndex(textIndex);
        
        // Compound index for common queries
        Index compoundIndex = new Index().on("sellerId", Sort.Direction.ASC)
            .on("category", Sort.Direction.ASC)
            .on("isActive", Sort.Direction.ASC)
            .named("seller_category_active_idx")
            .background();
        mongoTemplate.indexOps(ProductDocument.class).ensureIndex(compoundIndex);
        
        // Index on price for range queries
        Index priceIndex = new Index().on("pricing.sellingPrice", Sort.Direction.ASC)
            .named("price_idx")
            .background();
        mongoTemplate.indexOps(ProductDocument.class).ensureIndex(priceIndex);
        
        // Index on rating
        Index ratingIndex = new Index().on("rating.average", Sort.Direction.DESC)
            .named("rating_idx")
            .background();
        mongoTemplate.indexOps(ProductDocument.class).ensureIndex(ratingIndex);
    }
}
```

**7. MongoDB Best Practices for This Use Case**

**✅ DO:**
- Embed reviews (read together with product)
- Use compound indexes for common query patterns
- Use text indexes for search
- Set appropriate TTL for temporary data
- Use projections to fetch only needed fields
- Use aggregation pipeline for complex queries

**❌ DON'T:**
- Embed large arrays (limit reviews to recent 50)
- Create too many indexes (slows writes)
- Use deep nesting (max 3-4 levels)
- Store large binary data (use GridFS or external storage)

---

#### MongoDB Benefits for E-Commerce:
- **Flexible Schema**: Add new product attributes without migration
- **Embedded Documents**: Reviews, specifications read together
- **Array Queries**: Search by tags, categories efficiently
- **Horizontal Scaling**: Shard by sellerId or category
- **Rich Queries**: Complex queries with aggregation pipeline

### 2. Cassandra (Column-Family Store) - Complete Guide

**What is Cassandra?**
- Distributed, wide-column store (NoSQL database)
- Designed for high write throughput (millions of writes per second)
- No single point of failure (peer-to-peer architecture)
- Eventually consistent (tunable consistency)
- Linear scalability (add nodes, get more capacity)
- Masterless architecture (all nodes equal)

**When to Use Cassandra:**
- ✅ Time-series data (order events, logs, metrics)
- ✅ High write throughput (millions per second)
- ✅ Global distribution (multi-region)
- ✅ Event logging and audit trails
- ✅ IoT sensor data
- ✅ Real-time analytics
- ✅ Write-heavy workloads

**When NOT to Use Cassandra:**
- ❌ Complex queries with joins
- ❌ Strong consistency requirements
- ❌ Small datasets (overhead not worth it)
- ❌ Frequent updates to existing rows
- ❌ Complex transactions

---

#### Cassandra Setup and Configuration

**Step 1: Add Dependencies**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-cassandra</artifactId>
</dependency>
<dependency>
    <groupId>com.datastax.oss</groupId>
    <artifactId>java-driver-core</artifactId>
</dependency>
```

**Step 2: Configuration**

```yaml
# application.yaml
spring:
  data:
    cassandra:
      keyspace-name: ecommerce
      contact-points: localhost
      port: 9042
      local-datacenter: datacenter1
      # With authentication:
      # username: cassandra
      # password: cassandra
      # SSL:
      # ssl: true
      # Schema action:
      schema-action: CREATE_IF_NOT_EXISTS
```

**Step 3: Cassandra Configuration Class**

```java
@Configuration
@EnableCassandraRepositories(basePackages = "com.ekart.order_service.cassandra")
public class CassandraConfig extends AbstractCassandraConfiguration {
    
    @Value("${spring.data.cassandra.keyspace-name}")
    private String keyspace;
    
    @Value("${spring.data.cassandra.contact-points}")
    private String contactPoints;
    
    @Value("${spring.data.cassandra.port}")
    private int port;
    
    @Value("${spring.data.cassandra.local-datacenter}")
    private String localDatacenter;
    
    @Override
    protected String getKeyspaceName() {
        return keyspace;
    }
    
    @Override
    protected String getContactPoints() {
        return contactPoints;
    }
    
    @Override
    protected int getPort() {
        return port;
    }
    
    @Override
    protected String getLocalDataCenter() {
        return localDatacenter;
    }
    
    @Override
    public SchemaAction getSchemaAction() {
        return SchemaAction.CREATE_IF_NOT_EXISTS;
    }
    
    @Override
    public String[] getEntityBasePackages() {
        return new String[]{"com.ekart.order_service.cassandra.entity"};
    }
}
```

---

#### Real-World Example: Order Event Logging System

**Use Case:** Store all order events (created, paid, shipped, delivered) for audit trail, analytics, and event sourcing.

**Why Cassandra?**
- Millions of order events per day
- Write-heavy workload (append-only)
- Need to query events by order ID
- Time-series data (events ordered by time)
- Global distribution (events from multiple regions)
- No updates (events are immutable)

**1. Cassandra Table Design**

**Key Design Principles:**
- **Partition Key**: Determines which node stores data (order_id)
- **Clustering Key**: Determines sort order within partition (event_time)
- **Query-driven design**: Design tables based on queries you need

**Table 1: Order Events by Order ID**

```sql
CREATE KEYSPACE ecommerce 
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'datacenter1': 3
};

USE ecommerce;

-- Table: Get all events for an order (ordered by time)
CREATE TABLE order_events (
    order_id UUID,
    event_time TIMESTAMP,
    event_type TEXT,
    event_data TEXT,
    user_id BIGINT,
    seller_id BIGINT,
    amount DECIMAL,
    status TEXT,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY (order_id, event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

**Table 2: Order Events by User (for user activity)**

```sql
-- Table: Get all events for a user (ordered by time)
CREATE TABLE user_order_events (
    user_id BIGINT,
    event_time TIMESTAMP,
    order_id UUID,
    event_type TEXT,
    event_data TEXT,
    amount DECIMAL,
    status TEXT,
    PRIMARY KEY (user_id, event_time, order_id)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

**Table 3: Order Events by Seller (for seller dashboard)**

```sql
-- Table: Get all events for a seller (ordered by time)
CREATE TABLE seller_order_events (
    seller_id BIGINT,
    event_time TIMESTAMP,
    order_id UUID,
    event_type TEXT,
    event_data TEXT,
    amount DECIMAL,
    status TEXT,
    PRIMARY KEY (seller_id, event_time, order_id)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

**Table 4: Order Events by Date (for analytics)**

```sql
-- Table: Get events by date (for daily analytics)
CREATE TABLE order_events_by_date (
    event_date DATE,
    event_time TIMESTAMP,
    order_id UUID,
    event_type TEXT,
    user_id BIGINT,
    seller_id BIGINT,
    amount DECIMAL,
    status TEXT,
    PRIMARY KEY (event_date, event_time, order_id)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

**2. Java Entity Classes**

```java
// Order Event Entity
@Table("order_events")
public class OrderEvent {
    
    @PrimaryKey
    private OrderEventKey key;
    
    @Column("event_type")
    private String eventType;
    
    @Column("event_data")
    private String eventData;
    
    @Column("user_id")
    private Long userId;
    
    @Column("seller_id")
    private Long sellerId;
    
    @Column("amount")
    private BigDecimal amount;
    
    @Column("status")
    private String status;
    
    @Column("metadata")
    private Map<String, String> metadata;
    
    @PrimaryKeyClass
    public static class OrderEventKey implements Serializable {
        
        @PrimaryKeyColumn(name = "order_id", ordinal = 0, type = PrimaryKeyType.PARTITIONED)
        private UUID orderId;
        
        @PrimaryKeyColumn(name = "event_time", ordinal = 1, type = PrimaryKeyType.CLUSTERED)
        private Instant eventTime;
        
        // Getters and setters
        public UUID getOrderId() { return orderId; }
        public void setOrderId(UUID orderId) { this.orderId = orderId; }
        
        public Instant getEventTime() { return eventTime; }
        public void setEventTime(Instant eventTime) { this.eventTime = eventTime; }
    }
    
    // Getters and setters
    public OrderEventKey getKey() { return key; }
    public void setKey(OrderEventKey key) { this.key = key; }
    
    public String getEventType() { return eventType; }
    public void setEventType(String eventType) { this.eventType = eventType; }
    
    // ... other getters and setters
}

// User Order Event Entity
@Table("user_order_events")
public class UserOrderEvent {
    
    @PrimaryKey
    private UserOrderEventKey key;
    
    @Column("event_type")
    private String eventType;
    
    @Column("event_data")
    private String eventData;
    
    @Column("amount")
    private BigDecimal amount;
    
    @Column("status")
    private String status;
    
    @PrimaryKeyClass
    public static class UserOrderEventKey implements Serializable {
        
        @PrimaryKeyColumn(name = "user_id", ordinal = 0, type = PrimaryKeyType.PARTITIONED)
        private Long userId;
        
        @PrimaryKeyColumn(name = "event_time", ordinal = 1, type = PrimaryKeyType.CLUSTERED)
        private Instant eventTime;
        
        @PrimaryKeyColumn(name = "order_id", ordinal = 2, type = PrimaryKeyType.CLUSTERED)
        private UUID orderId;
        
        // Getters and setters
    }
    
    // Getters and setters
}

// Seller Order Event Entity
@Table("seller_order_events")
public class SellerOrderEvent {
    
    @PrimaryKey
    private SellerOrderEventKey key;
    
    @Column("event_type")
    private String eventType;
    
    @Column("event_data")
    private String eventData;
    
    @Column("amount")
    private BigDecimal amount;
    
    @Column("status")
    private String status;
    
    @PrimaryKeyClass
    public static class SellerOrderEventKey implements Serializable {
        
        @PrimaryKeyColumn(name = "seller_id", ordinal = 0, type = PrimaryKeyType.PARTITIONED)
        private Long sellerId;
        
        @PrimaryKeyColumn(name = "event_time", ordinal = 1, type = PrimaryKeyType.CLUSTERED)
        private Instant eventTime;
        
        @PrimaryKeyColumn(name = "order_id", ordinal = 2, type = PrimaryKeyType.CLUSTERED)
        private UUID orderId;
        
        // Getters and setters
    }
    
    // Getters and setters
}

// Date-based Order Event Entity
@Table("order_events_by_date")
public class OrderEventByDate {
    
    @PrimaryKey
    private OrderEventByDateKey key;
    
    @Column("event_type")
    private String eventType;
    
    @Column("user_id")
    private Long userId;
    
    @Column("seller_id")
    private Long sellerId;
    
    @Column("amount")
    private BigDecimal amount;
    
    @Column("status")
    private String status;
    
    @PrimaryKeyClass
    public static class OrderEventByDateKey implements Serializable {
        
        @PrimaryKeyColumn(name = "event_date", ordinal = 0, type = PrimaryKeyType.PARTITIONED)
        private LocalDate eventDate;
        
        @PrimaryKeyColumn(name = "event_time", ordinal = 1, type = PrimaryKeyType.CLUSTERED)
        private Instant eventTime;
        
        @PrimaryKeyColumn(name = "order_id", ordinal = 2, type = PrimaryKeyType.CLUSTERED)
        private UUID orderId;
        
        // Getters and setters
    }
    
    // Getters and setters
}
```

**3. Repository Interfaces**

```java
// Order Events Repository
@Repository
public interface OrderEventRepository extends CassandraRepository<OrderEvent, OrderEvent.OrderEventKey> {
    
    // Find all events for an order (ordered by time DESC)
    List<OrderEvent> findByKeyOrderIdOrderByKeyEventTimeDesc(UUID orderId);
    
    // Find events by order and event type
    @Query("SELECT * FROM order_events WHERE order_id = ?0 AND event_type = ?1")
    List<OrderEvent> findByOrderIdAndEventType(UUID orderId, String eventType);
    
    // Find events in time range for an order
    @Query("SELECT * FROM order_events WHERE order_id = ?0 AND event_time >= ?1 AND event_time <= ?2")
    List<OrderEvent> findByOrderIdAndTimeRange(UUID orderId, Instant startTime, Instant endTime);
    
    // Count events for an order
    long countByKeyOrderId(UUID orderId);
}

// User Order Events Repository
@Repository
public interface UserOrderEventRepository extends CassandraRepository<UserOrderEvent, UserOrderEvent.UserOrderEventKey> {
    
    // Find all events for a user (ordered by time DESC)
    List<UserOrderEvent> findByKeyUserIdOrderByKeyEventTimeDesc(Long userId);
    
    // Find events in time range for a user
    @Query("SELECT * FROM user_order_events WHERE user_id = ?0 AND event_time >= ?1 AND event_time <= ?2")
    List<UserOrderEvent> findByUserIdAndTimeRange(Long userId, Instant startTime, Instant endTime);
    
    // Find recent events for a user (with limit)
    @Query("SELECT * FROM user_order_events WHERE user_id = ?0 LIMIT ?1")
    List<UserOrderEvent> findRecentEventsByUserId(Long userId, int limit);
}

// Seller Order Events Repository
@Repository
public interface SellerOrderEventRepository extends CassandraRepository<SellerOrderEvent, SellerOrderEvent.SellerOrderEventKey> {
    
    // Find all events for a seller (ordered by time DESC)
    List<SellerOrderEvent> findByKeySellerIdOrderByKeyEventTimeDesc(Long sellerId);
    
    // Find events in time range for a seller
    @Query("SELECT * FROM seller_order_events WHERE seller_id = ?0 AND event_time >= ?1 AND event_time <= ?2")
    List<SellerOrderEvent> findBySellerIdAndTimeRange(Long sellerId, Instant startTime, Instant endTime);
}

// Date-based Order Events Repository
@Repository
public interface OrderEventByDateRepository extends CassandraRepository<OrderEventByDate, OrderEventByDate.OrderEventByDateKey> {
    
    // Find events for a specific date
    List<OrderEventByDate> findByKeyEventDateOrderByKeyEventTimeDesc(LocalDate eventDate);
    
    // Find events in date range
    @Query("SELECT * FROM order_events_by_date WHERE event_date >= ?0 AND event_date <= ?1")
    List<OrderEventByDate> findByDateRange(LocalDate startDate, LocalDate endDate);
}
```

**4. Service Implementation**

```java
@Service
public class OrderEventService {
    
    @Autowired
    private OrderEventRepository orderEventRepository;
    
    @Autowired
    private UserOrderEventRepository userOrderEventRepository;
    
    @Autowired
    private SellerOrderEventRepository sellerOrderEventRepository;
    
    @Autowired
    private OrderEventByDateRepository orderEventByDateRepository;
    
    @Autowired
    private CqlTemplate cqlTemplate;
    
    // Create order event (write to all relevant tables)
    @Transactional
    public void createOrderEvent(OrderEventDTO eventDTO) {
        Instant eventTime = Instant.now();
        UUID orderId = eventDTO.getOrderId();
        
        // 1. Create main order event
        OrderEvent orderEvent = new OrderEvent();
        OrderEvent.OrderEventKey key = new OrderEvent.OrderEventKey();
        key.setOrderId(orderId);
        key.setEventTime(eventTime);
        orderEvent.setKey(key);
        orderEvent.setEventType(eventDTO.getEventType());
        orderEvent.setEventData(eventDTO.getEventData());
        orderEvent.setUserId(eventDTO.getUserId());
        orderEvent.setSellerId(eventDTO.getSellerId());
        orderEvent.setAmount(eventDTO.getAmount());
        orderEvent.setStatus(eventDTO.getStatus());
        orderEvent.setMetadata(eventDTO.getMetadata());
        orderEventRepository.save(orderEvent);
        
        // 2. Create user order event
        UserOrderEvent userEvent = new UserOrderEvent();
        UserOrderEvent.UserOrderEventKey userKey = new UserOrderEvent.UserOrderEventKey();
        userKey.setUserId(eventDTO.getUserId());
        userKey.setEventTime(eventTime);
        userKey.setOrderId(orderId);
        userEvent.setKey(userKey);
        userEvent.setEventType(eventDTO.getEventType());
        userEvent.setEventData(eventDTO.getEventData());
        userEvent.setAmount(eventDTO.getAmount());
        userEvent.setStatus(eventDTO.getStatus());
        userOrderEventRepository.save(userEvent);
        
        // 3. Create seller order event
        SellerOrderEvent sellerEvent = new SellerOrderEvent();
        SellerOrderEvent.SellerOrderEventKey sellerKey = new SellerOrderEvent.SellerOrderEventKey();
        sellerKey.setSellerId(eventDTO.getSellerId());
        sellerKey.setEventTime(eventTime);
        sellerKey.setOrderId(orderId);
        sellerEvent.setKey(sellerKey);
        sellerEvent.setEventType(eventDTO.getEventType());
        sellerEvent.setEventData(eventDTO.getEventData());
        sellerEvent.setAmount(eventDTO.getAmount());
        sellerEvent.setStatus(eventDTO.getStatus());
        sellerOrderEventRepository.save(sellerEvent);
        
        // 4. Create date-based event
        OrderEventByDate dateEvent = new OrderEventByDate();
        OrderEventByDate.OrderEventByDateKey dateKey = new OrderEventByDate.OrderEventByDateKey();
        dateKey.setEventDate(LocalDate.now());
        dateKey.setEventTime(eventTime);
        dateKey.setOrderId(orderId);
        dateEvent.setKey(dateKey);
        dateEvent.setEventType(eventDTO.getEventType());
        dateEvent.setUserId(eventDTO.getUserId());
        dateEvent.setSellerId(eventDTO.getSellerId());
        dateEvent.setAmount(eventDTO.getAmount());
        dateEvent.setStatus(eventDTO.getStatus());
        orderEventByDateRepository.save(dateEvent);
    }
    
    // Get all events for an order
    public List<OrderEvent> getOrderEvents(UUID orderId) {
        return orderEventRepository.findByKeyOrderIdOrderByKeyEventTimeDesc(orderId);
    }
    
    // Get events for a user
    public List<UserOrderEvent> getUserOrderEvents(Long userId) {
        return userOrderEventRepository.findByKeyUserIdOrderByKeyEventTimeDesc(userId);
    }
    
    // Get events for a seller
    public List<SellerOrderEvent> getSellerOrderEvents(Long sellerId) {
        return sellerOrderEventRepository.findByKeySellerIdOrderByKeyEventTimeDesc(sellerId);
    }
    
    // Get events for a specific date
    public List<OrderEventByDate> getEventsByDate(LocalDate date) {
        return orderEventByDateRepository.findByKeyEventDateOrderByKeyEventTimeDesc(date);
    }
    
    // Bulk insert events (for high throughput)
    public void bulkInsertEvents(List<OrderEventDTO> events) {
        List<OrderEvent> orderEvents = new ArrayList<>();
        List<UserOrderEvent> userEvents = new ArrayList<>();
        List<SellerOrderEvent> sellerEvents = new ArrayList<>();
        List<OrderEventByDate> dateEvents = new ArrayList<>();
        
        Instant now = Instant.now();
        LocalDate today = LocalDate.now();
        
        for (OrderEventDTO eventDTO : events) {
            UUID orderId = eventDTO.getOrderId();
            
            // Create order event
            OrderEvent orderEvent = createOrderEventEntity(eventDTO, now);
            orderEvents.add(orderEvent);
            
            // Create user event
            UserOrderEvent userEvent = createUserEventEntity(eventDTO, now);
            userEvents.add(userEvent);
            
            // Create seller event
            SellerOrderEvent sellerEvent = createSellerEventEntity(eventDTO, now);
            sellerEvents.add(sellerEvent);
            
            // Create date event
            OrderEventByDate dateEvent = createDateEventEntity(eventDTO, now, today);
            dateEvents.add(dateEvent);
        }
        
        // Batch insert
        orderEventRepository.saveAll(orderEvents);
        userOrderEventRepository.saveAll(userEvents);
        sellerOrderEventRepository.saveAll(sellerEvents);
        orderEventByDateRepository.saveAll(dateEvents);
    }
    
    // Analytics: Get daily order count
    public long getDailyOrderCount(LocalDate date) {
        List<OrderEventByDate> events = orderEventByDateRepository
            .findByKeyEventDateOrderByKeyEventTimeDesc(date);
        
        return events.stream()
            .filter(e -> "ORDER_CREATED".equals(e.getEventType()))
            .count();
    }
    
    // Analytics: Get daily revenue
    public BigDecimal getDailyRevenue(LocalDate date) {
        List<OrderEventByDate> events = orderEventByDateRepository
            .findByKeyEventDateOrderByKeyEventTimeDesc(date);
        
        return events.stream()
            .filter(e -> "ORDER_PAID".equals(e.getEventType()))
            .map(OrderEventByDate::getAmount)
            .filter(Objects::nonNull)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    // Analytics: Get events by type for a date range
    public Map<String, Long> getEventCountsByType(LocalDate startDate, LocalDate endDate) {
        List<OrderEventByDate> events = orderEventByDateRepository
            .findByDateRange(startDate, endDate);
        
        return events.stream()
            .collect(Collectors.groupingBy(
                OrderEventByDate::getEventType,
                Collectors.counting()
            ));
    }
    
    // Helper methods
    private OrderEvent createOrderEventEntity(OrderEventDTO dto, Instant eventTime) {
        OrderEvent event = new OrderEvent();
        OrderEvent.OrderEventKey key = new OrderEvent.OrderEventKey();
        key.setOrderId(dto.getOrderId());
        key.setEventTime(eventTime);
        event.setKey(key);
        event.setEventType(dto.getEventType());
        event.setEventData(dto.getEventData());
        event.setUserId(dto.getUserId());
        event.setSellerId(dto.getSellerId());
        event.setAmount(dto.getAmount());
        event.setStatus(dto.getStatus());
        event.setMetadata(dto.getMetadata());
        return event;
    }
    
    // ... other helper methods
}
```

**5. Controller Example**

```java
@RestController
@RequestMapping("/api/cassandra/order-events")
public class OrderEventController {
    
    @Autowired
    private OrderEventService orderEventService;
    
    @PostMapping
    public ResponseEntity<Void> createOrderEvent(@RequestBody OrderEventDTO eventDTO) {
        orderEventService.createOrderEvent(eventDTO);
        return ResponseEntity.ok().build();
    }
    
    @PostMapping("/bulk")
    public ResponseEntity<Void> bulkCreateOrderEvents(@RequestBody List<OrderEventDTO> events) {
        orderEventService.bulkInsertEvents(events);
        return ResponseEntity.ok().build();
    }
    
    @GetMapping("/order/{orderId}")
    public ResponseEntity<List<OrderEvent>> getOrderEvents(@PathVariable UUID orderId) {
        List<OrderEvent> events = orderEventService.getOrderEvents(orderId);
        return ResponseEntity.ok(events);
    }
    
    @GetMapping("/user/{userId}")
    public ResponseEntity<List<UserOrderEvent>> getUserEvents(@PathVariable Long userId) {
        List<UserOrderEvent> events = orderEventService.getUserOrderEvents(userId);
        return ResponseEntity.ok(events);
    }
    
    @GetMapping("/seller/{sellerId}")
    public ResponseEntity<List<SellerOrderEvent>> getSellerEvents(@PathVariable Long sellerId) {
        List<SellerOrderEvent> events = orderEventService.getSellerOrderEvents(sellerId);
        return ResponseEntity.ok(events);
    }
    
    @GetMapping("/analytics/daily/{date}")
    public ResponseEntity<Map<String, Object>> getDailyAnalytics(@PathVariable LocalDate date) {
        long orderCount = orderEventService.getDailyOrderCount(date);
        BigDecimal revenue = orderEventService.getDailyRevenue(date);
        
        Map<String, Object> analytics = new HashMap<>();
        analytics.put("date", date);
        analytics.put("orderCount", orderCount);
        analytics.put("revenue", revenue);
        
        return ResponseEntity.ok(analytics);
    }
}
```

**6. Cassandra Best Practices**

**✅ DO:**
- Design tables based on queries (query-driven design)
- Use appropriate partition keys (avoid hot partitions)
- Use clustering keys for sorting
- Denormalize data (duplicate across tables)
- Use batch inserts for related writes
- Set appropriate TTL for time-series data
- Use materialized views for additional query patterns

**❌ DON'T:**
- Use too many partitions per query (limit to single partition when possible)
- Create hot partitions (all data in one partition)
- Use secondary indexes on high-cardinality columns
- Update existing rows frequently (Cassandra is write-optimized)
- Use joins (denormalize instead)
- Store large blobs (use external storage)

**7. Partition Key Design**

**Good Partition Keys:**
- `order_id` - Each order has its own partition
- `user_id` - Each user has their own partition
- `seller_id` - Each seller has their own partition
- `event_date` - Each day has its own partition

**Bad Partition Keys:**
- `event_type` - Too few partitions (hot partition)
- `status` - Low cardinality (few values)
- `created_at` - High cardinality but time-based (use date instead)

---

#### Cassandra Benefits for Event Logging:
- **High Write Throughput**: Millions of events per second
- **Linear Scalability**: Add nodes, get more capacity
- **No Single Point of Failure**: All nodes equal
- **Time-Series Queries**: Efficient time-ordered queries
- **Global Distribution**: Multi-region support
- **Immutable Data**: Perfect for event sourcing

---

#### MongoDB vs Cassandra: When to Use Which?

| Aspect | MongoDB | Cassandra |
|--------|---------|-----------|
| **Data Model** | Document (JSON-like) | Wide-column (key-value with columns) |
| **Schema** | Flexible, dynamic | Flexible but query-driven |
| **Write Pattern** | General purpose | Write-optimized (append-heavy) |
| **Read Pattern** | Rich queries, aggregations | Fast reads by partition key |
| **Consistency** | Strong (configurable) | Eventually consistent (tunable) |
| **Transactions** | Multi-document transactions | Single partition transactions |
| **Query Language** | Rich query language | CQL (limited) |
| **Joins** | No joins (use embedded docs) | No joins (denormalize) |
| **Best For** | Product catalogs, user profiles | Event logs, time-series, IoT |
| **Use Case** | Flexible schema, complex queries | High write throughput, time-series |

**Decision Matrix:**

**Use MongoDB when:**
- Need flexible schema (product catalogs with varying attributes)
- Need rich queries and aggregations
- Need embedded documents (reviews, specifications)
- Read-heavy workloads
- Complex data structures

**Use Cassandra when:**
- Need high write throughput (millions per second)
- Time-series data (events, logs, metrics)
- Append-only data (immutable events)
- Global distribution (multi-region)
- Simple queries by partition key

**Example: E-Commerce System**

```
SQL (H2/PostgreSQL):
  - Orders, Payments, Users (ACID requirements)
  
MongoDB:
  - Product Catalogs (flexible attributes)
  - User Profiles (preferences, settings)
  - Product Reviews (embedded documents)
  
Cassandra:
  - Order Events (audit trail)
  - User Activity Logs
  - Analytics Events
  - Click Streams
  
Redis:
  - Product Cache
  - Session Storage
  - Rate Limiting
  - Distributed Locks
```

---

#### Migration Strategy: SQL to NoSQL

**Scenario:** Migrate product catalog from SQL to MongoDB

**Step 1: Dual Write Pattern**

```java
@Service
public class ProductMigrationService {
    
    @Autowired
    private ProductRepository sqlRepository; // Existing SQL
    
    @Autowired
    private ProductMongoRepository mongoRepository; // New MongoDB
    
    @Transactional
    public void createProduct(ProductDTO productDTO) {
        // 1. Write to SQL (existing system)
        Product sqlProduct = convertToSQLEntity(productDTO);
        sqlRepository.save(sqlProduct);
        
        // 2. Write to MongoDB (new system)
        ProductDocument mongoProduct = convertToMongoDocument(productDTO);
        mongoRepository.save(mongoProduct);
        
        // 3. Handle failures
        // If MongoDB fails, log and retry later
        // If SQL fails, rollback MongoDB
    }
}
```

**Step 2: Data Migration Script**

```java
@Component
public class ProductDataMigrator {
    
    @Autowired
    private ProductRepository sqlRepository;
    
    @Autowired
    private ProductMongoRepository mongoRepository;
    
    public void migrateProducts() {
        int batchSize = 100;
        int page = 0;
        Page<Product> products;
        
        do {
            Pageable pageable = PageRequest.of(page, batchSize);
            products = sqlRepository.findAll(pageable);
            
            List<ProductDocument> mongoProducts = products.getContent().stream()
                .map(this::convertToMongoDocument)
                .collect(Collectors.toList());
            
            mongoRepository.saveAll(mongoProducts);
            
            page++;
        } while (products.hasNext());
    }
    
    private ProductDocument convertToMongoDocument(Product sqlProduct) {
        ProductDocument mongoProduct = new ProductDocument();
        mongoProduct.setSkuId(sqlProduct.getId());
        mongoProduct.setProductName(sqlProduct.getName());
        // ... map other fields
        
        // Transform relationships to embedded documents
        if (sqlProduct.getReviews() != null) {
            List<Review> reviews = sqlProduct.getReviews().stream()
                .map(this::convertReview)
                .collect(Collectors.toList());
            mongoProduct.setReviews(reviews);
        }
        
        return mongoProduct;
    }
}
```

**Step 3: Gradual Cutover**

```java
@Service
public class ProductService {
    
    @Value("${app.mongodb.enabled:false}")
    private boolean mongoEnabled;
    
    @Autowired
    private ProductRepository sqlRepository;
    
    @Autowired
    private ProductMongoRepository mongoRepository;
    
    public ProductDTO getProduct(Long skuId) {
        if (mongoEnabled) {
            // Use MongoDB
            return mongoRepository.findBySkuId(skuId)
                .map(this::convertToDTO)
                .orElse(null);
        } else {
            // Use SQL
            return sqlRepository.findById(skuId)
                .map(this::convertToDTO)
                .orElse(null);
        }
    }
}
```

---

#### Integration Pattern: Hybrid Approach

**Use Case:** E-commerce system using multiple databases

```java
@Service
public class ProductService {
    
    // SQL for transactional data
    @Autowired
    private ProductRepository sqlRepository;
    
    // MongoDB for product catalog
    @Autowired
    private ProductMongoRepository mongoRepository;
    
    // Redis for caching
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    // Cassandra for events
    @Autowired
    private OrderEventRepository eventRepository;
    
    public ProductDTO getProduct(Long skuId) {
        // 1. Check Redis cache
        String cacheKey = "product:" + skuId;
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return deserialize(cached);
        }
        
        // 2. Get from MongoDB (product catalog)
        ProductDocument mongoProduct = mongoRepository.findBySkuId(skuId)
            .orElseThrow(() -> new ProductNotFoundException(skuId));
        
        // 3. Get inventory from SQL (real-time data)
        Inventory inventory = sqlRepository.findInventoryBySkuId(skuId);
        
        // 4. Combine data
        ProductDTO product = convertToDTO(mongoProduct, inventory);
        
        // 5. Cache in Redis
        redisTemplate.opsForValue().set(cacheKey, serialize(product), 5, TimeUnit.MINUTES);
        
        return product;
    }
    
    @Transactional
    public void createOrder(OrderDTO orderDTO) {
        // 1. Create order in SQL (transactional)
        Order order = sqlRepository.save(convertToEntity(orderDTO));
        
        // 2. Log event in Cassandra (async, non-blocking)
        CompletableFuture.runAsync(() -> {
            OrderEvent event = createOrderEvent(order);
            eventRepository.save(event);
        });
        
        // 3. Invalidate cache in Redis
        redisTemplate.delete("product:" + orderDTO.getSkuId());
    }
}
```

---

### 3. DynamoDB (AWS Managed NoSQL)

**What is DynamoDB?**
- AWS managed NoSQL database
- Serverless, auto-scaling
- Single-digit millisecond latency
- Built-in security and backup

**When to Use:**
- Serverless applications
- High-traffic applications
- AWS ecosystem
- Mobile applications
- Gaming applications

**Example Use Case: User Sessions**

```java
// DynamoDB Table Structure
{
  "TableName": "user_sessions",
  "KeySchema": [
    {
      "AttributeName": "sessionId",
      "KeyType": "HASH"
    }
  ],
  "AttributeDefinitions": [
    {
      "AttributeName": "sessionId",
      "AttributeType": "S"
    }
  ]
}

// Java SDK Example
AmazonDynamoDB client = AmazonDynamoDBClientBuilder.standard().build();
DynamoDBMapper mapper = new DynamoDBMapper(client);

@DynamoDBTable(tableName = "user_sessions")
public class UserSession {
    @DynamoDBHashKey
    private String sessionId;
    
    @DynamoDBAttribute
    private Long userId;
    
    @DynamoDBAttribute
    private Instant expiresAt;
}
```

**Benefits:**
- Fully managed (no server management)
- Auto-scaling
- Pay-per-use pricing
- Global tables for multi-region

### 4. Neo4j (Graph Database)

**What is Neo4j?**
- Graph database
- Stores nodes and relationships
- Optimized for graph queries
- Cypher query language

**When to Use:**
- Social networks
- Recommendation engines
- Fraud detection
- Knowledge graphs
- Network analysis

**Example Use Case: Product Recommendations**

```cypher
// Cypher Query: Find products frequently bought together
MATCH (p1:Product)-[:BOUGHT_WITH]->(p2:Product)
WHERE p1.id = 100
RETURN p2, count(*) as frequency
ORDER BY frequency DESC
LIMIT 10
```

**Real-World Example: Recommendation Engine**

```java
@Node("Product")
public class Product {
    @Id
    private Long id;
    private String name;
    
    @Relationship(type = "BOUGHT_WITH", direction = Relationship.Direction.OUTGOING)
    private List<Product> frequentlyBoughtWith;
}

@Repository
public interface ProductRepository extends Neo4jRepository<Product, Long> {
    @Query("MATCH (p1:Product)-[:BOUGHT_WITH]->(p2:Product) " +
           "WHERE p1.id = $productId " +
           "RETURN p2 ORDER BY p2.frequency DESC LIMIT $limit")
    List<Product> findRecommendations(Long productId, int limit);
}
```

**Benefits:**
- Efficient graph traversals
- Relationship queries
- Pattern matching
- Recommendation algorithms

---

## When to Use NoSQL

### ✅ Use NoSQL When:

1. **High Write Throughput**: Millions of writes per second (Cassandra)
2. **Flexible Schema**: Schema changes frequently (MongoDB)
3. **Horizontal Scaling**: Need to scale across many servers
4. **Unstructured Data**: JSON, documents, graphs
5. **Caching**: Fast access to frequently used data (Redis)
6. **Real-Time Analytics**: Time-series data, events
7. **Session Storage**: User sessions, temporary data
8. **Graph Data**: Social networks, recommendations (Neo4j)

### ❌ Don't Use NoSQL When:

1. **ACID Requirements**: Need strict transactions
2. **Complex Relationships**: Many foreign keys and joins
3. **Structured Data**: Well-defined, stable schema
4. **Complex Queries**: Aggregations, subqueries, joins
5. **Small Dataset**: SQL is simpler for small data
6. **Reporting**: Complex analytics and reporting

### Hybrid Approach:

**Use Both SQL and NoSQL:**
- **SQL**: Core business data (orders, users, products)
- **Redis**: Caching, sessions, rate limiting
- **MongoDB**: Product catalogs, user preferences
- **Cassandra**: Event logs, analytics
- **Neo4j**: Recommendations, social graphs

---

## Best Practices

### 1. Redis Best Practices

**✅ DO:**
- Use appropriate data structures (Hash for objects, Set for unique items)
- Set TTL on all keys (prevent memory leaks)
- Use SCAN instead of KEYS (non-blocking)
- Use connection pooling
- Handle Redis failures gracefully
- Use Lua scripts for atomic operations
- Monitor memory usage

**❌ DON'T:**
- Store large objects (use database instead)
- Use KEYS in production (blocks Redis)
- Forget to set TTL
- Store sensitive data without encryption
- Use Redis as primary database (unless designed for it)

### 2. Key Naming Conventions

**✅ GOOD:**
```
user:100:profile
product:200:details
order:300:items
session:abc123
lock:inventory:500
```

**❌ BAD:**
```
user100profile
product_200_details
order-300-items
```

### 3. TTL Strategy

```java
// Short TTL for volatile data
redisTemplate.opsForValue().set("session:abc", data, 30, TimeUnit.MINUTES);

// Medium TTL for semi-static data
redisTemplate.opsForValue().set("product:100", data, 5, TimeUnit.HOURS);

// Long TTL for static data
redisTemplate.opsForValue().set("config:app", data, 24, TimeUnit.HOURS);
```

### 4. Error Handling

```java
public ProductDTO getProduct(Long skuId) {
    try {
        // Try cache first
        ProductDTO cached = cacheService.getProduct(skuId);
        if (cached != null) {
            return cached;
        }
        
        // Fallback to database
        return productRepository.findById(skuId);
    } catch (RedisConnectionFailureException e) {
        // Redis is down, use database
        logger.warn("Redis unavailable, using database", e);
        return productRepository.findById(skuId);
    }
}
```

### 5. Memory Management

```java
// Monitor memory usage
INFO memory

// Set max memory
CONFIG SET maxmemory 2gb

// Eviction policy
CONFIG SET maxmemory-policy allkeys-lru
```

---

## Real-Time Examples

### Example 1: Cache-Aside Pattern

**Scenario:** Product details are frequently accessed but don't change often.

```java
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ProductCacheService cacheService;
    
    public ProductDTO getProduct(Long skuId) {
        // 1. Check cache
        ProductDTO cached = cacheService.getProduct(skuId);
        if (cached != null) {
            return cached; // Cache hit
        }
        
        // 2. Cache miss - get from database
        Product product = productRepository.findById(skuId)
            .orElseThrow(() -> new ProductNotFoundException(skuId));
        
        ProductDTO dto = convertToDTO(product);
        
        // 3. Store in cache
        cacheService.putProduct(skuId, dto);
        
        return dto;
    }
    
    @Transactional
    public ProductDTO updateProduct(Long skuId, ProductDTO updated) {
        // 1. Update database
        Product product = productRepository.findById(skuId)
            .orElseThrow(() -> new ProductNotFoundException(skuId));
        
        product.setName(updated.getName());
        product.setPrice(updated.getPrice());
        productRepository.save(product);
        
        // 2. Invalidate cache
        cacheService.invalidateProduct(skuId);
        
        return convertToDTO(product);
    }
}
```

**Flow:**
1. Request → Check Redis → Cache Hit → Return
2. Request → Check Redis → Cache Miss → Database → Cache → Return
3. Update → Database → Invalidate Cache

### Example 2: Distributed Locking for Inventory

**Scenario:** Multiple services updating inventory simultaneously.

```java
@Service
public class InventoryService {
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @Autowired
    private DistributedLockService lockService;
    
    public void updateInventory(Long skuId, Integer quantity) {
        String lockKey = "inventory:" + skuId;
        
        lockService.executeWithLock(lockKey, Duration.ofSeconds(30), () -> {
            Inventory inventory = inventoryRepository.findBySkuId(skuId)
                .orElseThrow(() -> new InventoryNotFoundException(skuId));
            
            // Critical section - only one thread can execute
            if (inventory.getAvailableQuantity() >= quantity) {
                inventory.setAvailableQuantity(
                    inventory.getAvailableQuantity() - quantity
                );
                inventoryRepository.save(inventory);
            } else {
                throw new InsufficientInventoryException(skuId);
            }
            
            return null;
        });
    }
}
```

**Flow:**
1. Service A acquires lock → Updates inventory → Releases lock
2. Service B tries to acquire lock → Waits → Acquires → Updates → Releases

### Example 3: Rate Limiting with Redis

**Scenario:** Limit API requests per user/IP.

```java
@Service
public class RateLimitingService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public boolean isAllowed(String key, int maxRequests, Duration window) {
        String redisKey = "ratelimit:" + key;
        
        // Get current count
        String countStr = redisTemplate.opsForValue().get(redisKey);
        int count = countStr != null ? Integer.parseInt(countStr) : 0;
        
        if (count >= maxRequests) {
            return false; // Rate limit exceeded
        }
        
        // Increment count
        if (count == 0) {
            // First request - set with expiration
            redisTemplate.opsForValue().set(
                redisKey, 
                "1", 
                window.toSeconds(), 
                TimeUnit.SECONDS
            );
        } else {
            // Increment existing
            redisTemplate.opsForValue().increment(redisKey);
        }
        
        return true; // Allowed
    }
}
```

**Flow:**
1. Request → Check count → Count < limit → Increment → Allow
2. Request → Check count → Count >= limit → Deny

---

## Strategies and Patterns

### 1. Cache-Aside Pattern

**Purpose:** Application manages cache

**Flow:**
1. Check cache
2. If miss, get from database
3. Store in cache
4. On update, invalidate cache

**Use When:** Read-heavy workloads, data doesn't change frequently

### 2. Write-Through Pattern

**Purpose:** Write to cache and database simultaneously

**Flow:**
1. Write to cache
2. Write to database
3. Both must succeed

**Use When:** Need cache and database always in sync

### 3. Write-Behind Pattern

**Purpose:** Write to cache immediately, database asynchronously

**Flow:**
1. Write to cache
2. Return success
3. Write to database asynchronously

**Use When:** High write throughput, can tolerate eventual consistency

### 4. Read-Through Pattern

**Purpose:** Cache automatically loads from database on miss

**Flow:**
1. Check cache
2. If miss, cache loads from database
3. Return to application

**Use When:** Transparent caching, application doesn't manage cache

### 5. Distributed Locking Pattern

**Purpose:** Coordinate across multiple instances

**Flow:**
1. Acquire lock (SETNX)
2. Execute critical section
3. Release lock (Lua script)

**Use When:** Need mutual exclusion in distributed system

### 6. Pub/Sub Pattern

**Purpose:** Real-time messaging between services

**Flow:**
1. Publisher sends message to channel
2. Subscribers receive message
3. Process message

**Use When:** Event-driven architecture, real-time updates

---

## Interview Questions

### Q1: What is Redis and why use it?

**Answer:**
"Redis is an in-memory data structure store that can be used as a database, cache, or message broker. I use it for caching because it's extremely fast (sub-millisecond latency) and supports various data structures like strings, hashes, lists, sets, and sorted sets. In our system, I use Redis for product caching, distributed locking, and rate limiting. It reduces database load by 80% and improves response times from 100ms to 1ms for cached data."

### Q2: Explain Redis data structures and when to use each.

**Answer:**
"Redis supports multiple data structures:
- **Strings**: Simple key-value, use for caching objects, counters
- **Hashes**: Key-value pairs within a key, use for objects with multiple fields (product details)
- **Lists**: Ordered collection, use for queues, timelines
- **Sets**: Unordered unique collection, use for tags, followers
- **Sorted Sets**: Set with scores, use for leaderboards, rankings

I use hashes for product details because I can update individual fields without fetching the entire object, and sorted sets for product rankings by popularity or price."

### Q3: What is the difference between Redis and MongoDB?

**Answer:**
"Redis is an in-memory key-value store optimized for speed, while MongoDB is a disk-based document database optimized for flexibility. Redis is best for caching, sessions, and real-time data with sub-millisecond latency. MongoDB is best for persistent storage of documents with flexible schemas. I use Redis for caching product data and MongoDB for storing product catalogs with nested specifications and reviews."

### Q4: How does distributed locking work in Redis?

**Answer:**
"Distributed locking in Redis uses the SETNX (SET if Not eXists) command to atomically acquire a lock. I generate a unique UUID as the lock value and set it with an expiration time. To release the lock, I use a Lua script that atomically checks if we own the lock (by comparing the UUID) before deleting it. This prevents releasing someone else's lock. The expiration ensures locks are automatically released even if a process crashes."

### Q5: What happens when Redis goes down?

**Answer:**
"When Redis goes down, I handle it gracefully with a fallback strategy:
1. **Try-catch blocks**: Catch RedisConnectionFailureException
2. **Fallback to database**: Use database when Redis is unavailable
3. **Circuit breaker**: Stop trying Redis after multiple failures
4. **Monitoring**: Alert when Redis is down
5. **High availability**: Use Redis Sentinel or Cluster in production

In our system, if Redis is down, the application continues to work by falling back to the database, though with slower response times."

### Q6: Explain cache invalidation strategies.

**Answer:**
"I use different cache invalidation strategies:
1. **TTL-based**: Keys expire automatically (good for time-sensitive data)
2. **Manual invalidation**: Delete keys on updates (good for consistency)
3. **Cache-aside**: Application manages cache, invalidates on updates
4. **Write-through**: Update cache and database simultaneously
5. **Tag-based**: Invalidate all keys with a tag (e.g., all product:* keys)

I use TTL for product listings (5 minutes) and manual invalidation for individual products when they're updated to ensure consistency."

### Q7: What is the difference between KEYS and SCAN in Redis?

**Answer:**
"KEYS command blocks Redis while scanning all keys, which can cause performance issues in production. SCAN is non-blocking and iterates through keys in batches. I always use SCAN in production:

```java
ScanOptions options = ScanOptions.scanOptions().match("product:*").build();
try (Cursor<String> cursor = redisTemplate.scan(options)) {
    cursor.forEachRemaining(key -> {
        // Process key
    });
}
```

This allows Redis to serve other requests while scanning."

### Q8: How do you handle Redis memory limits?

**Answer:**
"I handle Redis memory limits by:
1. **Setting maxmemory**: Configure maximum memory Redis can use
2. **Eviction policies**: Use LRU (Least Recently Used) to evict least used keys
3. **TTL on all keys**: Ensure keys expire automatically
4. **Monitor memory**: Use INFO memory to track usage
5. **Data structure choice**: Use appropriate structures (hashes for objects instead of JSON strings)

I set maxmemory to 2GB and use allkeys-lru eviction policy, which removes least recently used keys when memory is full."

### Q9: When would you use MongoDB over SQL?

**Answer:**
"I use MongoDB when:
1. **Flexible schema**: Schema changes frequently (product catalogs with varying attributes)
2. **Nested data**: Documents with embedded objects (product with reviews, specifications)
3. **Horizontal scaling**: Need to scale across many servers
4. **High write throughput**: Millions of writes per second
5. **Unstructured data**: JSON documents, logs, user-generated content

For example, in an e-commerce system, I'd use MongoDB for product catalogs where each product has different attributes, and SQL for orders and payments where I need ACID guarantees."

### Q10: Explain CAP theorem in context of NoSQL.

**Answer:**
"CAP theorem states that a distributed system can guarantee only two of three properties:
- **Consistency**: All nodes see the same data
- **Availability**: System remains operational
- **Partition tolerance**: System continues despite network failures

NoSQL databases make different trade-offs:
- **Redis**: CP (Consistency + Partition tolerance) - Single node, consistent
- **MongoDB**: CP with eventual consistency option
- **Cassandra**: AP (Availability + Partition tolerance) - Eventually consistent
- **DynamoDB**: Configurable (can be CP or AP)

I choose based on requirements: Redis for caching (CP), Cassandra for high availability (AP), MongoDB for flexible consistency (configurable)."

---

## Summary

NoSQL databases provide:
- **Horizontal scaling** across multiple servers
- **Flexible schemas** that adapt to data
- **High performance** for specific use cases
- **Specialized data structures** (key-value, documents, graphs)

In our codebase:
- **Redis** for caching, distributed locking, rate limiting
- **H2 (SQL)** for primary data storage (development)
- **Future**: MongoDB for product catalogs, Cassandra for event logs

**Key Takeaways:**
- Use Redis for caching and real-time data
- Use MongoDB for flexible document storage
- Use Cassandra for high write throughput
- Use Neo4j for graph data and recommendations
- Always set TTL on Redis keys
- Handle Redis failures gracefully
- Use SCAN instead of KEYS
- Choose database based on use case

**Next Steps:**
- Implement MongoDB for product catalogs
- Add Redis Cluster for high availability
- Implement Cassandra for event logging
- Add Neo4j for product recommendations

