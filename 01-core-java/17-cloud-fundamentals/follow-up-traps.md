# Cloud Fundamentals — Follow-Up Traps

---

## Trap 1: EC2 vs ECS vs EKS — When to Use Which?

| Service | What it is | When to use |
|---|---|---|
| **EC2** | Raw virtual machines. You manage OS, runtime, patching | Legacy apps, max control needed, specific OS configs |
| **ECS** | AWS-managed container orchestrator. Run Docker containers. | Containerized apps, simpler than Kubernetes, AWS-native teams |
| **EKS** | Managed Kubernetes. AWS manages control plane. | When you need Kubernetes (multi-cloud portability, custom operators, large orgs with existing K8s expertise) |

**EC2 vs ECS:** ECS is almost always better for new containerized apps. EC2 requires managing AMIs, patching the OS, and installing the container runtime yourself. ECS handles scheduling, health checks, rolling deployments, and service discovery automatically.

**ECS vs EKS:** EKS wins when you have multi-cloud requirements (Kubernetes works on GCP, Azure, on-prem), when your team already knows Kubernetes, or when you need advanced features (custom operators, admission webhooks, Istio service mesh). ECS wins for AWS-only teams who want simplicity.

**Fargate** is the serverless compute option for both ECS and EKS — you don't manage the underlying EC2 instances at all. You pay per vCPU/memory per second. Use it when you want to run containers without caring about servers.

---

## Trap 2: Lambda Cold Start — What Causes It and How to Minimize?

**What is a cold start?**
When Lambda has no pre-warmed execution environment for your function, it must:
1. Provision a new container (AWS-managed)
2. Download your deployment package / container image
3. Start the JVM (for Java) — this is the big one
4. Run static initializers and Spring context loading

For a Spring Boot Lambda, this can take **5-15 seconds** on the first invocation. Subsequent invocations reuse the same container (warm) and respond in milliseconds.

**Minimization strategies:**

1. **Reduce package size** — smaller JAR = faster download + startup
   ```xml
   <!-- Use Spring Boot's layered JAR for Lambda -->
   <plugin>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-maven-plugin</artifactId>
   </plugin>
   ```

2. **Use GraalVM native image** — compiles Spring Boot to a native binary, eliminating JVM startup. Cold start drops from 10s to ~200ms.

3. **Provisioned Concurrency** — AWS keeps N execution environments pre-warmed at all times. You pay for the reserved capacity even when not in use.
   ```
   aws lambda put-provisioned-concurrency-config \
     --function-name my-function \
     --qualifier prod \
     --provisioned-concurrent-executions 5
   ```

4. **Lambda SnapStart** (Java 11+) — AWS takes a snapshot of the initialized execution environment and restores it for new instances. Reduces cold starts to ~1s for Spring Boot.

5. **Avoid Spring Boot for Lambdas** — consider lightweight alternatives (Micronaut, Quarkus, or plain Java) for latency-sensitive Lambda functions.

---

## Trap 3: S3 Eventual Consistency (Now Strong Consistency Since 2020)

**Historical gotcha (pre-December 2020):**
- New object PUT: eventual consistency — a LIST immediately after PUT might not show the new object
- Overwrite PUT / DELETE: eventual consistency — you might read the old version immediately after overwrite

**Since December 2020:** AWS S3 provides **strong read-after-write consistency** for all operations — GETs, PUTs, DELETEs, and LIST operations.

```
Before Dec 2020:
  PUT s3://bucket/file.txt
  GET s3://bucket/file.txt  <- might return 404 or old version!

After Dec 2020:
  PUT s3://bucket/file.txt
  GET s3://bucket/file.txt  <- guaranteed to return the new version
```

This is a common trap question — interviewers test whether you know the old behavior and whether you know it changed. The correct answer: "S3 is now strongly consistent as of December 2020."

---

## Trap 4: RDS Multi-AZ vs Read Replicas — Different Purposes

| Aspect | Multi-AZ | Read Replica |
|---|---|---|
| Purpose | **High availability** | **Read scaling / analytics** |
| Replication | Synchronous | Asynchronous |
| Standby | Not readable (standby only) | Readable |
| Failover | Automatic, ~1-2 min | Manual promotion |
| Cost | ~2x (two instances) | Per replica |
| Use when | You need HA / DR | You have read-heavy workloads |

