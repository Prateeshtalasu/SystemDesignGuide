# ☁️ Cloud (AWS Focus)

## 0️⃣ Prerequisites

Before diving into cloud computing, you should understand:

- **Networking Basics**: IP addresses, ports, DNS, HTTP (covered in Phase 2)
- **Server Concepts**: What a server does, how applications run on servers
- **Docker**: Containers and images (covered in Topic 4)
- **Databases**: SQL vs NoSQL, basic database operations (covered in Phase 3)

Quick refresher on **client-server model**: A client (browser, mobile app) sends requests to a server. The server processes requests and returns responses. The server runs on a computer somewhere, and cloud computing is about where that "somewhere" is.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Before Cloud

Imagine launching a startup in 2005:

**Problem 1: Capital Expenditure (CapEx) Nightmare**

To launch your web application, you need:
```
- 4 servers: $20,000
- Network equipment: $5,000
- Storage arrays: $15,000
- Rack space in datacenter: $2,000/month
- Bandwidth: $1,000/month
- System administrator salary: $80,000/year
- Total upfront: $40,000+
- Monthly: $3,000+ (before you have any customers)
```

You haven't written a line of code, and you've spent $40,000.

**Problem 2: Capacity Planning Guesswork**

Your app might get 100 users or 100,000 users. You must guess:
- Too little capacity → Site crashes under load, users leave
- Too much capacity → Money wasted on idle servers

Lead time to add capacity: 4-6 weeks (order hardware, ship, rack, configure).

**Problem 3: The 3 AM Pager**

Server hardware fails. Disk crashes. Network switch dies. Power outage.

Someone needs to:
- Monitor 24/7
- Respond to incidents
- Replace failed hardware
- Manage backups
- Apply security patches

This is expensive and distracting from building your product.

**Problem 4: Global Reach is Hard**

Your users are in USA, Europe, and Asia. To serve them well:
- Build datacenters on three continents
- Manage three sets of infrastructure
- Handle data replication across regions
- Comply with local regulations

Only Fortune 500 companies could afford this.

**Problem 5: Scaling is Slow**

Black Friday traffic spike:
```
Normal traffic: 1,000 requests/second
Black Friday: 50,000 requests/second
```

With on-premises:
1. Realize you need more capacity (too late)
2. Order servers (1-2 weeks)
3. Servers arrive, rack them (1 week)
4. Configure and deploy (days)
5. Black Friday is over

### What Breaks Without Cloud

| Scenario | On-Premises | Cloud |
|----------|-------------|-------|
| Startup costs | $50,000+ upfront | $0 upfront, pay as you go |
| Scaling | Weeks | Minutes |
| Global presence | Years and millions | Hours and hundreds |
| Hardware failure | Your problem | Provider's problem |
| Capacity planning | Guess and pray | Scale on demand |
| Innovation speed | Slow (hardware constraints) | Fast (API call away) |

---

## 2️⃣ Intuition and Mental Model

### The Utility Computing Analogy

Think of cloud computing like **electricity**.

**Before electric utilities** (1880s):
- Factories had their own generators
- Hired engineers to maintain them
- Sized generators for peak demand (wasted capacity)
- Generator failure = factory stops

**After electric utilities**:
- Plug into the grid
- Pay for what you use
- Utility handles generation, maintenance, scaling
- You focus on your business

Cloud computing is the same for servers:
- Don't buy servers, rent them
- Pay for what you use
- Provider handles hardware, maintenance, scaling
- You focus on your application

### The AWS Mental Model

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              AWS CLOUD                                   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         REGION (us-east-1)                       │   │
│  │                                                                  │   │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐   │   │
│  │  │ Availability     │  │ Availability     │  │ Availability │   │   │
│  │  │ Zone A           │  │ Zone B           │  │ Zone C       │   │   │
│  │  │ (us-east-1a)     │  │ (us-east-1b)     │  │ (us-east-1c) │   │   │
│  │  │                  │  │                  │  │              │   │   │
│  │  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────┐  │   │   │
│  │  │  │ EC2        │  │  │  │ EC2        │  │  │  │ EC2    │  │   │   │
│  │  │  │ RDS        │  │  │  │ RDS        │  │  │  │ RDS    │  │   │   │
│  │  │  │ EBS        │  │  │  │ EBS        │  │  │  │ EBS    │  │   │   │
│  │  │  └────────────┘  │  │  └────────────┘  │  │  └────────┘  │   │   │
│  │  └──────────────────┘  └──────────────────┘  └──────────────┘   │   │
│  │                                                                  │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │              Regional Services (span all AZs)               │ │   │
│  │  │  S3, DynamoDB, SQS, SNS, Lambda, API Gateway               │ │   │
│  │  └────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                    Global Services                                  │ │
│  │  IAM, Route 53, CloudFront, WAF                                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key concepts**:
- **Region**: Geographic area (us-east-1 = N. Virginia, eu-west-1 = Ireland)
- **Availability Zone (AZ)**: Isolated datacenter within a region
- **Regional Services**: Automatically replicated across AZs
- **Global Services**: Work across all regions

