# SQL Database - Complete Implementation Guide

## Table of Contents
1. [What is SQL Database?](#what-is-sql-database)
2. [SQL Databases Used in Our Codebase](#sql-databases-used-in-our-codebase)
3. [Spring Data JPA and Hibernate](#spring-data-jpa-and-hibernate)
4. [Entity Design and Relationships](#entity-design-and-relationships)
5. [Repository Pattern](#repository-pattern)
6. [Transactions and ACID Properties](#transactions-and-acid-properties)
7. [Query Optimization](#query-optimization)
8. [When to Use SQL Databases](#when-to-use-sql-databases)
9. [Best Practices](#best-practices)
10. [Real-Time Examples from Codebase](#real-time-examples-from-codebase)
11. [Strategies and Patterns](#strategies-and-patterns)
12. [Interview Questions](#interview-questions)

---

## What is SQL Database?

**SQL (Structured Query Language)** databases are relational databases that store data in tables with rows and columns. They enforce ACID properties and use structured schemas.

### Key Characteristics:
- **ACID Compliance**: Atomicity, Consistency, Isolation, Durability
- **Structured Schema**: Predefined tables, columns, and relationships
- **Relationships**: Foreign keys, joins, referential integrity
- **SQL Queries**: Standardized query language
- **Transactions**: Support for complex multi-step operations

### Common SQL Databases:
- **PostgreSQL**: Open-source, feature-rich
- **MySQL**: Popular, widely used
- **Oracle**: Enterprise-grade
- **SQL Server**: Microsoft's solution
- **H2**: In-memory database (used in our codebase for development)

---

## SQL Databases Used in Our Codebase

### H2 Database (In-Memory)

**Location:** `application.yaml` files in all services

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:productdb
    driverClassName: org.h2.Driver
    username: sa
    password: 
  h2:
    console:
      enabled: true
      path: /h2-console
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: update
    show-sql: true
```

**Why H2?**
- **Development**: Fast, no setup required
- **Testing**: Isolated, in-memory database
- **Prototyping**: Quick iteration
- **Production**: Not recommended (use PostgreSQL/MySQL)

**Services Using H2:**
1. **Product Service** (`productdb`)
2. **Seller Service** (`sellerdb`)
3. **User Service** (`userdb`)

---

## Spring Data JPA and Hibernate

### What is JPA?
**JPA (Java Persistence API)** is a specification for managing relational data in Java applications.

### What is Hibernate?
**Hibernate** is the most popular JPA implementation. It provides:
- Object-Relational Mapping (ORM)
- Automatic SQL generation
- Lazy loading
- Caching

### Configuration in Our Codebase

**File:** `application.yaml`

```yaml
spring:
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: update  # Creates/updates schema automatically
    show-sql: true      # Logs SQL queries
```

**DDL Auto Modes:**
- `none`: No automatic schema management
- `validate`: Validates schema, no changes
- `update`: Updates schema (adds new tables/columns)
- `create`: Drops and creates schema on startup
- `create-drop`: Creates on startup, drops on shutdown

---

## Entity Design and Relationships

### Basic Entity Example

**File:** `product-service/src/main/java/com/ekart/product_service/entity/Cart.java`

```java
@Entity
@Table(name = "carts", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"buyer_id", "sku_id"})
})
public class Cart {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "buyer_id", nullable = false)
    private Long buyerId;

    @Column(name = "sku_id", nullable = false)
    private Long skuId;

    @Column(name = "quantity", nullable = false)
    private Integer quantity = 1;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

### Key Annotations:

1. **@Entity**: Marks class as JPA entity
2. **@Table**: Specifies table name and constraints
3. **@Id**: Primary key
4. **@GeneratedValue**: Auto-generation strategy
   - `IDENTITY`: Database auto-increment
   - `SEQUENCE`: Uses database sequence
   - `TABLE`: Uses table for ID generation
5. **@Column**: Column mapping
6. **@PrePersist/@PreUpdate**: Lifecycle callbacks

### Relationship Example

**File:** `seller-service/src/main/java/com/ekart/seller_service/entity/SPU.java`

```java
@Entity
@Table(name = "spus")
public class SPU {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "seller_id", nullable = false)
    private Long sellerId;

    @OneToMany(mappedBy = "spuId", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<SKU> skus;
}
```

### Relationship Types:

1. **@OneToOne**: One-to-one relationship
2. **@OneToMany**: One-to-many relationship
3. **@ManyToOne**: Many-to-one relationship
4. **@ManyToMany**: Many-to-many relationship

### Fetch Types:

- **EAGER**: Loads immediately (can cause N+1 problem)
- **LAZY**: Loads on demand (recommended)

### Cascade Types:

- **ALL**: All operations cascade
- **PERSIST**: Save operations cascade
- **MERGE**: Update operations cascade
- **REMOVE**: Delete operations cascade
- **REFRESH**: Refresh operations cascade

---

## Repository Pattern

### Basic Repository

**File:** `seller-service/src/main/java/com/ekart/seller_service/repository/SellerRepository.java`

```java
@Repository
public interface SellerRepository extends JpaRepository<Seller, Long> {
    Optional<Seller> findByEmail(String email);
    boolean existsByEmail(String email);
}
```

### Spring Data JPA Methods:

**Built-in Methods:**
- `save(entity)`: Save or update
- `findById(id)`: Find by ID
- `findAll()`: Find all
- `delete(entity)`: Delete entity
- `count()`: Count entities
- `existsById(id)`: Check existence

**Query Methods (Derived Queries):**
```java
// Find by single field
findByEmail(String email)
findByName(String name)

// Find by multiple fields
findByEmailAndName(String email, String name)
findByEmailOrName(String email, String name)

// Find with conditions
findByAgeGreaterThan(int age)
findByAgeBetween(int min, int max)
findByNameContaining(String name)
findByNameLike(String pattern)

// Sorting
findByOrderByNameAsc()
findByOrderByCreatedAtDesc()

// Limiting
findFirst10By()
findTop5ByOrderByCreatedAtDesc()
```

### Custom Queries

**File:** `user-service/src/main/java/com/ekart/user_service/repository/RefreshTokenRepository.java`

```java
@Repository
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {
    
    @Modifying(clearAutomatically = true)
    @Query("DELETE FROM RefreshToken rt WHERE rt.user = :user")
    void deleteByUser(@Param("user") User user);
    
    @Modifying(clearAutomatically = true)
    @Query("DELETE FROM RefreshToken rt WHERE rt.token = :token")
    void deleteByToken(@Param("token") String token);
}
```

### Query Annotations:

1. **@Query**: Custom JPQL or SQL query
2. **@Modifying**: Marks query as modifying (update/delete)
3. **@Param**: Binds method parameter to query parameter

### JPQL vs Native SQL:

**JPQL (Java Persistence Query Language):**
```java
@Query("SELECT s FROM Seller s WHERE s.email = :email")
Optional<Seller> findByEmail(@Param("email") String email);
```

**Native SQL:**
```java
@Query(value = "SELECT * FROM sellers WHERE email = :email", nativeQuery = true)
Optional<Seller> findByEmailNative(@Param("email") String email);
```

---

## Transactions and ACID Properties

### What are Transactions?

A **transaction** is a sequence of operations that either all succeed or all fail.

### ACID Properties:

1. **Atomicity**: All or nothing
2. **Consistency**: Data remains valid
3. **Isolation**: Concurrent transactions don't interfere
4. **Durability**: Committed changes persist

### Transaction Example

**File:** `seller-service/src/main/java/com/ekart/seller_service/service/SellerService.java`

```java
@Service
public class SellerService {
    
    @Autowired
    private SellerRepository sellerRepository;
    
    @Transactional
    public Seller createSeller(Seller seller) {
        // All operations in this method are atomic
        Seller saved = sellerRepository.save(seller);
        // If any operation fails, entire transaction rolls back
        return saved;
    }
    
    @Transactional
    public void updateSeller(Long id, Seller updatedSeller) {
        Seller seller = sellerRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Seller not found"));
        
        seller.setBusinessName(updatedSeller.getBusinessName());
        seller.setEmail(updatedSeller.getEmail());
        
        sellerRepository.save(seller);
        // Transaction commits here
    }
}
```

### Transaction Propagation:

**@Transactional(propagation = Propagation.REQUIRED)** (Default)
- Uses existing transaction or creates new one

**@Transactional(propagation = Propagation.REQUIRES_NEW)**
- Always creates new transaction

**@Transactional(propagation = Propagation.SUPPORTS)**
- Uses transaction if exists, otherwise no transaction

**@Transactional(propagation = Propagation.NOT_SUPPORTED)**
- Suspends current transaction if exists

**@Transactional(propagation = Propagation.NEVER)**
- Throws exception if transaction exists

### Isolation Levels:

1. **READ_UNCOMMITTED**: Can read uncommitted data (dirty reads)
2. **READ_COMMITTED**: Only reads committed data (default)
3. **REPEATABLE_READ**: Prevents non-repeatable reads
4. **SERIALIZABLE**: Highest isolation, prevents all anomalies

### Transaction Rollback:

```java
@Transactional(rollbackFor = Exception.class)
public void processOrder(Order order) {
    // Rolls back on any exception
}

@Transactional(noRollbackFor = BusinessException.class)
public void processOrder(Order order) {
    // Doesn't rollback on BusinessException
}
```

---

## Query Optimization

### 1. Use Indexes

```java
@Entity
@Table(name = "sellers", indexes = {
    @Index(name = "idx_email", columnList = "email"),
    @Index(name = "idx_business_name", columnList = "business_name")
})
public class Seller {
    @Column(nullable = false, unique = true)
    private String email;  // Automatically indexed (unique)
}
```

### 2. Avoid N+1 Problem

**❌ BAD: N+1 Queries**
```java
List<SPU> spus = spuRepository.findAll();  // 1 query
for (SPU spu : spus) {
    List<SKU> skus = skuRepository.findBySpuId(spu.getId());  // N queries
}
// Total: 1 + N queries
```

**✅ GOOD: Join Fetch**
```java
@Query("SELECT s FROM SPU s JOIN FETCH s.skus")
List<SPU> findAllWithSkus();
// Total: 1 query with JOIN
```

### 3. Use Pagination

```java
Pageable pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending());
Page<Seller> sellers = sellerRepository.findAll(pageable);
```

### 4. Use Projections

```java
public interface SellerSummary {
    Long getId();
    String getBusinessName();
    String getEmail();
}

@Query("SELECT s.id as id, s.businessName as businessName, s.email as email FROM Seller s")
List<SellerSummary> findAllSummaries();
```

### 5. Batch Operations

```java
@Modifying
@Query("UPDATE Seller s SET s.isActive = :isActive WHERE s.id IN :ids")
void updateActiveStatus(@Param("ids") List<Long> ids, @Param("isActive") Boolean isActive);
```

---

## When to Use SQL Databases

### ✅ Use SQL When:

1. **Structured Data**: Well-defined schema
2. **Relationships**: Complex relationships between entities
3. **ACID Requirements**: Need transactions and consistency
4. **Complex Queries**: Joins, aggregations, subqueries
5. **Data Integrity**: Foreign keys, constraints, validations
6. **Reporting**: Complex analytics and reporting

### ❌ Don't Use SQL When:

1. **Unstructured Data**: JSON, documents, graphs
2. **High Write Throughput**: Millions of writes per second
3. **Horizontal Scaling**: Need to scale across many servers
4. **Simple Key-Value**: Simple get/set operations
5. **Temporary Data**: Session data, cache

---

## Best Practices

### 1. Entity Design

**✅ DO:**
- Use meaningful table and column names
- Add indexes on frequently queried columns
- Use appropriate data types
- Add constraints (NOT NULL, UNIQUE, CHECK)
- Use @PrePersist/@PreUpdate for timestamps

**❌ DON'T:**
- Use reserved keywords as column names
- Create too many indexes (slows writes)
- Use TEXT for small strings
- Forget to handle null values

### 2. Repository Design

**✅ DO:**
- Use derived query methods when possible
- Use @Query for complex queries
- Use pagination for large datasets
- Use projections for read-only data

**❌ DON'T:**
- Use EAGER fetching everywhere
- Load entire entity when only need few fields
- Use N+1 queries
- Forget to handle Optional

### 3. Transaction Management

**✅ DO:**
- Keep transactions short
- Use appropriate isolation levels
- Handle exceptions properly
- Use @Transactional at service layer

**❌ DON'T:**
- Use transactions for read-only operations (unless needed)
- Hold locks for long time
- Catch and swallow exceptions in transactions
- Use too high isolation levels

### 4. Performance

**✅ DO:**
- Use indexes strategically
- Use JOIN FETCH to avoid N+1
- Use pagination
- Monitor slow queries
- Use connection pooling

**❌ DON'T:**
- Select all columns when only need few
- Use SELECT * in production
- Ignore query performance
- Create indexes on every column

---

## Real-Time Examples from Codebase

### Example 1: Seller Entity with Relationships

**File:** `seller-service/src/main/java/com/ekart/seller_service/entity/Seller.java`

```java
@Entity
@Table(name = "sellers")
public class Seller {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Enumerated(EnumType.STRING)
    @Column(name = "kyb_status")
    private KYBStatus kybStatus = KYBStatus.PENDING;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
}
```

**Key Points:**
- Uses `@Enumerated(EnumType.STRING)` for enum storage
- Auto-generates timestamps with `@PrePersist`
- Uses `unique = true` for email (creates unique index)

### Example 2: Transactional Service Method

**File:** `product-service/src/main/java/com/ekart/product_service/service/CartService.java`

```java
@Service
public class CartService {
    
    @Autowired
    private CartRepository cartRepository;
    
    @Transactional
    public Cart addToCart(Long buyerId, Long skuId, Long sellerId, Integer quantity) {
        // Check if item already in cart
        Optional<Cart> existing = cartRepository.findByBuyerIdAndSkuId(buyerId, skuId);
        
        if (existing.isPresent()) {
            Cart cart = existing.get();
            cart.setQuantity(cart.getQuantity() + quantity);
            return cartRepository.save(cart);
        } else {
            Cart newCart = new Cart(buyerId, skuId, sellerId, quantity);
            return cartRepository.save(newCart);
        }
    }
}
```

**Key Points:**
- `@Transactional` ensures atomicity
- If exception occurs, entire operation rolls back
- Uses Optional for null safety

### Example 3: Custom Query with Modifying

**File:** `user-service/src/main/java/com/ekart/user_service/repository/RefreshTokenRepository.java`

```java
@Modifying(clearAutomatically = true)
@Query("DELETE FROM RefreshToken rt WHERE rt.user = :user")
void deleteByUser(@Param("user") User user);
```

**Key Points:**
- `@Modifying` required for UPDATE/DELETE queries
- `clearAutomatically = true` clears persistence context
- Uses JPQL (not native SQL)

---

## Strategies and Patterns

### 1. Repository Pattern

**Purpose:** Abstraction layer between business logic and data access

**Benefits:**
- Testability (can mock repositories)
- Flexibility (can switch implementations)
- Clean separation of concerns

### 2. Unit of Work Pattern

**Purpose:** Track changes and commit as single transaction

**Implementation:** Spring's `@Transactional` provides this automatically

### 3. Active Record Pattern

**Purpose:** Entity contains its own persistence logic

**Example:**
```java
@Entity
public class Seller {
    public Seller save() {
        // Save logic
    }
}
```

**Note:** Spring Data JPA uses Repository pattern instead

### 4. Data Access Object (DAO) Pattern

**Purpose:** Separate data access logic

**Implementation:** Spring Data JPA repositories are DAOs

### 5. Specification Pattern

**Purpose:** Build complex queries dynamically

**Example:**
```java
public class SellerSpecifications {
    public static Specification<Seller> hasEmail(String email) {
        return (root, query, cb) -> cb.equal(root.get("email"), email);
    }
}

List<Seller> sellers = sellerRepository.findAll(
    SellerSpecifications.hasEmail("test@example.com")
);
```

---

## Interview Questions

### Q1: What is the difference between JPA and Hibernate?

**Answer:**
"JPA is a specification (interface) for managing relational data in Java. Hibernate is an implementation of JPA. JPA defines the standard API, while Hibernate provides the actual implementation. You can use JPA annotations and switch between different JPA implementations (Hibernate, EclipseLink, etc.) without changing your code."

### Q2: What is the N+1 problem and how do you solve it?

**Answer:**
"The N+1 problem occurs when you fetch a list of entities and then fetch related entities for each one, resulting in 1 query for the list and N queries for related entities. I solve it by using JOIN FETCH in JPQL queries or @EntityGraph annotation. For example:

```java
@Query("SELECT s FROM SPU s JOIN FETCH s.skus")
List<SPU> findAllWithSkus();
```

This fetches everything in a single query with a JOIN instead of N+1 queries."

### Q3: Explain @Transactional annotation.

**Answer:**
"@Transactional ensures that a method executes within a database transaction. If the method completes successfully, changes are committed. If an exception occurs, changes are rolled back. I use it at the service layer for business operations that need atomicity. I can configure propagation behavior, isolation level, and rollback conditions."

### Q4: What is the difference between EAGER and LAZY fetching?

**Answer:**
"EAGER fetching loads related entities immediately when the parent entity is loaded, which can cause performance issues and unnecessary data loading. LAZY fetching loads related entities only when accessed, which is more efficient. I prefer LAZY fetching and use JOIN FETCH when I need related data, avoiding the N+1 problem while maintaining performance."

### Q5: How do you handle database migrations?

**Answer:**
"In development, I use `ddl-auto: update` for automatic schema updates. In production, I use Flyway or Liquibase for version-controlled migrations. These tools track applied migrations and ensure database schema consistency across environments. I never use `ddl-auto: create` or `update` in production."

### Q6: What is the difference between JPQL and native SQL?

**Answer:**
"JPQL (Java Persistence Query Language) is database-agnostic and works with entities. Native SQL is database-specific and works with tables directly. I prefer JPQL for portability, but use native SQL for complex queries that JPQL can't handle efficiently. I always use parameter binding to prevent SQL injection."

### Q7: How do you optimize database queries?

**Answer:**
"I optimize queries by:
1. Adding indexes on frequently queried columns
2. Using JOIN FETCH to avoid N+1 problems
3. Using pagination for large datasets
4. Using projections to fetch only needed fields
5. Monitoring slow queries and analyzing execution plans
6. Using batch operations for bulk updates
7. Properly configuring connection pooling"

### Q8: What are the ACID properties?

**Answer:**
"ACID stands for:
- **Atomicity**: All operations in a transaction succeed or all fail
- **Consistency**: Database remains in valid state after transaction
- **Isolation**: Concurrent transactions don't interfere with each other
- **Durability**: Committed changes persist even after system failure

SQL databases guarantee these properties, which is why I use them for critical operations like order processing and inventory management."

### Q9: How do you handle database connections in a multi-threaded environment?

**Answer:**
"I use connection pooling (HikariCP is the default in Spring Boot) which manages a pool of database connections. Each thread gets a connection from the pool, uses it, and returns it. This prevents connection exhaustion and improves performance. I configure the pool size based on my application's concurrency needs."

### Q10: What is the difference between save() and saveAndFlush()?

**Answer:**
"save() queues the entity for persistence but doesn't immediately write to the database. saveAndFlush() immediately writes changes to the database. I use save() for normal operations and saveAndFlush() when I need the generated ID immediately or need to catch database constraint violations right away."

---

## Summary

SQL databases provide:
- **ACID guarantees** for data consistency
- **Structured schema** for data integrity
- **Complex queries** with joins and aggregations
- **Relationships** with foreign keys
- **Transactions** for atomic operations

In our codebase:
- **H2** for development/testing
- **Spring Data JPA** for data access
- **Hibernate** as JPA implementation
- **Repository pattern** for abstraction
- **@Transactional** for transaction management

**Next Steps:**
- Migrate to PostgreSQL/MySQL for production
- Add Flyway for database migrations
- Implement connection pooling optimization
- Add query performance monitoring

