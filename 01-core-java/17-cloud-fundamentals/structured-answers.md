# Cloud Fundamentals — Structured Answers

---

## Q1: What Is the Difference Between EC2, ECS, and EKS?

**EC2 (Elastic Compute Cloud):**
Raw virtual machines. You control everything: OS, patching, JVM installation, security hardening, Docker runtime. Use when you have legacy apps that can't be containerized, or when you need full control over the OS configuration.

**ECS (Elastic Container Service):**
AWS's native container orchestration platform. You define Task Definitions (CPU, memory, Docker image, env vars) and Services (desired count, load balancer attachment). AWS handles scheduling containers onto EC2 instances or Fargate. Much less operational overhead than EC2 for containerized apps.

```json
// ECS Task Definition (simplified)
{
  "family": "spring-boot-api",
  "containerDefinitions": [{
    "name": "api",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3",
    "portMappings": [{"containerPort": 8080}],
    "environment": [
      {"name": "SPRING_PROFILES_ACTIVE", "value": "prod"}
    ],
    "secrets": [
      {"name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:..."}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {"awslogs-group": "/ecs/spring-boot-api", "awslogs-region": "us-east-1"}
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"],
      "interval": 30, "timeout": 5, "retries": 3
    }
  }],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512", "memory": "1024",
  "networkMode": "awsvpc"
}
```

**EKS (Elastic Kubernetes Service):**
Managed Kubernetes. AWS manages the control plane (API server, etcd, scheduler). You manage worker nodes (or use Fargate). Use when your organization has Kubernetes expertise, needs multi-cloud portability, or requires Kubernetes-native tooling (Helm, Istio, Argo CD).

**Decision guide:**
- New containerized app, AWS-only, simplicity matters → ECS Fargate
- Existing Kubernetes deployments, large org, multi-cloud → EKS
- Legacy app, specific OS requirements → EC2

---

## Q2: How Does AWS Lambda Work and What Are Its Limitations?

Lambda executes your code in response to triggers (HTTP via API Gateway, S3 events, SQS messages, CloudWatch cron, etc.). AWS provides and manages the execution environment; you only write the function.

```java
// Spring Boot Lambda with AWS Lambda Web Adapter
@SpringBootApplication
public class LambdaApplication {
    public static void main(String[] args) {
        SpringApplication.run(LambdaApplication.class, args);
    }
}

// Or using aws-serverless-java-container for Spring Boot
public class StreamLambdaHandler implements RequestStreamHandler {
    private static final SpringBootLambdaContainerHandler<AwsProxyRequest, AwsProxyResponse> handler;

    static {
        try {
            handler = SpringBootLambdaContainerHandler.getAwsProxyHandler(Application.class);
        } catch (ContainerInitializationException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void handleRequest(InputStream in, OutputStream out, Context context) throws IOException {
        handler.proxyStream(in, out, context);
    }
}
```

**Lambda limitations (hard limits):**
| Limit | Value |
|---|---|
| Max execution time | 15 minutes |
| Memory | 128MB – 10GB |
| Deployment package | 50MB zipped, 250MB unzipped |
| Container image | 10GB |
| Concurrent executions | 1,000 per region (soft limit, increasable) |
| Ephemeral disk (/tmp) | 512MB – 10GB |
| Payload (request/response) | 6MB (synchronous), 256KB (async) |

**Lambda is NOT suitable for:**
- Long-running background jobs (> 15 min) — use ECS/Fargate or Step Functions
- Persistent connections (database connection pooling is wasteful)
- Workloads requiring more than 10GB memory
- Ultra-low-latency requirements where cold starts are unacceptable
- Stateful applications

---

## Q3: What Is the Difference Between RDS Multi-AZ and Read Replicas?

**Multi-AZ — for High Availability:**
```
                    [Application]
                         |
               DNS: mydb.cluster.rds.amazonaws.com
                         |
                    [Primary DB]  ---- synchronous replication ---> [Standby DB]
                   (AZ: us-east-1a)                                (AZ: us-east-1b)

Primary fails:
  1. RDS detects failure (monitoring heartbeat)
  2. Promotes standby to primary (60-120 seconds)
  3. Updates DNS endpoint to point to new primary
  4. Application reconnects to same endpoint — no code change needed
```

- Synchronous replication = zero data loss on failover
- Standby is NOT accessible for reads — it's a hot standby only
- Automatic failover managed by AWS
- Cost: ~2x (two instances running)

**Read Replicas — for Read Scaling:**
```
                    [Application]
                    /            \
             writes              reads
               |                   |
          [Primary DB]       [Read Replica 1]
          (us-east-1a)       [Read Replica 2]  <- up to 5 replicas (RDS)
                             [Read Replica 3]  <- up to 15 (Aurora)

Replication: asynchronous (replica may lag by seconds)
```

