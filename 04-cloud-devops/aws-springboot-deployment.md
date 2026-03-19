# AWS Spring Boot Deployment Guide

> Deploying Spring Boot applications to AWS using ECS, EKS, Elastic Beanstalk, and Lambda.

---

## Deployment Options for Spring Boot on AWS

```
┌─────────────────────────────────────────────────────────────────────┐
│  Deployment Options                                                   │
│                                                                      │
│  EC2 (manual)     → Full control, high maintenance                   │
│  Elastic Beanstalk → Managed PaaS, easy deployment                   │
│  ECS (Fargate)    → Serverless containers, no EC2 management         │
│  EKS              → Kubernetes, complex but powerful                  │
│  Lambda           → Serverless functions, cold start latency          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## ECS Fargate Deployment (Recommended for Most Use Cases)

### Architecture

```
Internet
    │
    ▼
Route 53 (DNS)
    │
    ▼
Application Load Balancer (ALB)
    │   ┌─ HTTPS termination
    │   └─ Health checks
    ▼
ECS Service
  ├── Task 1 [Spring Boot Container + Sidecar]
  ├── Task 2 [Spring Boot Container + Sidecar]
  └── Task 3 [Spring Boot Container + Sidecar]
    │
    ▼
VPC Private Subnet
  ├── RDS PostgreSQL (Multi-AZ)
  ├── ElastiCache Redis
  └── Secrets Manager (DB credentials)
```

### Task Definition

```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/myapp:latest",
      "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
      "environment": [
        {"name": "SPRING_PROFILES_ACTIVE", "value": "prod"}
      ],
      "secrets": [
        {
          "name": "SPRING_DATASOURCE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:myapp/db-password"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,
        "timeout": 10,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

---

## Spring Boot Configuration for AWS

```yaml
# application-prod.yml
spring:
  datasource:
    # Use RDS endpoint
    url: jdbc:postgresql://${RDS_ENDPOINT}:5432/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}  # Injected from Secrets Manager
    hikari:
      maximum-pool-size: 10
      connection-timeout: 30000

  redis:
    # ElastiCache endpoint
    host: ${REDIS_ENDPOINT}
    port: 6379

cloud:
  aws:
    region:
      static: us-east-1
    credentials:
      use-default-provider-chain: true  # Uses IAM task role

management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
```

---

## ECR — Elastic Container Registry

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin ACCOUNT.dkr.ecr.us-east-1.amazonaws.com

# Tag and push image
docker tag myapp:latest ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Deploy to ECS (trigger new deployment)
aws ecs update-service \
    --cluster myapp-cluster \
    --service myapp-service \
    --force-new-deployment
```

---

## Auto Scaling for ECS Service

```json
{
  "AutoScalingGroupName": "myapp-ecs-service",
  "MinCapacity": 2,
  "MaxCapacity": 20,
  "TargetTrackingScalingPolicies": [
    {
      "TargetValue": 70.0,
      "PredefinedMetricSpecification": {
        "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
      }
    },
    {
      "TargetValue": 1000.0,
      "CustomizedMetricSpecification": {
        "MetricName": "RequestCountPerTarget",
        "Namespace": "AWS/ApplicationELB"
      }
    }
  ]
}
```

---

## Key AWS Services for Spring Boot Backend

| Service | Purpose |
|---------|---------|
| ECS/EKS | Container orchestration |
| ECR | Container image registry |
| RDS | Managed relational DB (PostgreSQL, MySQL) |
| ElastiCache | Managed Redis/Memcached |
| Secrets Manager | Secure credentials storage |
| Parameter Store | Configuration values |
| SQS | Message queue (async processing) |
| SNS | Pub/sub notifications |
| ALB | Load balancer with health checks |
| CloudWatch | Logging, monitoring, alarms |
| X-Ray | Distributed tracing |
| IAM | Authentication and authorization |
| VPC | Network isolation |

---

## IAM Roles for ECS Tasks (Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:myapp/*"
    },
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage", "sqs:ReceiveMessage", "sqs:DeleteMessage"],
      "Resource": "arn:aws:sqs:us-east-1:ACCOUNT:myapp-queue"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::myapp-uploads/*"
    }
  ]
}
```

Never use long-lived access keys in containers. Always use IAM task roles.

---

## Spring Boot + SQS Integration

```java
@Component
public class OrderEventListener {

    @SqsListener("${sqs.order-events.url}")
    public void processOrderEvent(@Payload String messageBody,
                                   @Header("MessageId") String messageId) {
        log.info("Processing order event: {}", messageId);
        OrderEvent event = objectMapper.readValue(messageBody, OrderEvent.class);
        orderProcessor.process(event);
    }
}

@Service
public class OrderEventPublisher {

    @Autowired
    private SqsTemplate sqsTemplate;

    @Value("${sqs.order-events.url}")
    private String queueUrl;

    public void publishOrderCreated(Order order) {
        sqsTemplate.send(queueUrl, new OrderCreatedEvent(order.getId(), order.getTotal()));
    }
}
```