---

## 3️⃣ Compute Services

### EC2 (Elastic Compute Cloud)

**What it is**: Virtual servers in the cloud. You choose CPU, memory, storage, and OS.

**Mental model**: Renting a computer by the hour.

```
┌─────────────────────────────────────────────────┐
│                  EC2 Instance                    │
│                                                  │
│  ┌─────────────┐  ┌─────────────────────────┐   │
│  │    vCPU     │  │       Memory            │   │
│  │   (2-96+)   │  │     (1GB - 768GB)       │   │
│  └─────────────┘  └─────────────────────────┘   │
│                                                  │
│  ┌─────────────────────────────────────────────┐ │
│  │              Operating System                │ │
│  │     (Amazon Linux, Ubuntu, Windows, etc.)    │ │
│  └─────────────────────────────────────────────┘ │
│                                                  │
│  ┌─────────────────────────────────────────────┐ │
│  │              Your Application                │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
         │
         │ Attached Storage
         ▼
┌─────────────────┐
│   EBS Volume    │
│  (Persistent)   │
└─────────────────┘
```

**Instance Types**:

| Family | Use Case | Example |
|--------|----------|---------|
| t3 | General purpose, burstable | Web servers, dev environments |
| m6i | General purpose, balanced | Application servers |
| c6i | Compute optimized | Batch processing, gaming |
| r6i | Memory optimized | In-memory databases, caching |
| i3 | Storage optimized | Data warehousing |
| g4dn | GPU instances | Machine learning, video encoding |

**Naming convention**: `t3.medium`
- `t3`: Instance family
- `medium`: Size (nano, micro, small, medium, large, xlarge, 2xlarge...)

**Pricing Models**:

| Model | Discount | Commitment | Use Case |
|-------|----------|------------|----------|
| On-Demand | 0% | None | Unpredictable workloads |
| Reserved | 30-72% | 1-3 years | Steady-state workloads |
| Spot | Up to 90% | None (can be interrupted) | Fault-tolerant batch jobs |
| Savings Plans | Up to 72% | $/hour commitment | Flexible compute needs |

### Lambda (Serverless)

**What it is**: Run code without managing servers. Pay only for execution time.

**Mental model**: A function that runs when triggered, then disappears.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Trigger   │────▶│   Lambda    │────▶│   Output    │
│             │     │  Function   │     │             │
│ - API call  │     │             │     │ - Response  │
│ - S3 upload │     │ Your code   │     │ - Write DB  │
│ - Schedule  │     │ runs here   │     │ - Send msg  │
│ - SQS msg   │     │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
```

**Lambda Example (Java)**:

```java
// Handler class
package com.example;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;

public class HelloHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent request, Context context) {
        context.getLogger().log("Received request: " + request.getBody());
        
        return new APIGatewayProxyResponseEvent()
            .withStatusCode(200)
            .withBody("{\"message\": \"Hello from Lambda!\"}");
    }
}
```

**Lambda Limits**:
- Max execution time: 15 minutes
- Max memory: 10GB
- Max package size: 250MB (unzipped)
- Concurrent executions: 1000 (default, can increase)

**When to use Lambda**:
- Event-driven processing (S3 uploads, API requests)
- Scheduled tasks (cron jobs)
- Short-lived operations
- Variable traffic with idle periods

**When NOT to use Lambda**:
- Long-running processes (>15 min)
- High-throughput, steady traffic (EC2 may be cheaper)
- Applications requiring persistent connections
- Workloads needing GPUs

### ECS (Elastic Container Service)

**What it is**: Run Docker containers on AWS-managed infrastructure.

```
┌─────────────────────────────────────────────────────────────────┐
│                         ECS Cluster                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                        Service                               │ │
│  │  (Maintains desired count of tasks)                         │ │
│  │                                                              │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │ │
│  │  │    Task      │  │    Task      │  │    Task      │      │ │
│  │  │              │  │              │  │              │      │ │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │      │ │
│  │  │ │Container │ │  │ │Container │ │  │ │Container │ │      │ │
│  │  │ │ (app)    │ │  │ │ (app)    │ │  │ │ (app)    │ │      │ │
│  │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │      │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Launch Type:                                                    │
│  - EC2: You manage the instances                                │
│  - Fargate: AWS manages infrastructure (serverless containers)  │
└─────────────────────────────────────────────────────────────────┘
```

### EKS (Elastic Kubernetes Service)

**What it is**: Managed Kubernetes on AWS. AWS manages the control plane, you manage worker nodes.

Use EKS when:
- You need Kubernetes features
- You want portability across clouds
- You have Kubernetes expertise

Use ECS when:
- Simpler container orchestration is enough
- You're AWS-native
- You want less operational overhead

### Fargate

**What it is**: Serverless compute for containers. No EC2 instances to manage.

```
ECS/EKS with EC2:
┌─────────────────┐
│  You manage:    │
│  - EC2 instances│
│  - Scaling      │
│  - Patching     │
└─────────────────┘

