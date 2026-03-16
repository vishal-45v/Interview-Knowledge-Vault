# Production Failure: Distributed System Network Partition

---

## Scenario: Network Split Between Services

**Situation:** Network switch failure causes Order Service to lose connectivity to Inventory Service. Orders are placed without inventory decremented. Inventory oversold.

---

## What Went Wrong

```
Order Service → (network split) → Inventory Service
       │
       ▼
  "Inventory service timed out — assume in stock" (optimistic fallback)
  → Order created ✓
  → Inventory NOT decremented ✗
  → 500 oversold items in 10 minutes
```

---

## Saga Pattern for Distributed Transactions

```java
// Choreography-based saga with compensating transactions:

// 1. OrderService creates order in PENDING state
@Transactional
public Order createOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request, OrderStatus.PENDING));
    eventPublisher.publishEvent(new OrderCreatedEvent(order));
    return order;
}

// 2. InventoryService listens and reserves inventory
@TransactionalEventListener
@Retryable(value = Exception.class, maxAttempts = 3)
public void onOrderCreated(OrderCreatedEvent event) {
    try {
        inventoryService.reserve(event.getOrderId(), event.getItems());
        eventPublisher.publishEvent(new InventoryReservedEvent(event.getOrderId()));
    } catch (InsufficientInventoryException e) {
        eventPublisher.publishEvent(new InventoryFailedEvent(event.getOrderId(), e.getMessage()));
    }
}

// 3. OrderService listens for outcome
@EventListener
public void onInventoryReserved(InventoryReservedEvent event) {
    orderRepository.updateStatus(event.getOrderId(), OrderStatus.CONFIRMED);
}

@EventListener
public void onInventoryFailed(InventoryFailedEvent event) {
    orderRepository.updateStatus(event.getOrderId(), OrderStatus.CANCELLED);
    // Notify customer
}
```

---

## Circuit Breaker as Defense

```java
// Don't blindly assume success on timeout!
@CircuitBreaker(name = "inventory",
    fallbackMethod = "inventoryFallback")
public InventoryResult reserve(Long orderId, List<OrderItem> items) {
    return inventoryService.reserve(orderId, items);
}

// PESSIMISTIC fallback: Reject order rather than risk overselling
public InventoryResult inventoryFallback(Long orderId, List<OrderItem> items, Exception e) {
    log.error("Inventory service unavailable, rejecting order {}", orderId, e);
    throw new ServiceUnavailableException("Unable to process order - inventory service unavailable");
}
```

---

## Eventual Consistency Reconciliation

```
After network partition is resolved:
1. Compare Order DB state with Inventory DB state
2. Find orders created during partition
3. For each: attempt to decrement inventory
4. If inventory insufficient: cancel order, refund, notify customer

Reconciliation job:
@Scheduled(fixedDelay = 60000)  // Every minute
public void reconcileOrders() {
    List<Order> pendingOrders = orderRepository.findByStatus(PENDING_INVENTORY);
    for (Order order : pendingOrders) {
        try {
            inventoryService.reserve(order.getId(), order.getItems());
            order.setStatus(CONFIRMED);
        } catch (InsufficientInventoryException e) {
            order.setStatus(CANCELLED);
            refundService.refund(order);
        }
        orderRepository.save(order);
    }
}
```
