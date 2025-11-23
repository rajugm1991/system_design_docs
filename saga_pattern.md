# Saga Pattern - Complete Implementation Guide

## Table of Contents
1. [What is Saga Pattern?](#what-is-saga-pattern)
2. [How Saga is Used in Our Codebase](#how-saga-is-used-in-our-codebase)
3. [Code Structure and Files](#code-structure-and-files)
4. [Complete Flow: From API Call to Completion](#complete-flow-from-api-call-to-completion)
5. [Detailed Code Walkthrough](#detailed-code-walkthrough)
6. [When to Use Saga Pattern?](#when-to-use-saga-pattern)
7. [Best Practices](#best-practices)
8. [Interview Questions & Answers](#interview-questions--answers)

---

## What is Saga Pattern?

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

## How Saga is Used in Our Codebase

### Overview

In our e-commerce system, Saga pattern is used for **order processing** (checkout flow). When a user places an order, multiple services need to coordinate:
- Create order
- Reserve inventory
- Process payment
- Update inventory
- Send notification

### Complete Request Flow

```
┌─────────────────────────────────────────────────────────┐
│              COMPLETE REQUEST FLOW                       │
└─────────────────────────────────────────────────────────┘

1. User clicks "Checkout" (Frontend)
        │
        ▼
2. POST /api/products/checkout (CheckoutController)
        │
        ▼
3. OrderService.checkout()
        │
        ├─→ Validates cart items
        │
        ▼
4. OrderSagaOrchestrator.processOrder()
        │
        ├─→ Step 1: CreateOrderStep
        ├─→ Step 2: ReserveInventoryStep
        ├─→ Step 3: ProcessPaymentStep
        ├─→ Step 4: UpdateInventoryStep
        └─→ Step 5: SendNotificationStep
        │
        ▼
5. Returns SagaResult
        │
        ▼
6. OrderService returns ApiResponse
        │
        ▼
7. CheckoutController returns HTTP Response
```

### Code Structure

```
product-service/
└── src/main/java/com/ekart/product_service/
    ├── controller/
    │   └── CheckoutController.java          ← Entry point
    ├── service/
    │   └── OrderService.java                ← Calls saga orchestrator
    ├── saga/
    │   └── OrderSagaOrchestrator.java       ← Main orchestrator
    └── saga/step/
        ├── SagaStep.java                    ← Interface
        ├── CreateOrderStep.java             ← Step 1
        ├── ReserveInventoryStep.java        ← Step 2
        ├── ProcessPaymentStep.java          ← Step 3
        ├── UpdateInventoryStep.java         ← Step 4
        └── SendNotificationStep.java        ← Step 5
```

---

## Code Structure and Files

### 1. Entry Point: CheckoutController

**File:** `product-service/src/main/java/.../controller/CheckoutController.java`

```java
@RestController
@RequestMapping("/api/products/checkout")
public class CheckoutController {
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping
    public ResponseEntity<ApiResponse<Map<String, Object>>> checkout(
            @RequestHeader("X-User-Id") String userIdHeader,
            @Valid @RequestBody CheckoutRequest request) {
        
        Long buyerId = Long.parseLong(userIdHeader);
        
        // Call OrderService which uses Saga
        ApiResponse<Map<String, Object>> response = 
            orderService.checkout(buyerId, request);
        
        return ResponseEntity.ok(response);
    }
}
```

**What it does:**
- Receives checkout request from frontend
- Extracts buyer ID from header
- Calls `OrderService.checkout()`
- Returns response to user

### 2. Order Service: Validates and Triggers Saga

**File:** `product-service/src/main/java/.../service/OrderService.java`

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderSagaOrchestrator sagaOrchestrator;
    
    public ApiResponse<Map<String, Object>> checkout(
            Long buyerId, CheckoutRequest request) {
        
        // 1. Validate cart items
        for (CartItemRequest item : request.getCartItems()) {
            // Check if product exists
            // Check if inventory is available
        }
        
        // 2. Trigger Saga
        SagaResult sagaResult = sagaOrchestrator.processOrder(buyerId, request);
        
        // 3. Return result
        if (sagaResult.isSuccess()) {
            return new ApiResponse<>(true, "Order placed", responseData);
        } else {
            return new ApiResponse<>(false, sagaResult.getMessage(), null);
        }
    }
}
```

**What it does:**
- Validates cart items before starting saga
- Calls saga orchestrator
- Returns success/failure response

### 3. Saga Orchestrator: Coordinates All Steps

**File:** `product-service/src/main/java/.../saga/OrderSagaOrchestrator.java`

```java
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private CreateOrderStep createOrderStep;
    @Autowired
    private ReserveInventoryStep reserveInventoryStep;
    @Autowired
    private ProcessPaymentStep processPaymentStep;
    @Autowired
    private UpdateInventoryStep updateInventoryStep;
    @Autowired
    private SendNotificationStep sendNotificationStep;
    
    public SagaResult processOrder(Long buyerId, CheckoutRequest request) {
        // 1. Create saga context
        String sagaId = UUID.randomUUID().toString();
        SagaContext context = new SagaContext(sagaId, buyerId, request);
        
        // 2. Track completed steps
        List<SagaStep> completedSteps = new ArrayList<>();
        
        try {
            // 3. Execute steps in order
            createOrderStep.execute(context);
            completedSteps.add(createOrderStep);
            
            reserveInventoryStep.execute(context);
            completedSteps.add(reserveInventoryStep);
            
            processPaymentStep.execute(context);
            completedSteps.add(processPaymentStep);
            
            updateInventoryStep.execute(context);
            completedSteps.add(updateInventoryStep);
            
            sendNotificationStep.execute(context);
            completedSteps.add(sendNotificationStep);
            
            // 4. Success!
            return new SagaResult(true, "Success", context);
            
        } catch (Exception e) {
            // 5. Failure - compensate!
            compensate(completedSteps, context, e);
            return new SagaResult(false, e.getMessage(), context);
        }
    }
    
    private void compensate(List<SagaStep> completedSteps, 
                           SagaContext context, Exception error) {
        // Compensate in reverse order
        for (int i = completedSteps.size() - 1; i >= 0; i--) {
            SagaStep step = completedSteps.get(i);
            step.compensate(context);
        }
    }
}
```

**What it does:**
- Creates unique saga ID
- Executes steps sequentially
- Tracks completed steps
- If any step fails, compensates in reverse order

---

## Complete Flow: From API Call to Completion

### Step-by-Step Flow with Code

```
┌─────────────────────────────────────────────────────────┐
│              STEP 1: API Request                        │
└─────────────────────────────────────────────────────────┘

Frontend → POST /api/products/checkout
{
  "cartItems": [
    {"skuId": 1, "quantity": 2},
    {"skuId": 2, "quantity": 1}
  ],
  "shippingAddress": "...",
  "paymentMethod": "CREDIT_CARD"
}

        │
        ▼
┌─────────────────────────────────────────────────────────┐
│              STEP 2: CheckoutController                 │
└─────────────────────────────────────────────────────────┘

CheckoutController.checkout()
    │
    ├─→ Extracts buyerId from header
    │
    └─→ Calls OrderService.checkout(buyerId, request)
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│              STEP 3: OrderService Validation            │
└─────────────────────────────────────────────────────────┘

OrderService.checkout()
    │
    ├─→ Validates each cart item
    │   ├─→ Product exists?
    │   └─→ Inventory available?
    │
    └─→ Calls sagaOrchestrator.processOrder()
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│              STEP 4: Saga Orchestrator Starts           │
└─────────────────────────────────────────────────────────┘

OrderSagaOrchestrator.processOrder()
    │
    ├─→ Creates SagaContext
    │   └─→ sagaId = UUID.randomUUID()
    │
    ├─→ Initializes completedSteps = []
    │
    └─→ Starts executing steps...
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│              STEP 5: Execute Steps                      │
└─────────────────────────────────────────────────────────┘

Step 1: CreateOrderStep.execute()
    │
    ├─→ Generates orderId = "ORD-XXXXXXXX"
    ├─→ Sets context.setOrderId(orderId)
    └─→ completedSteps.add(createOrderStep) ✅
        │
        ▼
Step 2: ReserveInventoryStep.execute()
    │
    ├─→ For each cart item:
    │   ├─→ Acquires distributed lock
    │   ├─→ Reserves inventory
    │   └─→ Releases lock
    ├─→ Sets context.setInventoryReserved(true)
    └─→ completedSteps.add(reserveInventoryStep) ✅
        │
        ▼
Step 3: ProcessPaymentStep.execute()
    │
    ├─→ Calculates total amount
    ├─→ Processes payment (simulated)
    ├─→ Sets context.setPaymentProcessed(true)
    └─→ completedSteps.add(processPaymentStep) ✅
        │
        ▼
Step 4: UpdateInventoryStep.execute()
    │
    ├─→ For each cart item:
    │   ├─→ Acquires distributed lock
    │   ├─→ Updates inventory (deducts stock)
    │   └─→ Releases lock
    ├─→ Sets context.setInventoryUpdated(true)
    └─→ completedSteps.add(updateInventoryStep) ✅
        │
        ▼
Step 5: SendNotificationStep.execute()
    │
    ├─→ Sends order confirmation email
    ├─→ Sets context.setNotificationSent(true)
    └─→ completedSteps.add(sendNotificationStep) ✅
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│              STEP 6: Saga Success                       │
└─────────────────────────────────────────────────────────┘

SagaResult(true, "Order processed successfully", context)
    │
    ├─→ Publishes OrderCompleted event
    │
    └─→ Returns to OrderService
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│              STEP 7: Response to User                   │
└─────────────────────────────────────────────────────────┘

OrderService returns:
{
  "success": true,
  "message": "Order placed successfully",
  "data": {
    "orderId": "ORD-XXXXXXXX",
    "buyerId": 1,
    "status": "CONFIRMED",
    "sagaId": "uuid-here"
  }
}
```

---

## Detailed Code Walkthrough

### Step 1: CreateOrderStep

**File:** `saga/step/CreateOrderStep.java`

```java
@Component
public class CreateOrderStep implements SagaStep {
    
    @Override
    public void execute(SagaContext context) throws Exception {
        // Generate unique order ID
        String orderId = "ORD-" + UUID.randomUUID()
            .toString().substring(0, 8).toUpperCase();
        
        // Store in context for other steps
        context.setOrderId(orderId);
        
        // In production: Save order to database
        // orderRepository.save(new Order(orderId, ...));
    }
    
    @Override
    public void compensate(SagaContext context) throws Exception {
        // Cancel the order
        if (context.getOrderId() != null) {
            // In production: Update order status to CANCELLED
            // orderRepository.updateStatus(context.getOrderId(), "CANCELLED");
            context.addCompensationLog("Order cancelled: " + context.getOrderId());
        }
    }
}
```

**What happens:**
- Generates unique order ID (e.g., "ORD-A1B2C3D4")
- Stores in context for other steps to use
- Compensation: Cancels the order

### Step 2: ReserveInventoryStep

**File:** `saga/step/ReserveInventoryStep.java`

```java
@Component
public class ReserveInventoryStep implements SagaStep {
    
    @Autowired
    private DistributedLockService lockService;
    
    @Override
    public void execute(SagaContext context) throws Exception {
        // For each item in cart
        context.getRequest().getCartItems().forEach(item -> {
            String lockKey = "inventory:reserve:" + item.getSkuId();
            
            // Use distributed lock to prevent race conditions
            lockService.executeWithLock(lockKey, Duration.ofSeconds(30), () -> {
                // Reserve inventory
                // In production: Call inventory service
                // inventoryService.reserve(item.getSkuId(), item.getQuantity());
                return null;
            });
        });
        
        context.setInventoryReserved(true);
    }
    
    @Override
    public void compensate(SagaContext context) throws Exception {
        if (context.isInventoryReserved()) {
            // Release reserved inventory
            context.getRequest().getCartItems().forEach(item -> {
                String lockKey = "inventory:release:" + item.getSkuId();
                
                lockService.executeWithLock(lockKey, Duration.ofSeconds(30), () -> {
                    // Release inventory
                    // inventoryService.release(item.getSkuId(), item.getQuantity());
                    return null;
                });
            });
            
            context.setInventoryReserved(false);
        }
    }
}
```

**What happens:**
- Reserves inventory for each cart item
- Uses distributed locks to prevent race conditions
- Compensation: Releases reserved inventory

### Step 3: ProcessPaymentStep

**File:** `saga/step/ProcessPaymentStep.java`

```java
@Component
public class ProcessPaymentStep implements SagaStep {
    
    @Override
    public void execute(SagaContext context) throws Exception {
        // Calculate total amount
        BigDecimal totalAmount = context.getRequest().getCartItems().stream()
            .map(item -> {
                BigDecimal unitPrice = BigDecimal.valueOf(1000); // Mock
                return unitPrice.multiply(BigDecimal.valueOf(item.getQuantity()));
            })
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        // Process payment
        boolean paymentSuccess = simulatePaymentProcessing(
            context.getOrderId(), totalAmount);
        
        if (!paymentSuccess) {
            throw new PaymentProcessingException("Payment failed");
        }
        
        context.setPaymentProcessed(true);
    }
    
    @Override
    public void compensate(SagaContext context) throws Exception {
        if (context.isPaymentProcessed()) {
            // Refund payment
            boolean refundSuccess = simulateRefund(context.getOrderId());
            
            if (!refundSuccess) {
                throw new RefundException("Refund failed");
            }
            
            context.setPaymentProcessed(false);
        }
    }
}
```

**What happens:**
- Calculates total order amount
- Processes payment (charges card)
- Compensation: Refunds payment if saga fails

### Step 4: UpdateInventoryStep

**File:** `saga/step/UpdateInventoryStep.java`

```java
@Component
public class UpdateInventoryStep implements SagaStep {
    
    @Autowired
    private DistributedLockService lockService;
    
    @Override
    public void execute(SagaContext context) throws Exception {
        // Update inventory (deduct stock)
        context.getRequest().getCartItems().forEach(item -> {
            String lockKey = "inventory:update:" + item.getSkuId();
            
            lockService.executeWithLock(lockKey, Duration.ofSeconds(30), () -> {
                // Deduct inventory
                // inventoryService.deduct(item.getSkuId(), item.getQuantity());
                return null;
            });
        });
        
        context.setInventoryUpdated(true);
    }
    
    @Override
    public void compensate(SagaContext context) throws Exception {
        if (context.isInventoryUpdated()) {
            // Restore inventory
            context.getRequest().getCartItems().forEach(item -> {
                String lockKey = "inventory:restore:" + item.getSkuId();
                
                lockService.executeWithLock(lockKey, Duration.ofSeconds(30), () -> {
                    // Restore inventory
                    // inventoryService.restore(item.getSkuId(), item.getQuantity());
                    return null;
                });
            });
            
            context.setInventoryUpdated(false);
        }
    }
}
```

**What happens:**
- Deducts inventory after payment is confirmed
- Uses distributed locks
- Compensation: Restores inventory

### Step 5: SendNotificationStep

**File:** `saga/step/SendNotificationStep.java`

```java
@Component
public class SendNotificationStep implements SagaStep {
    
    @Override
    public void execute(SagaContext context) throws Exception {
        // Send order confirmation email
        sendOrderConfirmation(context.getBuyerId(), context.getOrderId());
        context.setNotificationSent(true);
    }
    
    @Override
    public void compensate(SagaContext context) throws Exception {
        // Notifications are idempotent, but send cancellation notice
        if (context.isNotificationSent()) {
            sendOrderCancellation(context.getBuyerId(), context.getOrderId());
        }
    }
}
```

**What happens:**
- Sends order confirmation email
- Compensation: Sends cancellation email (optional)

---

### SagaContext: State Management

**File:** `saga/OrderSagaOrchestrator.java` (inner class)

```java
public static class SagaContext {
    private String sagaId;              // Unique saga identifier
    private Long buyerId;               // Who placed the order
    private CheckoutRequest request;    // Original request
    private String orderId;             // Generated order ID
    private boolean inventoryReserved;  // Step 2 completed?
    private boolean paymentProcessed;   // Step 3 completed?
    private boolean inventoryUpdated;   // Step 4 completed?
    private boolean notificationSent;   // Step 5 completed?
    private List<String> compensationLog; // Log of compensations
}
```

**Purpose:**
- Holds state throughout saga execution
- Allows steps to share data
- Tracks which steps completed (for idempotency)

---

### Visual: Complete Code Flow

```
┌─────────────────────────────────────────────────────────┐
│         API REQUEST FLOW                                 │
└─────────────────────────────────────────────────────────┘

HTTP POST /api/products/checkout
    │
    ▼
CheckoutController.checkout()
    │
    ├─→ Extracts buyerId
    │
    └─→ OrderService.checkout(buyerId, request)
        │
        ├─→ Validates cart items
        │
        └─→ OrderSagaOrchestrator.processOrder()
            │
            ├─→ Creates SagaContext
            │   └─→ sagaId, buyerId, request
            │
            ├─→ List<SagaStep> completedSteps = []
            │
            └─→ try {
                    │
                    ├─→ Step 1: createOrderStep.execute(context)
                    │   └─→ orderId = "ORD-XXXX"
                    │   └─→ completedSteps.add(createOrderStep)
                    │
                    ├─→ Step 2: reserveInventoryStep.execute(context)
                    │   └─→ Reserves inventory
                    │   └─→ completedSteps.add(reserveInventoryStep)
                    │
                    ├─→ Step 3: processPaymentStep.execute(context)
                    │   └─→ Processes payment
                    │   └─→ completedSteps.add(processPaymentStep)
                    │
                    ├─→ Step 4: updateInventoryStep.execute(context)
                    │   └─→ Updates inventory
                    │   └─→ completedSteps.add(updateInventoryStep)
                    │
                    └─→ Step 5: sendNotificationStep.execute(context)
                        └─→ Sends email
                        └─→ completedSteps.add(sendNotificationStep)
                        │
                        └─→ return SagaResult(true, "Success", context)
                
                } catch (Exception e) {
                    │
                    └─→ compensate(completedSteps, context, e)
                        │
                        ├─→ For each step in reverse:
                        │   ├─→ sendNotificationStep.compensate()
                        │   ├─→ updateInventoryStep.compensate()
                        │   ├─→ processPaymentStep.compensate()
                        │   ├─→ reserveInventoryStep.compensate()
                        │   └─→ createOrderStep.compensate()
                        │
                        └─→ return SagaResult(false, error, context)
                }
```

---

### Example: Real Execution Log

```
2025-11-11 19:51:44.502 [http-nio-8082-exec-5] INFO  OrderSagaOrchestrator - Starting order saga: sagaId=4b0f192b-29b8-4457-8c46-5dd35ec55ac6

2025-11-11 19:51:44.502 [http-nio-8082-exec-5] INFO  OrderSagaOrchestrator - Saga Step 1: Creating order
2025-11-11 19:51:44.503 [http-nio-8082-exec-5] INFO  CreateOrderStep - Order created: orderId=ORD-3309BAC3

2025-11-11 19:51:44.503 [http-nio-8082-exec-5] INFO  OrderSagaOrchestrator - Saga Step 2: Reserving inventory
2025-11-11 19:51:44.933 [http-nio-8082-exec-5] INFO  ReserveInventoryStep - Inventory reserved for order: ORD-3309BAC3

2025-11-11 19:51:44.933 [http-nio-8082-exec-5] INFO  OrderSagaOrchestrator - Saga Step 3: Processing payment
2025-11-11 19:51:45.034 [http-nio-8082-exec-5] INFO  ProcessPaymentStep - Payment processed for order: ORD-3309BAC3, amount: 2000

2025-11-11 19:51:45.034 [http-nio-8082-exec-5] INFO  OrderSagaOrchestrator - Saga Step 4: Updating inventory
2025-11-11 19:51:45.038 [http-nio-8082-exec-5] INFO  UpdateInventoryStep - Inventory updated for order: ORD-3309BAC3

2025-11-11 19:51:45.038 [http-nio-8082-exec-5] INFO  OrderSagaOrchestrator - Saga Step 5: Sending notification
2025-11-11 19:51:45.038 [http-nio-8082-exec-5] INFO  SendNotificationStep - Notification sent for order: ORD-3309BAC3

2025-11-11 19:51:45.038 [http-nio-8082-exec-5] INFO  OrderSagaOrchestrator - Order saga completed successfully: sagaId=4b0f192b-29b8-4457-8c46-5dd35ec55ac6
```

---

### Example: Failure with Compensation

```
2025-11-11 20:00:00.000 [http-nio-8082-exec-1] INFO  OrderSagaOrchestrator - Starting order saga: sagaId=abc-123

2025-11-11 20:00:00.010 [http-nio-8082-exec-1] INFO  OrderSagaOrchestrator - Saga Step 1: Creating order
2025-11-11 20:00:00.020 [http-nio-8082-exec-1] INFO  CreateOrderStep - Order created: orderId=ORD-12345678

2025-11-11 20:00:00.030 [http-nio-8082-exec-1] INFO  OrderSagaOrchestrator - Saga Step 2: Reserving inventory
2025-11-11 20:00:00.100 [http-nio-8082-exec-1] INFO  ReserveInventoryStep - Inventory reserved for order: ORD-12345678

2025-11-11 20:00:00.200 [http-nio-8082-exec-1] INFO  OrderSagaOrchestrator - Saga Step 3: Processing payment
2025-11-11 20:00:00.300 [http-nio-8082-exec-1] ERROR ProcessPaymentStep - Payment failed: Insufficient funds
2025-11-11 20:00:00.300 [http-nio-8082-exec-1] ERROR OrderSagaOrchestrator - Order saga failed: sagaId=abc-123

2025-11-11 20:00:00.310 [http-nio-8082-exec-1] INFO  OrderSagaOrchestrator - Compensating 2 completed steps

2025-11-11 20:00:00.320 [http-nio-8082-exec-1] INFO  ReserveInventoryStep - Compensating ReserveInventoryStep
2025-11-11 20:00:00.400 [http-nio-8082-exec-1] INFO  ReserveInventoryStep - Inventory released for order: ORD-12345678

2025-11-11 20:00:00.410 [http-nio-8082-exec-1] INFO  CreateOrderStep - Compensating CreateOrderStep
2025-11-11 20:00:00.420 [http-nio-8082-exec-1] INFO  CreateOrderStep - Order cancelled: orderId=ORD-12345678
```

---

### Key Implementation Details

#### 1. SagaContext State Tracking

```java
// Context tracks state for idempotency
context.setInventoryReserved(true);  // Step 2 done
context.setPaymentProcessed(true);   // Step 3 done
context.setInventoryUpdated(true);   // Step 4 done
context.setNotificationSent(true);   // Step 5 done
```

**Why?** Allows steps to check if already executed (idempotency).

#### 2. Distributed Locks

```java
// Used in ReserveInventoryStep and UpdateInventoryStep
lockService.executeWithLock(lockKey, Duration.ofSeconds(30), () -> {
    // Critical section - only one saga can execute at a time
    inventoryService.reserve(skuId, quantity);
    return null;
});
```

**Why?** Prevents race conditions when multiple orders process simultaneously.

#### 3. Compensation in Reverse Order

```java
// Compensate from last to first
for (int i = completedSteps.size() - 1; i >= 0; i--) {
    SagaStep step = completedSteps.get(i);
    step.compensate(context);
}
```

**Why?** Must undo in reverse order to maintain consistency.

#### 4. Event Publishing

```java
// After successful saga
eventPublisher.publishOrderCompleted(context.getOrderId(), buyerId);

// After failed saga
eventPublisher.publishOrderCancelled(context.getOrderId(), buyerId, error);
```

**Why?** Other services can react to order completion/cancellation.

---

### Integration with Other Services

```
┌─────────────────────────────────────────────────────────┐
│         SAGA INTEGRATION                                 │
└─────────────────────────────────────────────────────────┘

OrderSagaOrchestrator
    │
    ├─→ Uses DistributedLockService
    │   └─→ Prevents concurrent inventory updates
    │
    ├─→ Uses OrderEventPublisher
    │   ├─→ Publishes OrderCompleted event
    │   └─→ Publishes OrderCancelled event
    │
    └─→ Steps can call external services:
        ├─→ Inventory Service (via RestTemplate)
        ├─→ Payment Service (via RestTemplate)
        └─→ Notification Service (via RestTemplate)
```

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