- Asynchronous replication = small potential for stale reads
- Replicas ARE readable — offload reporting, analytics, read-heavy queries
- Manual promotion needed for DR (not automatic)
- Can be in different regions (cross-region read replicas for global apps)

**Use both together:**
- Multi-AZ on Primary: for HA
- Read Replica(s): for read scaling
- Multi-AZ on Replica: replica itself can be Multi-AZ for extra HA

---

## Q4: How Do You Design a Spring Boot Deployment on AWS ECS?

```
Architecture:
  GitHub -> CodePipeline -> CodeBuild -> ECR -> ECS Service -> ALB

Step-by-step:

1. Containerize the application
   Dockerfile:
   FROM eclipse-temurin:21-jre-alpine
   WORKDIR /app
   COPY target/myapp.jar app.jar
   EXPOSE 8080
   ENTRYPOINT ["java", "-XX:+UseContainerSupport", \
               "-XX:MaxRAMPercentage=75.0", \
               "-jar", "app.jar"]

2. Push to Amazon ECR
   aws ecr create-repository --repository-name myapp
   docker build -t myapp:v1.0 .
   docker tag myapp:v1.0 <account>.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0
   docker push <account>.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0

3. Create Task Definition (JSON above in Q1)

4. Create ECS Service with ALB
   aws ecs create-service \
     --cluster prod-cluster \
     --service-name myapp-service \
     --task-definition myapp:1 \
     --desired-count 3 \
     --launch-type FARGATE \
     --load-balancers targetGroupArn=...,containerName=api,containerPort=8080 \
     --network-configuration "awsvpcConfiguration={subnets=[subnet-private-1,subnet-private-2],securityGroups=[sg-app]}"

5. Configure Application Load Balancer
   - Listener: HTTPS 443 -> Target Group (health check: /actuator/health)
   - HTTP 80 -> redirect to HTTPS

6. Set up Auto Scaling
   aws application-autoscaling register-scalable-target \
     --service-namespace ecs \
     --scalable-dimension ecs:service:DesiredCount \
     --resource-id service/prod-cluster/myapp-service \
     --min-capacity 2 --max-capacity 10

   aws application-autoscaling put-scaling-policy \
     --policy-type TargetTrackingScaling \
     --policy-name cpu-tracking \
     --target-tracking-scaling-policy-configuration \
       '{
         "TargetValue": 70.0,
         "PredefinedMetricSpecification": {"PredefinedMetricType": "ECSServiceAverageCPUUtilization"}
       }'
```

**Spring Boot application.yml for ECS:**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/${DB_NAME:mydb}
    username: ${DB_USER}
    password: ${DB_PASSWORD}   # from Secrets Manager via ECS task definition
  jpa:
    hibernate.ddl-auto: validate  # never "create" or "create-drop" in prod

management:
  endpoints:
    web.exposure.include: health,info,metrics,prometheus
  endpoint.health.show-details: when_authorized