ECS/EKS with Fargate:
┌─────────────────┐
│  AWS manages:   │
│  - Infrastructure│
│  - Scaling      │
│  - Patching     │
│                 │
│  You specify:   │
│  - CPU/Memory   │
│  - Container    │
└─────────────────┘
```

---

## 4️⃣ Storage Services

### S3 (Simple Storage Service)

**What it is**: Object storage for any amount of data. Not a file system, but a key-value store for files.

**Mental model**: Infinite hard drive with a flat structure.

```
┌─────────────────────────────────────────────────────────────────┐
│                         S3 Bucket                                │
│                    (my-company-data)                            │
│                                                                  │
│  Key                              │  Value (Object)              │
│  ─────────────────────────────────│──────────────────────────── │
│  images/logo.png                  │  [binary data]              │
│  images/banner.jpg                │  [binary data]              │
│  documents/report-2024.pdf        │  [binary data]              │
│  backups/db-2024-01-15.sql.gz     │  [binary data]              │
│                                                                  │
│  Note: "folders" are just key prefixes, not real directories    │
└─────────────────────────────────────────────────────────────────┘
```

**S3 Storage Classes**:

| Class | Use Case | Availability | Cost |
|-------|----------|--------------|------|
| Standard | Frequently accessed | 99.99% | $$$ |
| Intelligent-Tiering | Unknown access patterns | 99.9% | $$ (+ monitoring fee) |
| Standard-IA | Infrequent access | 99.9% | $$ |
| One Zone-IA | Infrequent, non-critical | 99.5% | $ |
| Glacier Instant | Archive, instant retrieval | 99.9% | $ |
| Glacier Flexible | Archive, minutes to hours | 99.99% | ¢ |
| Glacier Deep Archive | Archive, 12+ hours | 99.99% | ¢¢ |

**S3 Features**:
- **Versioning**: Keep multiple versions of objects
- **Lifecycle policies**: Auto-transition to cheaper storage classes
- **Replication**: Cross-region or same-region replication
- **Encryption**: Server-side (SSE-S3, SSE-KMS) or client-side
- **Static website hosting**: Serve HTML/CSS/JS directly

### EBS (Elastic Block Store)

**What it is**: Block storage volumes for EC2 instances. Like a virtual hard drive.

```
┌─────────────────┐         ┌─────────────────┐
│  EC2 Instance   │◀───────▶│   EBS Volume    │
│                 │         │                 │
│  /dev/xvda      │         │  100 GB SSD     │
│  (root volume)  │         │  (gp3)          │
└─────────────────┘         └─────────────────┘
         │
         │ Can attach
         │ additional volumes
         ▼
┌─────────────────┐
│   EBS Volume    │
│                 │
│  500 GB HDD     │
│  (st1)          │
└─────────────────┘
```

**EBS Volume Types**:

| Type | Use Case | IOPS | Throughput |
|------|----------|------|------------|
| gp3 | General purpose SSD | 3,000-16,000 | 125-1,000 MB/s |
| gp2 | General purpose SSD (legacy) | 3,000-16,000 | Tied to size |
| io2 | High-performance SSD | Up to 256,000 | 4,000 MB/s |
| st1 | Throughput HDD | 500 | 500 MB/s |
| sc1 | Cold HDD | 250 | 250 MB/s |

**Key differences from S3**:
- EBS: Block storage, attached to single EC2, supports file systems
- S3: Object storage, accessed via HTTP, unlimited size

### EFS (Elastic File System)

**What it is**: Managed NFS file system. Multiple EC2 instances can mount the same file system.

```
┌─────────────────────────────────────────────────────────────────┐
│                          EFS File System                         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    /shared-data                           │   │
│  │                                                           │   │
│  │    /logs         /uploads        /config                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
           │                    │                    │
           │ NFS mount          │ NFS mount          │ NFS mount
           ▼                    ▼                    ▼
    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │ EC2 Instance │    │ EC2 Instance │    │ EC2 Instance │
    │   (AZ-a)     │    │   (AZ-b)     │    │   (AZ-c)     │
    └──────────────┘    └──────────────┘    └──────────────┘
