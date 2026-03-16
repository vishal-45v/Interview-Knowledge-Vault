# Production Failure: Message Queue Backlog

---

## Scenario: Kafka Consumer Group Falling Behind

**Symptom:** Order notifications delayed by 6 hours. Consumer lag growing rapidly.

```
Normal:
  Orders/sec: 1,000
  Notification processing: 1,000/sec
  Lag: ~0 (real-time)

Incident:
  Orders/sec: 1,000
  Notification processing: 200/sec (email provider rate-limited)
  Lag growth: 800 messages/sec
  After 6 hours: 800 × 21,600 = 17 million messages backlog!
```

---

## Root Cause Analysis

```
1. Orders published to Kafka → order-events topic
2. NotificationConsumer reads and sends email via Mailgun API
3. Mailgun throttles us (429 Too Many Requests)
4. Consumer retries with no backoff → still throttled
5. Consumer falls further behind

Log pattern:
  [WARN] Failed to send email, retrying in 1s
  [WARN] Failed to send email, retrying in 1s
  [WARN] ... (repeating constantly)
```

---

## Fixes

### Exponential Backoff with Jitter

```java
@RetryableTopic(
    attempts = "5",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, random = true),
    dltTopicSuffix = "-dlt"  // Dead letter topic
)
@KafkaListener(topics = "order-events")
public void processOrderEvent(OrderEvent event) {
    emailService.sendOrderConfirmation(event);
}
```

### Dead Letter Queue for Poison Messages

```java
@DltHandler
public void handleDltRecord(OrderEvent event, @Header KafkaHeaders.RECEIVED_TOPIC topic) {
    log.error("Event moved to DLT after max retries: {}, topic: {}", event, topic);
    alertService.sendAlert("Event processing failed permanently: " + event.getOrderId());
    // Store for manual review
    failedEventRepository.save(new FailedEvent(event, topic));
}
```

### Scale Consumers to Reduce Backlog

```bash
# Increase consumer instances (Kubernetes)
kubectl scale deployment notification-consumer --replicas=20

# Or configure multiple concurrent consumers per partition
@KafkaListener(topics = "order-events", concurrency = "5")  # 5 consumer threads
public void processOrderEvent(OrderEvent event) { ... }

# Note: max concurrency = number of partitions in topic
```

### Circuit Breaker for Email Provider

```java
@CircuitBreaker(name = "email-provider")
public void sendEmail(String to, String subject, String body) {
    mailgunClient.send(to, subject, body);
}

// When circuit opens: stop consuming from Kafka (pause consumer)
@EventListener
public void onCircuitBreakerStateChange(CircuitBreakerOnStateTransitionEvent event) {
    if (event.getStateTransition().getToState() == OPEN) {
        kafkaListenerEndpointRegistry.getListenerContainer("order-events").pause();
    } else if (event.getStateTransition().getToState() == CLOSED) {
        kafkaListenerEndpointRegistry.getListenerContainer("order-events").resume();
    }
}
```

---

## Backlog Recovery Plan

```
When email provider is back:
1. Resume consumers
2. Increase consumer parallelism to process backlog faster
3. Monitor lag: kafka-consumer-groups.sh --describe
4. Set priority: process new orders first (latest offset), then catch up on older
5. Notify customers of delay if > acceptable threshold
6. After backlog cleared: scale consumers back down
```