**Multi-AZ:** one primary, one synchronous standby in a different AZ. If the primary fails, AWS automatically fails over to the standby. Zero data loss (synchronous). The standby is NOT used for reads — its sole purpose is failover.

**Read Replica:** an asynchronous copy of the primary. You can direct your read queries (`SELECT`) to a replica to offload the primary. Lag can be seconds to minutes. Not a failover target automatically (you must manually promote).

**Common mistake:** "I'll use a Read Replica for high availability." Wrong — Multi-AZ is for HA. A Read Replica is for performance scaling.

---

## Trap 5: SQS At-Least-Once vs Exactly-Once Delivery

**Standard Queue:**
- **At-least-once delivery** — a message may be delivered more than once (if the consumer processes it but crashes before deleting it, or due to distributed system internals)
- High throughput, best-effort ordering
- Your consumers MUST be idempotent

```java
// Idempotent consumer — safe to process twice
@SqsListener("order-processing-queue")
public void processOrder(OrderMessage msg) {
    if (orderRepository.existsById(msg.getOrderId())) {
        log.info("Order {} already processed, skipping", msg.getOrderId());
        return;  // idempotency check
    }
    orderService.process(msg);
}
```

**FIFO Queue:**
- **Exactly-once processing** — AWS deduplicates messages within a 5-minute window
- Strict ordering within a message group
- Lower throughput (3,000 msg/s with batching vs nearly unlimited for standard)
- Higher cost

Use FIFO when: ordering matters (financial transactions) or duplicate processing is catastrophic. Use Standard when: throughput matters and your application is idempotent.

---

## Trap 6: IAM Role vs IAM User — Which Should Your EC2 Use?

**Never use IAM User credentials (access key + secret) on an EC2 instance.**

```java
// BAD — hardcoded or file-based credentials on EC2
AmazonS3 s3 = AmazonS3ClientBuilder.standard()
    .withCredentials(new AWSStaticCredentialsProvider(
        new BasicAWSCredentials("AKIAIOSFODNN7EXAMPLE", "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY")
    ))
    .build();
// If someone gets into this EC2, they have your keys permanently
// Keys don't rotate automatically
// Keys are visible in code/config files
```

**Always use IAM Roles:**
```java
// GOOD — no credentials in code; SDK automatically fetches temporary credentials
// from the EC2 instance metadata service (169.254.169.254)
AmazonS3 s3 = AmazonS3ClientBuilder.standard()
    // No credentials specified — SDK uses instance role automatically
    .withRegion("us-east-1")
    .build();
```

IAM Roles provide:
- **Temporary credentials** — auto-rotated every ~6 hours by AWS
- **No secret management** — no credentials to leak, store, or rotate manually
- **Principle of least privilege** — grant only the S3 permissions this instance needs
- **Audit trail** — CloudTrail logs every API call with the role ARN

---

## Trap 7: VPC Security Group vs NACL — Stateful vs Stateless

| Aspect | Security Group | NACL (Network ACL) |
|---|---|---|
| Level | Instance/ENI level | Subnet level |
| State | **Stateful** | **Stateless** |
| Rules | Allow only (no deny) | Allow and Deny |
| Rule evaluation | All rules evaluated | Rules evaluated in order (number) |
| Return traffic | Automatically allowed | Must explicitly allow in both directions |

**Stateful (Security Group):**
If you allow inbound port 443, the response traffic is automatically allowed outbound — you don't need to add an outbound rule.

**Stateless (NACL):**
You must explicitly allow both inbound AND outbound for a conversation to work. If you allow inbound port 443 but forget the outbound ephemeral ports (1024-65535), responses are blocked.

**Typical setup:**
- Security Groups: fine-grained control per instance. EC2 in app tier can talk to RDS on port 5432; nothing else can.
- NACLs: additional coarse-grained layer. Block entire IP ranges (known malicious subnets). Most teams leave NACLs at "allow all" and rely on Security Groups.

---

## Trap 8: CloudWatch Metrics vs Logs vs Alarms