```

---

## Q5: What Is an IAM Role and Why Should EC2 Instances Use Roles Instead of Credentials?

An IAM Role is an AWS identity with a set of permissions. Unlike an IAM user, a role has no long-term credentials (no access key/secret key). Instead, AWS provides temporary security credentials to whoever assumes the role — valid for 15 minutes to 12 hours, then auto-rotated.

```json
// IAM Role Policy — least privilege for a Spring Boot app
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": ["sqs:ReceiveMessage", "sqs:DeleteMessage", "sqs:GetQueueAttributes"],
      "Resource": "arn:aws:sqs:us-east-1:123456789:order-queue"
    },
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/myapp/*"
    }
  ]
}
```

```java
// In your Spring Boot app — no credentials anywhere
@Service
public class S3Service {
    // AWS SDK automatically discovers credentials from:
    // 1. Environment variables (for local dev)
    // 2. EC2 instance metadata (for ECS/EC2 with role)
    // 3. ~/.aws/credentials (for local dev)
    private final S3Client s3Client = S3Client.builder()
        .region(Region.US_EAST_1)
        .build();   // no credentials specified

    public void uploadFile(String bucket, String key, InputStream data) {
        s3Client.putObject(
            PutObjectRequest.builder().bucket(bucket).key(key).build(),
            RequestBody.fromInputStream(data, data.available())
        );
    }
}
```

**Why roles > IAM user credentials on EC2:**
1. No long-term secrets to leak, rotate, or store
2. Temporary creds auto-rotate every 6 hours — stolen creds expire quickly
3. If an EC2 is compromised, the blast radius is limited to the role's permissions
4. No secrets in code, config files, or environment variables that persist after the instance terminates
5. CloudTrail logs show which role performed each action — better audit trail

---

## Q6: How Does AWS SQS Guarantee Message Delivery?

**Persistence:** Messages are stored redundantly across multiple AZs within the region. A message is not lost unless you explicitly delete it.

**Visibility Timeout:** When a consumer reads a message, it becomes invisible to other consumers for the visibility timeout period (default 30s, max 12h). The consumer must delete the message after processing. If the consumer crashes without deleting, the message becomes visible again and can be re-processed by another consumer.

```java
@Configuration
public class SqsConfig {
    @Bean
    public QueueMessagingTemplate queueMessagingTemplate(AmazonSQSAsync amazonSQS) {
        return new QueueMessagingTemplate(amazonSQS);
    }
}

@Component
public class OrderConsumer {
    @SqsListener(value = "order-queue", deletionPolicy = SqsMessageDeletionPolicy.ON_SUCCESS)
    public void processOrder(OrderMessage order, Acknowledgment ack) {
        try {
            orderService.process(order);
            ack.acknowledge();  // explicit delete after successful processing
        } catch (TransientException e) {
            // Don't ack — message becomes visible again after visibility timeout
            // SQS will retry
        } catch (PermanentException e) {
            ack.acknowledge();  // ack to remove from main queue
            deadLetterService.send(order);  // send to DLQ manually
        }
    }
}
```

**Dead Letter Queue (DLQ):**
After `maxReceiveCount` failures, SQS moves the message to a DLQ. Set up a CloudWatch Alarm on DLQ message count to alert your team.

**Long Polling:** instead of returning immediately when the queue is empty (wasting API calls), the consumer waits up to 20 seconds for a message to arrive. Reduces cost and improves latency.

---

## Q7: What Is the Difference Between a VPC Security Group and NACL?

```java
// Security Group (enforced at ENI/instance level)
// Rule: allow inbound TCP 8080 from ALB security group
// STATEFUL: response traffic is automatically allowed
// Result: any traffic matching this rule is allowed; everything else is implicitly denied

// NACL (enforced at subnet boundary)
// STATELESS: must explicitly allow BOTH directions
// Inbound rule 100: ALLOW TCP 8080 from 0.0.0.0/0
// Outbound rule 100: ALLOW TCP 1024-65535 to 0.0.0.0/0  <- ephemeral ports for responses!
// If you forget the outbound rule, responses are blocked even though inbound is allowed
```

**Practical configuration:**

```
NACL for private subnet (app tier):
  Inbound:
    100 ALLOW TCP 8080 from 10.0.1.0/24 (public subnet with ALB)
    110 ALLOW TCP 1024-65535 from 0.0.0.0/0 (responses to outbound connections)
    * DENY all other inbound
  Outbound:
    100 ALLOW TCP 5432 to 10.0.5.0/24 (DB subnet)
    110 ALLOW TCP 443 to 0.0.0.0/0 (HTTPS to internet via NAT)
    120 ALLOW TCP 1024-65535 to 10.0.1.0/24 (responses to ALB)
    * DENY all other outbound

Security Group for ECS Task:
  Inbound: TCP 8080 from ALB security group
  Outbound: TCP 5432 to RDS security group
  Outbound: TCP 443 to 0.0.0.0/0 (HTTPS calls to external APIs)
  (No need to explicitly allow response traffic — stateful)
```

---

## Q8: How Do You Implement Distributed Tracing in an AWS Microservices Architecture?

```java
// Spring Boot with AWS X-Ray

// 1. Add dependency
// io.opentelemetry:opentelemetry-api
// io.opentelemetry:opentelemetry-sdk
// com.amazonaws:aws-xray-recorder-sdk-spring (or OTEL X-Ray exporter)

// 2. Propagate trace context across services
@Configuration
public class TracingConfig {
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate rt = new RestTemplate();
        rt.getInterceptors().add((request, body, execution) -> {
            // Inject X-Ray trace ID into HTTP headers for downstream services
            AWSXRay.getCurrentSegment().ifPresent(seg -> {
                request.getHeaders().set("X-Amzn-Trace-Id",
                    "Root=" + seg.getTraceId() + ";Parent=" + seg.getId() + ";Sampled=1");
            });
            return execution.execute(request, body);
        });
        return rt;
    }
}

// 3. Annotate service with tracing
@Service
public class OrderService {
    @XRayEnabled  // creates a subsegment for this method
    public Order placeOrder(OrderRequest req) {
        AWSXRay.beginSubsegment("validate-order");
        try {
            validate(req);
        } finally {
            AWSXRay.endSubsegment();
        }

        AWSXRay.beginSubsegment("save-order");
        try {
            return orderRepo.save(new Order(req));
        } finally {
            AWSXRay.endSubsegment();
        }
    }
}
```

**With OpenTelemetry (modern, vendor-neutral):**
```yaml
# Use ADOT (AWS Distro for OpenTelemetry) sidecar container in ECS
# No code changes needed — OTEL agent instruments Spring Boot automatically

# ECS Task Definition — add ADOT sidecar
containers:
  - name: adot-collector
    image: public.ecr.aws/aws-observability/aws-otel-collector:latest
    environment:
      - name: AOT_CONFIG_CONTENT
        value: |
          receivers:
            otlp:
              protocols: {grpc: {endpoint: "0.0.0.0:4317"}}
          exporters:
            awsxray: {}
            awsemf: {}
          service:
            pipelines:
              traces: {receivers: [otlp], exporters: [awsxray]}
              metrics: {receivers: [otlp], exporters: [awsemf]}
```

X-Ray Service Map: shows all microservices, their dependencies, latency, and error rates as a visual graph. Essential for finding which service is causing a latency spike in a chain of calls.

---

## Q9: What Is Auto-Scaling and How Do You Configure It for a Spring Boot App?

Auto-scaling adjusts the number of running instances based on load. For ECS, this means adding or removing Fargate tasks.

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/production/spring-boot-api \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 20

# Target tracking: maintain 70% CPU utilization
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/production/spring-boot-api \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'

# Custom metric: scale on ALB request count per target
aws application-autoscaling put-scaling-policy \
  --policy-name alb-requests-per-target \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 1000.0,
    "CustomizedMetricSpecification": {
      "MetricName": "RequestCountPerTarget",
      "Namespace": "AWS/ApplicationELB",
      "Dimensions": [{"Name": "TargetGroup", "Value": "targetgroup/my-tg/abc123"}],
      "Statistic": "Sum"
    }
  }'
```

**Spring Boot: expose metrics for CloudWatch Custom Metrics:**
```java
@Configuration
public class MetricsConfig {
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> cloudWatchCustomizer(
            CloudWatchConfig cwConfig) {
        return registry -> {
            // Custom business metric: active orders being processed
            Gauge.builder("app.orders.processing", orderService, OrderService::getActiveCount)
                .description("Orders currently being processed")
                .register(registry);
        };
    }
}
```

**Scale-in protection:** mark ECS tasks as protected during long operations to prevent them from being terminated mid-process:
```java
// Set scale-in protection before long operation
ecsClient.updateTaskProtection(req -> req
    .cluster(clusterName)
    .tasks(getCurrentTaskArn())
    .protectionEnabled(true)
    .expiresInMinutes(30)
);
try {
    performLongRunningOperation();
} finally {
    ecsClient.updateTaskProtection(req -> req
        .cluster(clusterName).tasks(getCurrentTaskArn()).protectionEnabled(false));
}
```

---

## Q10: How Do You Manage Secrets in AWS — Parameter Store vs Secrets Manager?

**AWS Parameter Store (simple, free):**
```java
@Configuration
public class ParameterStoreConfig {
    @Bean
    public SsmClient ssmClient() {
        return SsmClient.builder().region(Region.US_EAST_1).build();
    }
}

@Service
public class ConfigService {
    @Autowired private SsmClient ssm;

    public String getParam(String name) {
        return ssm.getParameter(req -> req.name(name).withDecryption(true))
            .parameter().value();
    }
}

// Or Spring Cloud AWS — auto-loads parameters as Spring properties
// /config/myapp/spring.datasource.url -> spring.datasource.url property
```

```yaml
# application.yml
spring:
  cloud:
    aws:
      paramstore:
        prefix: /config
        default-context: myapp
        enabled: true
```

**AWS Secrets Manager (with auto-rotation):**
```java
@Configuration
public class SecretsManagerConfig {
    @Bean
    @RefreshScope  // allows re-fetching after rotation
    public DataSource dataSource(SecretsManagerClient secretsManager) {
        String secretJson = secretsManager.getSecretValue(
            req -> req.secretId("prod/myapp/rds")
        ).secretString();

        ObjectMapper mapper = new ObjectMapper();
        Map<String, String> secret = mapper.readValue(secretJson, Map.class);

        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://" + secret.get("host") + ":5432/" + secret.get("dbname"));
        config.setUsername(secret.get("username"));
        config.setPassword(secret.get("password"));
        config.setMaximumPoolSize(10);
        return new HikariDataSource(config);
    }
}
```

**Auto-rotation setup for RDS password:**
```json
{
  "RotationRules": {
    "AutomaticallyAfterDays": 30
  },
  "RotationLambdaARN": "arn:aws:lambda:us-east-1:...:function:SecretsManagerRDSRotation"
}
```

AWS provides pre-built rotation Lambda functions for RDS, Redshift, and DocumentDB. When rotation runs, it:
1. Creates a new password
2. Updates the DB user password
3. Updates the secret in Secrets Manager
4. Verifies the new password works
5. Marks rotation complete

Your application picks up the new password on the next `getSecretValue()` call (or you can set up `@RefreshScope` + Spring Cloud Config for hot reload).