```

Use EFS when:
- Multiple instances need shared file access
- You need a POSIX file system
- Content management, web serving, data sharing

---

## 5️⃣ Database Services

### RDS (Relational Database Service)

**What it is**: Managed relational databases. AWS handles backups, patching, replication.

**Supported engines**: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Aurora

```
┌─────────────────────────────────────────────────────────────────┐
│                        RDS Instance                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  AWS Manages:                                                │ │
│  │  - Hardware provisioning                                     │ │
│  │  - Database setup                                            │ │
│  │  - Patching                                                  │ │
│  │  - Backups (automated, point-in-time recovery)              │ │
│  │  - Multi-AZ failover                                        │ │
│  │  - Read replicas                                            │ │
│  │  - Monitoring                                                │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  You Manage:                                                 │ │
│  │  - Schema design                                             │ │
│  │  - Query optimization                                        │ │
│  │  - Application connections                                   │ │
│  │  - Security groups                                           │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Multi-AZ Deployment**:

```
┌─────────────────────┐         ┌─────────────────────┐
│    Primary (AZ-a)   │◀───────▶│   Standby (AZ-b)    │
│                     │  Sync   │                     │
│  ┌───────────────┐  │  Repl   │  ┌───────────────┐  │
│  │  PostgreSQL   │  │         │  │  PostgreSQL   │  │
│  │  (Active)     │  │         │  │  (Standby)    │  │
│  └───────────────┘  │         │  └───────────────┘  │
└─────────────────────┘         └─────────────────────┘
         ▲
         │ All traffic
         │
    Application
```

Failover is automatic. DNS endpoint stays the same.

### Aurora

**What it is**: AWS-built database compatible with MySQL and PostgreSQL. 5x faster than standard MySQL.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Aurora Cluster                            │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Shared Storage Layer                      │ │
│  │              (6 copies across 3 AZs)                        │ │
│  │                                                              │ │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐│ │
│  │  │Copy 1│  │Copy 2│  │Copy 3│  │Copy 4│  │Copy 5│  │Copy 6││ │
│  │  │ AZ-a │  │ AZ-a │  │ AZ-b │  │ AZ-b │  │ AZ-c │  │ AZ-c ││ │
│  │  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘│ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              ▲                                   │
│                              │                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Primary    │    │ Read Replica │    │ Read Replica │       │
│  │   (Writer)   │    │   (Reader)   │    │   (Reader)   │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

**Aurora Serverless**: Auto-scaling Aurora. Pay per second, scales to zero.

### DynamoDB

**What it is**: Managed NoSQL database. Key-value and document store.

**Mental model**: Infinitely scalable hash table.

```
┌─────────────────────────────────────────────────────────────────┐
│                       DynamoDB Table                             │
│                        (Users)                                   │
│                                                                  │
│  Partition Key │  Sort Key  │  Attributes                       │
│  (user_id)     │  (none)    │                                   │
│  ──────────────│────────────│─────────────────────────────────  │
│  user-001      │            │  {name: "Alice", email: "..."}    │
│  user-002      │            │  {name: "Bob", email: "..."}      │
│  user-003      │            │  {name: "Charlie", email: "..."}  │
│                                                                  │
│  Capacity:                                                       │
│  - On-Demand: Pay per request                                   │
│  - Provisioned: Set read/write capacity units                   │
└─────────────────────────────────────────────────────────────────┘
```

**DynamoDB Features**:
- Single-digit millisecond latency at any scale
- Auto-scaling
- Global tables (multi-region replication)
- Streams (change data capture)
- TTL (auto-delete expired items)

**When to use DynamoDB**:
- Key-value lookups
- Session storage
- Gaming leaderboards
- IoT data
- High-throughput, low-latency needs

**When NOT to use DynamoDB**:
- Complex queries with joins
- Ad-hoc analytics
- Small datasets (RDS may be cheaper)

### ElastiCache

**What it is**: Managed Redis or Memcached.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ElastiCache Cluster                           │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Redis Cluster Mode                     │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │   │
│  │  │  Shard 1    │  │  Shard 2    │  │  Shard 3    │       │   │
│  │  │ Primary     │  │ Primary     │  │ Primary     │       │   │
│  │  │ + Replica   │  │ + Replica   │  │ + Replica   │       │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

Use cases:
- Session storage
- Database query caching
- Real-time analytics
- Leaderboards
- Pub/sub messaging

