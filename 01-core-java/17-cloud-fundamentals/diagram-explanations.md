# Cloud Fundamentals — Diagram Explanations

---

## 1. IaaS vs PaaS vs SaaS — Responsibility Model

```
What YOU manage vs What the CLOUD manages:

                  IaaS         PaaS         SaaS
                (EC2)        (Beanstalk)   (Gmail)
Application      YOU          YOU          CLOUD
Data             YOU          YOU          CLOUD
Runtime          YOU          CLOUD        CLOUD
Middleware       YOU          CLOUD        CLOUD
OS               YOU          CLOUD        CLOUD
Virtualization   CLOUD        CLOUD        CLOUD
Servers          CLOUD        CLOUD        CLOUD
Storage          CLOUD        CLOUD        CLOUD
Networking       CLOUD        CLOUD        CLOUD

IaaS: Most control, most responsibility
PaaS: Focus on your app and data
SaaS: Just use it
```

---

## 2. AWS 3-Tier Architecture (ALB → EC2/ECS → RDS)

```
Internet
    |
    | (HTTPS 443)
    v
[Application Load Balancer]
    |  (port 8080, health check /actuator/health)
    |  Round-robin or least-connections
    |
    +--- EC2/ECS Instance 1 (Spring Boot)
    +--- EC2/ECS Instance 2 (Spring Boot)   <- Auto Scaling Group
    +--- EC2/ECS Instance 3 (Spring Boot)
    |
    | (port 5432 — internal only)
    v
[RDS PostgreSQL]
    |
    +--- Primary (AZ us-east-1a)  <- reads + writes
    +--- Standby (AZ us-east-1b)  <- hot standby (Multi-AZ)

[ElastiCache Redis]
    |
    +--- Session cache / query cache

Network:
  ALB in PUBLIC subnets (has internet-facing IP)
  EC2/ECS in PRIVATE subnets (no direct internet access)
  RDS in PRIVATE subnets (accessible only from app tier)
  NAT Gateway for EC2 to reach internet for updates/external APIs
```

---

## 3. Auto-Scaling Group Behavior

```
Normal load (3 instances):
  [Instance 1]  [Instance 2]  [Instance 3]
   CPU: 40%      CPU: 45%      CPU: 42%

CPU spike (e.g., traffic surge):
  [Instance 1]  [Instance 2]  [Instance 3]
   CPU: 80%      CPU: 85%      CPU: 82%
                    |
          CloudWatch Alarm triggered
          (CPU > 70% for 2 consecutive minutes)
                    |
          Auto Scaling launches new instances
                    |
  [Instance 1]  [Instance 2]  [Instance 3]  [Instance 4]  [Instance 5]
   CPU: 50%      CPU: 48%      CPU: 52%      CPU: 45%       CPU: 47%

Traffic drops (e.g., overnight):
  All instances CPU < 30% for 10 minutes
                    |
          Scale-in event: terminate 2 instances
          (always terminates least recently used by default)
                    |
  [Instance 1]  [Instance 2]  [Instance 3]  <- back to desired capacity

Configuration:
  Minimum: 2      <- never go below 2 (HA guarantee)
  Desired:  3
  Maximum: 10     <- never exceed 10 (cost control)
  Scale-out: CPU > 70% for 2 min -> add 2 instances
  Scale-in:  CPU < 30% for 10 min -> remove 1 instance
  Cooldown: 300s  <- wait 5 min before next scaling action
```

---

## 4. VPC With Public/Private Subnets and NAT Gateway