**CloudWatch Metrics:**
- Numerical time-series data: CPU%, memory, request count, latency
- Retained for 15 months
- Used as the data source for Alarms and dashboards
- Custom metrics from your app:
  ```java
  cloudWatch.putMetricData(new PutMetricDataRequest()
      .withNamespace("MyApp")
      .withMetricData(new MetricDatum()
          .withMetricName("OrderProcessingTime")
          .withValue((double) processingTimeMs)
          .withUnit(StandardUnit.Milliseconds)));
  ```

**CloudWatch Logs:**
- Raw log streams from your application, Lambda, ECS, etc.
- Log Groups → Log Streams → Log Events
- Log Insights: query language for searching logs
- Metric Filters: extract metrics FROM log patterns
  ```
  # Extract HTTP 5xx count from ALB access logs
  [ip, date, method, uri, status=5*, ...]
  ```

**CloudWatch Alarms:**
- Watches a metric; triggers action when threshold crossed
- Actions: send SNS notification, trigger Auto Scaling, stop/terminate an EC2
- States: OK, ALARM, INSUFFICIENT_DATA

They work together: Logs generate Metrics (via Metric Filters); Alarms watch Metrics; Alarms notify via SNS → Lambda / PagerDuty.

---

## Trap 9: Spot Instances — When NOT to Use Them

Spot instances are spare EC2 capacity at 70-90% discount but can be **terminated by AWS with 2 minutes notice** when the capacity is needed back.

**Good use cases:**
- Batch processing jobs (can be checkpointed and restarted)
- Big data processing (Spark/EMR)
- CI/CD build workers (stateless, short-lived)
- Dev/test environments

**Never use for:**
- Databases (termination = data loss or corruption)
- Stateful services where in-flight transactions can't be interrupted
- Primary production web servers (unless mixed with On-Demand for baseline)
- Any workload that can't tolerate unexpected 2-minute interruption

**Safe pattern: Spot + On-Demand mixed fleet**
```
Auto Scaling Group:
  On-Demand base: 2 instances  <- always available
  Spot for scaling: up to 8 instances  <- 70% cheaper, but interruptible
```

---

## Trap 10: DynamoDB Partition Key Design and Hot Partition Problem

DynamoDB distributes data across partitions using the **partition key** hash. Each partition has a throughput ceiling (~3,000 RCU and 1,000 WCU per second).

**Hot partition problem:** if many requests hit the same partition key, that partition becomes overwhelmed while others are idle.

```
BAD partition key: "status" with values "ACTIVE" or "INACTIVE"
  -> 90% of requests hit "ACTIVE" partition = hot partition = throttling

BAD partition key: sequential IDs (1, 2, 3, 4...)
  -> all new writes go to the latest partition = hot write partition

GOOD partition key: userId (or orderId, or customerId)
  -> requests distributed across millions of unique users = even distribution

Pattern for time-series data:
  BAD:  PK="events" (all events in one partition)
  GOOD: PK="events#2024-01#shard-3" (write sharding across multiple partitions)
        -> randomly assign shard (0-9) at write time
        -> query all shards and merge at read time
```

**Write sharding:**
```java
// Add random shard suffix to distribute hot writes
String partitionKey = "events#" + yearMonth + "#shard-" + ThreadLocalRandom.current().nextInt(10);
```

A well-designed partition key is the most important DynamoDB design decision. Get it wrong and no amount of provisioned throughput will fix the hot partition problem.

---

## Trap 11: What Is the Difference Between AWS Parameter Store and Secrets Manager?

| Aspect | Parameter Store | Secrets Manager |
|---|---|---|
| Cost | Free (Standard tier) | $0.40/secret/month + $0.05/10K API calls |
| Auto-rotation | No (must implement manually) | Yes (built-in rotation for RDS, Redshift, etc.) |
| Secret versioning | Yes | Yes |
| Encryption | Optional (SecureString = KMS) | Always encrypted with KMS |
| Hierarchy | Yes (/prod/myapp/db-password) | Flat |
| Use case | App configs + simple secrets | Database credentials, API keys needing rotation |

**Rule of thumb:**
- Database passwords → Secrets Manager (auto-rotation prevents stale credentials)
- Feature flags, app config, non-sensitive params → Parameter Store (free, organized)
- API keys without rotation → either works, Parameter Store is cheaper