---

## 6️⃣ Messaging Services

### SQS (Simple Queue Service)

**What it is**: Managed message queue. Decouple producers from consumers.

```
┌──────────────┐     ┌─────────────────────┐     ┌──────────────┐
│   Producer   │────▶│    SQS Queue        │────▶│   Consumer   │
│              │     │                     │     │              │
│  Order       │     │  ┌───┐ ┌───┐ ┌───┐ │     │  Order       │
│  Service     │     │  │msg│ │msg│ │msg│ │     │  Processor   │
│              │     │  └───┘ └───┘ └───┘ │     │              │
└──────────────┘     └─────────────────────┘     └──────────────┘
```

**Queue Types**:

| Type | Ordering | Deduplication | Throughput |
|------|----------|---------------|------------|
| Standard | Best-effort | At-least-once | Unlimited |
| FIFO | Strict FIFO | Exactly-once | 3,000 msg/sec (with batching) |

**SQS Java Example**:

```java
// Send message
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.*;

SqsClient sqs = SqsClient.create();

SendMessageRequest sendRequest = SendMessageRequest.builder()
    .queueUrl("https://sqs.us-east-1.amazonaws.com/123456789/my-queue")
    .messageBody("{\"orderId\": \"12345\", \"amount\": 99.99}")
    .delaySeconds(0)
    .build();

sqs.sendMessage(sendRequest);

// Receive messages
ReceiveMessageRequest receiveRequest = ReceiveMessageRequest.builder()
    .queueUrl("https://sqs.us-east-1.amazonaws.com/123456789/my-queue")
    .maxNumberOfMessages(10)
    .waitTimeSeconds(20)  // Long polling
    .build();

List<Message> messages = sqs.receiveMessage(receiveRequest).messages();

for (Message message : messages) {
    // Process message
    processOrder(message.body());
    
    // Delete after successful processing
    sqs.deleteMessage(DeleteMessageRequest.builder()
        .queueUrl(queueUrl)
        .receiptHandle(message.receiptHandle())
        .build());
}
```

### SNS (Simple Notification Service)

**What it is**: Pub/sub messaging. One message, many subscribers.

```
                              ┌──────────────────┐
                         ┌───▶│  SQS Queue       │
                         │    │  (order-proc)    │
┌──────────────┐         │    └──────────────────┘
│   Publisher  │         │
│              │    ┌────┴────┐
│  Order       │───▶│   SNS   │
│  Service     │    │  Topic  │
│              │    └────┬────┘
└──────────────┘         │    ┌──────────────────┐
                         ├───▶│  Lambda          │
                         │    │  (send-email)    │
                         │    └──────────────────┘
                         │
                         │    ┌──────────────────┐
                         └───▶│  HTTP Endpoint   │
                              │  (analytics)     │
                              └──────────────────┘
```

**SNS Subscribers**:
- SQS queues
- Lambda functions
- HTTP/HTTPS endpoints
- Email
- SMS
- Mobile push notifications

### EventBridge

**What it is**: Serverless event bus. Route events based on rules.

```
┌─────────────────────────────────────────────────────────────────┐
│                        EventBridge                               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                         Event Bus                            │ │
│  │                                                              │ │
│  │  Rule 1: source = "order-service"                           │ │
│  │          detail-type = "OrderCreated"                       │ │
│  │          → Target: Lambda (process-order)                   │ │
│  │                                                              │ │
│  │  Rule 2: source = "order-service"                           │ │
│  │          detail-type = "OrderCreated"                       │ │
│  │          detail.amount > 1000                               │ │
│  │          → Target: SNS (high-value-orders)                  │ │
│  │                                                              │ │
│  │  Rule 3: schedule = "rate(1 hour)"                          │ │
│  │          → Target: Lambda (cleanup-job)                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Kinesis

**What it is**: Real-time streaming data. For high-throughput, ordered event streams.

```
┌──────────────┐     ┌─────────────────────────────────────────┐
│  Producers   │     │           Kinesis Data Stream           │
│              │     │                                         │
│  IoT devices │     │  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  Clickstream │────▶│  │ Shard 1 │  │ Shard 2 │  │ Shard 3 │ │
│  Logs        │     │  └─────────┘  └─────────┘  └─────────┘ │
│              │     │                                         │
└──────────────┘     └─────────────────────────────────────────┘
                                        │
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
              ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
              │   Lambda     │  │   Kinesis    │  │   Kinesis    │
              │   Consumer   │  │   Analytics  │  │   Firehose   │
              └──────────────┘  └──────────────┘  └──────────────┘
                                                         │
                                                         ▼
                                                  ┌──────────────┐
                                                  │      S3      │
                                                  │  (archive)   │
                                                  └──────────────┘