```
Region: us-east-1
VPC: 10.0.0.0/16
+------------------------------------------------------------+
|                                                            |
|  Availability Zone: us-east-1a     AZ: us-east-1b         |
|  +---------------------------+  +---------------------------+ |
|  | PUBLIC SUBNET             |  | PUBLIC SUBNET             | |
|  | 10.0.1.0/24               |  | 10.0.2.0/24               | |
|  | [ALB]  [NAT Gateway]      |  | [ALB]  [Bastion Host]     | |
|  +---------------------------+  +---------------------------+ |
|              |                              |                |
|  +---------------------------+  +---------------------------+ |
|  | PRIVATE SUBNET (App)      |  | PRIVATE SUBNET (App)      | |
|  | 10.0.3.0/24               |  | 10.0.4.0/24               | |
|  | [ECS Task] [ECS Task]     |  | [ECS Task] [ECS Task]     | |
|  +---------------------------+  +---------------------------+ |
|              |                              |                |
|  +---------------------------+  +---------------------------+ |
|  | PRIVATE SUBNET (DB)       |  | PRIVATE SUBNET (DB)       | |
|  | 10.0.5.0/24               |  | 10.0.6.0/24               | |
|  | [RDS Primary]             |  | [RDS Standby]             | |
|  +---------------------------+  +---------------------------+ |
+------------------------------------------------------------+
        |
   Internet Gateway
        |
   Internet

Traffic flow:
  Inbound: Internet -> IGW -> ALB (public subnet) -> ECS (private subnet)
  Outbound from ECS: ECS -> NAT Gateway (public subnet) -> IGW -> Internet
  DB: No internet access. Only ECS can connect (security group rules).

  NAT Gateway: allows private subnet resources to initiate outbound
               connections (e.g., pull Docker images, call external APIs)
               but PREVENTS inbound connections from internet
```

---

## 5. S3 + CloudFront CDN for Static Assets

```
                         [S3 Bucket]
                         (origin)
                         index.html
                         main.js
                         styles.css
                         images/logo.png
                              |
              [CloudFront Distribution]
              (CDN with edge locations)
                    /        |        \
              /             |             \
  [Edge: Frankfurt]  [Edge: Singapore]  [Edge: São Paulo]
   TTL: 24 hours      TTL: 24 hours      TTL: 24 hours

User in Frankfurt requests main.js:
  1. Browser -> CloudFront Frankfurt edge
  2. Edge checks cache: HIT -> return immediately (latency: ~5ms)
     Edge checks cache: MISS -> fetch from S3 in us-east-1 (latency: ~200ms)
                                Cache for TTL, then serve from cache

Benefits:
  - Files served from nearest edge (~5ms) vs distant S3 (~200ms)
  - Reduces load on origin S3
  - CloudFront handles DDoS (absorbs at edge)
  - HTTPS termination at edge (free SSL cert via ACM)
  - Cache invalidation when you deploy new version: aws cloudfront create-invalidation --paths "/*"
```

---

## 6. SQS → Lambda Event-Driven Processing

```
PRODUCER SERVICE                 SQS QUEUE               LAMBDA CONSUMER
(Spring Boot API)                                         (processes messages)

POST /orders                     Standard Queue           Lambda function
  |                              (or FIFO for ordering)      |
  | -> publish message           +-------------------+        |
  |    {orderId, items, userId}  | MSG 1             |        |
  |                              | MSG 2             |------->| process order
  |    HTTP 202 Accepted         | MSG 3             |        | charge payment
  |    (async — don't wait)      | MSG 4             |        | send email
                                 | MSG 5 (invisible) |<---    | update inventory
                                 +-------------------+    |   |
                                                          |   | if success:
                                                          |   | delete message
                                                          |   | if failure:
                                                          |-->| message becomes
                                                              | visible again
                                                              | (retry)

Dead Letter Queue (DLQ):
  After maxReceiveCount failures (e.g., 3), message moved to DLQ
  Team gets alerted, investigates the failed message

Benefits of SQS:
  - Producer never waits for order processing
  - Consumer can scale independently
  - Messages survive consumer crashes (visibility timeout)
  - Auto-scales Lambda based on queue depth
  - At-least-once delivery (standard queue)
  - Exactly-once + ordering (FIFO queue)
```