```

**SQS vs SNS vs Kinesis**:

| Feature | SQS | SNS | Kinesis |
|---------|-----|-----|---------|
| Pattern | Queue | Pub/Sub | Stream |
| Ordering | FIFO available | No | Per shard |
| Retention | 14 days | None (instant) | 1-365 days |
| Replay | No | No | Yes |
| Throughput | Unlimited | Unlimited | Per shard |
| Use case | Task queues | Fan-out | Real-time analytics |

---

## 7️⃣ Networking Services

### VPC (Virtual Private Cloud)

**What it is**: Your private network in AWS. You control IP ranges, subnets, routing, and security.

```
┌─────────────────────────────────────────────────────────────────┐
│                     VPC (10.0.0.0/16)                            │
│                                                                  │
│  ┌────────────────────────────┐  ┌────────────────────────────┐ │
│  │    Public Subnet           │  │    Public Subnet           │ │
│  │    (10.0.1.0/24)           │  │    (10.0.2.0/24)           │ │
│  │    AZ-a                    │  │    AZ-b                    │ │
│  │                            │  │                            │ │
│  │  ┌──────────────────────┐  │  │  ┌──────────────────────┐  │ │
│  │  │   NAT Gateway        │  │  │  │   Load Balancer      │  │ │
│  │  └──────────────────────┘  │  │  └──────────────────────┘  │ │
│  └────────────────────────────┘  └────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────┐  ┌────────────────────────────┐ │
│  │    Private Subnet          │  │    Private Subnet          │ │
│  │    (10.0.3.0/24)           │  │    (10.0.4.0/24)           │ │
│  │    AZ-a                    │  │    AZ-b                    │ │
│  │                            │  │                            │ │
│  │  ┌──────────────────────┐  │  │  ┌──────────────────────┐  │ │
│  │  │   EC2 (App Server)   │  │  │  │   EC2 (App Server)   │  │ │
│  │  └──────────────────────┘  │  │  └──────────────────────┘  │ │
│  │  ┌──────────────────────┐  │  │  ┌──────────────────────┐  │ │
│  │  │   RDS (Primary)      │  │  │  │   RDS (Standby)      │  │ │
│  │  └──────────────────────┘  │  │  └──────────────────────┘  │ │
│  └────────────────────────────┘  └────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Internet Gateway                         │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                          Internet
```

**Key Components**:
- **Subnet**: Range of IP addresses. Public (has route to internet) or private.
- **Internet Gateway**: Allows public subnet resources to reach internet.
- **NAT Gateway**: Allows private subnet resources to reach internet (outbound only).
- **Route Table**: Rules for where network traffic is directed.
- **Security Group**: Stateful firewall for instances.
- **Network ACL**: Stateless firewall for subnets.

### ELB (Elastic Load Balancer)

**Types**:

| Type | Layer | Use Case |
|------|-------|----------|
| Application LB (ALB) | Layer 7 (HTTP) | Web apps, path-based routing |
| Network LB (NLB) | Layer 4 (TCP/UDP) | High performance, static IP |
| Gateway LB (GWLB) | Layer 3 | Third-party appliances |
| Classic LB | Layer 4/7 | Legacy (don't use for new) |

```
                         ┌─────────────────┐
                         │  Application    │
                         │  Load Balancer  │
                         └────────┬────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
       ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
       │ Target Group │   │ Target Group │   │ Target Group │
       │  /api/*      │   │  /web/*      │   │  /admin/*    │
       └──────────────┘   └──────────────┘   └──────────────┘
              │                   │                   │
              ▼                   ▼                   ▼
       ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
       │  EC2/ECS     │   │  EC2/ECS     │   │  EC2/ECS     │
       │  (API)       │   │  (Web)       │   │  (Admin)     │
       └──────────────┘   └──────────────┘   └──────────────┘
```

### Route 53

**What it is**: Managed DNS service.

**Routing Policies**:

| Policy | Description |
|--------|-------------|
| Simple | Single resource |
| Weighted | Distribute traffic by weight (A/B testing) |
| Latency | Route to lowest latency region |
| Failover | Active-passive failover |
| Geolocation | Route based on user location |
| Geoproximity | Route based on resource location |
| Multivalue | Return multiple healthy records |

### CloudFront

**What it is**: Content Delivery Network (CDN). Cache content at edge locations worldwide.

```
┌─────────────────────────────────────────────────────────────────┐
│                         CloudFront                               │
│                                                                  │
│  User (Tokyo)                                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────┐                                               │
│  │ Edge Location│ ◀─── Cache Hit: Return immediately           │
│  │ (Tokyo)      │                                               │
│  └──────────────┘                                               │
│       │                                                         │
│       │ Cache Miss                                              │
│       ▼                                                         │
│  ┌──────────────┐                                               │
│  │ Regional     │                                               │
│  │ Edge Cache   │                                               │
│  └──────────────┘                                               │
│       │                                                         │
│       │ Cache Miss                                              │
│       ▼                                                         │
│  ┌──────────────┐                                               │
│  │   Origin     │ (S3, ALB, EC2, or any HTTP server)           │
│  │ (us-east-1)  │                                               │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
```

### API Gateway

**What it is**: Managed API front door. Handle authentication, throttling, caching.

```
┌─────────────────────────────────────────────────────────────────┐
│                       API Gateway                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Features:                                                   │ │
│  │  - Authentication (IAM, Cognito, Lambda authorizer)         │ │
│  │  - Rate limiting & throttling                               │ │
│  │  - Request/response transformation                          │ │
│  │  - Caching                                                  │ │
│  │  - API versioning                                           │ │
│  │  - OpenAPI/Swagger support                                  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│       ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│       │   Lambda     │ │   EC2/ECS    │ │   HTTP       │       │
│       │              │ │              │ │   Backend    │       │
│       └──────────────┘ └──────────────┘ └──────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8️⃣ IAM (Identity and Access Management)

### Core Concepts

```
┌─────────────────────────────────────────────────────────────────┐
│                           IAM                                    │
│                                                                  │
│  ┌─────────────────┐                                            │
│  │      User       │  Human identity                            │
│  │   (alice)       │  Has username/password or access keys      │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐                                            │
│  │     Group       │  Collection of users                       │
│  │  (developers)   │  Policies attached to group apply to all   │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐                                            │
│  │      Role       │  Identity for AWS services or federation  │
│  │ (ec2-app-role)  │  No long-term credentials                  │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐                                            │
│  │     Policy      │  JSON document defining permissions        │
│  │                 │  Attached to users, groups, or roles       │
│  └─────────────────┘                                            │
└─────────────────────────────────────────────────────────────────┘
```

### Policy Structure

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3ReadAccess",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ],
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "192.168.1.0/24"
                }
            }
        },
        {
            "Sid": "DenyDeleteBucket",
            "Effect": "Deny",
            "Action": "s3:DeleteBucket",
            "Resource": "*"
        }
    ]
}
```

### Least Privilege Principle

**Bad**: Give admin access to everything
```json
{
    "Effect": "Allow",
    "Action": "*",
    "Resource": "*"
}
```

**Good**: Give only what's needed
```json
{
    "Effect": "Allow",
    "Action": [
        "s3:GetObject",
        "s3:PutObject"
    ],
    "Resource": "arn:aws:s3:::my-app-uploads/*"
}
```

### IAM Roles for EC2

```
┌─────────────────────────────────────────────────────────────────┐
│                      EC2 Instance                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Instance Profile                                            │ │
│  │  (contains IAM Role)                                        │ │
│  │                                                              │ │
│  │  Role: app-server-role                                      │ │
│  │  Policies:                                                  │ │
│  │    - AmazonS3ReadOnlyAccess                                 │ │
│  │    - AmazonSQSFullAccess                                    │ │
│  │    - CloudWatchLogsFullAccess                               │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Your application automatically gets temporary credentials       │
│  via instance metadata service. No hardcoded keys!              │
└─────────────────────────────────────────────────────────────────┘
```

```java
// Java SDK automatically uses instance role credentials
S3Client s3 = S3Client.create();  // No credentials needed!
s3.getObject(GetObjectRequest.builder()
    .bucket("my-bucket")
    .key("my-file.txt")
    .build());
```

---

## 9️⃣ Cost Optimization

### Cost Pillars

1. **Right-sizing**: Use appropriate instance sizes
2. **Reserved capacity**: Commit for discounts
3. **Spot instances**: Use for fault-tolerant workloads
4. **Storage tiering**: Move cold data to cheaper storage
5. **Turn off unused resources**: Stop dev environments at night

### Cost Monitoring Tools

- **AWS Cost Explorer**: Visualize spending over time
- **AWS Budgets**: Set alerts when spending exceeds threshold
- **Cost Allocation Tags**: Track costs by project/team
- **Trusted Advisor**: Recommendations for cost savings

### Common Cost Mistakes

| Mistake | Solution |
|---------|----------|
| Oversized instances | Use CloudWatch to right-size |
| Unused EBS volumes | Delete unattached volumes |
| Old snapshots | Lifecycle policies to delete |
| Data transfer costs | Use VPC endpoints, same-AZ |
| Always-on dev environments | Auto-shutdown schedules |
| Not using Reserved Instances | Analyze steady-state workloads |

---

## 🔟 Interview Follow-Up Questions

### Q1: "Explain the difference between EC2, Lambda, and ECS."

**Answer**:
These are three compute options with different levels of abstraction:

**EC2**: Virtual machines. You manage everything: OS, patching, scaling, deployment. Most control, most responsibility. Use for long-running applications, specific OS needs, or when you need full control.

**Lambda**: Serverless functions. No servers to manage. Code runs in response to events, scales automatically, pay per execution. Use for event-driven workloads, short tasks (<15 min), variable traffic.

**ECS/Fargate**: Container orchestration. You package apps in containers, AWS manages running them. Middle ground between EC2 and Lambda. Use for containerized microservices, when you need more control than Lambda but less than EC2.

### Q2: "When would you use SQS vs SNS vs Kinesis?"

**Answer**:
**SQS**: Message queue for decoupling services. One producer, one consumer (per message). Use for task queues, job processing, handling traffic spikes.

**SNS**: Pub/sub for fan-out. One message goes to many subscribers. Use for notifications, event broadcasting, triggering multiple downstream processes.

**Kinesis**: Real-time streaming for ordered, high-volume data. Retains data for replay. Use for log aggregation, real-time analytics, IoT data ingestion.

Example: Order placed → SNS notifies multiple services → One subscriber puts message in SQS for processing → Another streams to Kinesis for analytics.

### Q3: "How would you design a highly available architecture on AWS?"

**Answer**:
Key principles:

1. **Multi-AZ**: Deploy across at least 2 Availability Zones
2. **Load balancing**: ALB/NLB distributes traffic
3. **Auto-scaling**: EC2 Auto Scaling or Fargate scales capacity
4. **Managed databases**: RDS Multi-AZ or Aurora for automatic failover
5. **Stateless applications**: Store state in ElastiCache or DynamoDB
6. **Health checks**: ALB health checks remove unhealthy instances
7. **DNS failover**: Route 53 health checks for regional failover

Architecture:
- Route 53 → CloudFront → ALB (Multi-AZ) → ECS/EC2 (Multi-AZ) → RDS Multi-AZ
- ElastiCache for sessions, S3 for static assets

### Q4: "What is the principle of least privilege and how do you implement it in AWS?"

**Answer**:
Least privilege means giving only the minimum permissions needed to perform a task. In AWS:

1. **Use IAM roles, not users**: Roles have temporary credentials
2. **Specific actions**: `s3:GetObject` not `s3:*`
3. **Specific resources**: `arn:aws:s3:::my-bucket/*` not `*`
4. **Conditions**: Restrict by IP, time, MFA
5. **Service-specific roles**: Each service gets its own role
6. **Regular audits**: Use IAM Access Analyzer to find unused permissions
7. **Policy boundaries**: Set maximum permissions for a role

Example: An application that reads from S3 and writes to SQS gets a role with only those specific permissions, not admin access.

### Q5: "How do you optimize costs on AWS?"

**Answer**:
I approach cost optimization in layers:

1. **Visibility first**: Enable Cost Explorer, set up budgets and alerts
2. **Right-sizing**: Use CloudWatch metrics to identify over-provisioned instances
3. **Reserved capacity**: Analyze steady-state workloads for 1-3 year commitments
4. **Spot instances**: Use for fault-tolerant batch processing
5. **Storage tiering**: S3 lifecycle policies, delete unused EBS/snapshots
6. **Architecture**: Use serverless where appropriate, VPC endpoints to reduce data transfer
7. **Scheduling**: Stop dev/test environments outside business hours
8. **Tagging**: Tag resources for cost allocation by team/project

I also review the AWS Trusted Advisor recommendations regularly.

---

## 1️⃣1️⃣ One Clean Mental Summary

AWS provides building blocks for running applications in the cloud. Compute options range from full control (EC2) to serverless (Lambda). Storage includes object storage (S3), block storage (EBS), and file systems (EFS). Databases are managed (RDS, DynamoDB) so you focus on data, not operations.

Networking (VPC) gives you an isolated network with public and private subnets. Load balancers (ALB/NLB) distribute traffic. Route 53 handles DNS. CloudFront caches content globally.

Messaging services (SQS, SNS, Kinesis) decouple components. IAM controls who can do what. Design for high availability with Multi-AZ deployments, and optimize costs with right-sizing, reserved instances, and monitoring.

The cloud's value proposition: trade capital expense for operational expense, scale on demand, and focus on your application instead of infrastructure.

