# 📈 Capacity Planning

## 0️⃣ Prerequisites

Before diving into capacity planning, you should understand:

- **Back-of-Envelope Calculations**: Estimation techniques (Phase 1, Topic 11)
- **Metrics and Monitoring**: Understanding system metrics (Topic 10)
- **Load Testing**: How to measure system capacity (Topic 15)
- **Cloud Infrastructure**: Scaling options (Topic 5)

Quick refresher on **throughput**: Throughput is the rate at which a system processes requests, typically measured in requests per second (RPS) or transactions per second (TPS).

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Without Capacity Planning

**Problem 1: The "Black Friday Crash"**

```
Normal traffic: 10,000 requests/second
Black Friday: 100,000 requests/second

What happened:
- No one predicted the traffic
- Servers overloaded
- Database connections exhausted
- Site down for 4 hours
- $2 million revenue lost
```

**Problem 2: The "Over-Provisioned Waste"**

```
Provisioned capacity: 100 servers
Actual usage: 10 servers worth

Monthly cost: $50,000
Actual need: $5,000

$45,000/month wasted because "we might need it"
```

**Problem 3: The "Slow Creep to Failure"**

```
January: 50% CPU utilization
March: 60% CPU utilization
June: 75% CPU utilization
September: 90% CPU utilization
October: 100% CPU utilization → Outage

No one noticed the trend.
No one planned for growth.
```

**Problem 4: The "Scaling Panic"**

```
Monday: Traffic spike detected
Monday: "We need more servers!"
Monday: Procurement process started
Tuesday: Approval pending
Wednesday: Servers ordered
Friday: Servers delivered
Next Monday: Servers configured

Traffic spike was Monday.
Capacity available next Monday.
Week of degraded service.
```

**Problem 5: The "Unknown Limits"**

```
Question: "How many users can we support?"
Answer: "We don't know"

Question: "When do we need to scale?"
Answer: "When it breaks"

Question: "What's our cost per user?"
Answer: "No idea"
```

### What Breaks Without Capacity Planning

| Scenario | Without Planning | With Planning |
|----------|-----------------|---------------|
| Traffic spikes | Outages | Prepared |
| Cost management | Waste or shortage | Optimized |
| Growth | Reactive | Proactive |
| Scaling decisions | Guesswork | Data-driven |
| Budget forecasting | Impossible | Accurate |

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Analogy

Think of capacity planning like **running a restaurant**.

**Without capacity planning**:
- Don't know how many customers to expect
- Sometimes too few staff (long waits)
- Sometimes too many staff (wasted wages)
- Run out of ingredients unexpectedly
- Can't plan for holidays

**With capacity planning**:
- Historical data predicts customer volume
- Staff scheduled based on expected demand
- Inventory ordered based on forecasts
- Extra capacity for holidays
- Know when to expand

### Capacity Planning Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAPACITY PLANNING CYCLE                       │
│                                                                  │
│  1. MEASURE                                                     │
│     - Current utilization                                       │
│     - Traffic patterns                                          │
│     - Resource consumption                                      │
│                                                                  │
│  2. FORECAST                                                    │
│     - Growth rate                                               │
│     - Seasonal patterns                                         │
│     - Business events (launches, sales)                        │
│                                                                  │
│  3. PLAN                                                        │
│     - When will we hit limits?                                 │
│     - What resources are needed?                               │
│     - What's the cost?                                         │
│                                                                  │
│  4. PROVISION                                                   │
│     - Add capacity before needed                               │
│     - Auto-scaling policies                                    │
│     - Reserved instances                                       │
│                                                                  │
│  5. REVIEW                                                      │
│     - Compare forecast vs actual                               │
│     - Adjust models                                            │
│     - Repeat cycle                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Key Metrics for Capacity Planning

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAPACITY METRICS                              │
│                                                                  │
│  UTILIZATION                                                    │
│  - CPU usage (%)                                                │
│  - Memory usage (%)                                             │
│  - Disk usage (%)                                               │
│  - Network bandwidth (%)                                        │
│                                                                  │
│  SATURATION                                                     │
│  - Request queue length                                         │
│  - Connection pool usage                                        │
│  - Thread pool usage                                            │
│                                                                  │
│  THROUGHPUT                                                     │
│  - Requests per second                                          │
│  - Transactions per second                                      │
│  - Data processed per second                                    │
│                                                                  │
│  HEADROOM                                                       │
│  - Available capacity = Max capacity - Current usage           │
│  - Buffer for spikes                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Capacity Planning Process

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAPACITY PLANNING PROCESS                     │
│                                                                  │
│  Step 1: Establish Baseline                                     │
│  ────────────────────────────────────────────────────────────  │
│  - Measure current capacity                                     │
│  - Identify resource limits                                     │
│  - Document current utilization                                 │
│                                                                  │
│  Step 2: Understand Demand                                      │
│  ────────────────────────────────────────────────────────────  │
│  - Analyze traffic patterns                                     │
│  - Identify peak times                                          │
│  - Understand seasonal variations                               │
│                                                                  │
│  Step 3: Forecast Growth                                        │
│  ────────────────────────────────────────────────────────────  │
│  - Historical growth rate                                       │
│  - Business projections                                         │
│  - Planned events (launches, marketing)                        │
│                                                                  │
│  Step 4: Calculate Requirements                                 │
│  ────────────────────────────────────────────────────────────  │
│  - When will we hit 70% utilization?                           │
│  - What resources needed for 2x growth?                        │
│  - What's the cost?                                            │
│                                                                  │
│  Step 5: Plan Scaling                                           │
│  ────────────────────────────────────────────────────────────  │
│  - Auto-scaling policies                                        │
│  - Reserved capacity                                            │
│  - Timeline for scaling                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Utilization Thresholds

```
┌─────────────────────────────────────────────────────────────────┐
│                    UTILIZATION ZONES                             │
│                                                                  │
│  0-50%: GREEN (Healthy)                                         │
│  - Plenty of headroom                                           │
│  - Can handle spikes                                            │
│  - Possibly over-provisioned                                    │
│                                                                  │
│  50-70%: YELLOW (Watch)                                         │
│  - Normal operating range                                       │
│  - Plan for growth                                              │
│  - Monitor trends                                               │
│                                                                  │
│  70-85%: ORANGE (Plan)                                          │
│  - Start scaling planning                                       │
│  - Limited spike capacity                                       │
│  - Provision more resources                                     │
│                                                                  │
│  85-100%: RED (Critical)                                        │
│  - Scale immediately                                            │
│  - At risk of overload                                          │
│  - Performance degradation likely                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation: Capacity Planning in Practice

### Step 1: Back-of-Envelope Calculation

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAPACITY ESTIMATION                           │
│                                                                  │
│  Scenario: E-commerce platform                                  │
│                                                                  │
│  Given:                                                         │
│  - 10 million daily active users                               │
│  - Average 20 page views per user                              │
│  - Peak traffic: 3x average                                    │
│  - Each page view = 5 API calls                                │
│                                                                  │
│  Calculations:                                                  │
│                                                                  │
│  Daily page views:                                              │
│  10M users × 20 pages = 200M page views/day                    │
│                                                                  │
│  Daily API calls:                                               │
│  200M × 5 = 1B API calls/day                                   │
│                                                                  │
│  Average RPS:                                                   │
│  1B / 86,400 seconds ≈ 11,574 RPS                              │
│                                                                  │
│  Peak RPS:                                                      │
│  11,574 × 3 = 34,722 RPS                                       │
│                                                                  │
│  If each server handles 1,000 RPS:                             │
│  Peak servers needed: 35 servers                               │
│  With 30% headroom: 35 × 1.3 = 46 servers                      │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Growth Forecasting

```java
// Capacity forecasting service
@Service
public class CapacityForecastService {
    
    @Autowired
    private MetricsRepository metricsRepository;
    
    public CapacityForecast forecast(String resourceType, int monthsAhead) {
        // Get historical data
        List<MetricDataPoint> history = metricsRepository
            .getMonthlyMetrics(resourceType, 12);  // Last 12 months
        
        // Calculate growth rate
        double growthRate = calculateGrowthRate(history);
        
        // Current utilization
        double currentUtilization = history.get(history.size() - 1).getValue();
        
        // Project future utilization
        List<ForecastPoint> projections = new ArrayList<>();
        double utilization = currentUtilization;
        
        for (int month = 1; month <= monthsAhead; month++) {
            utilization = utilization * (1 + growthRate);
            projections.add(new ForecastPoint(month, utilization));
        }
        
        // Find when we hit 80% threshold
        int monthsToThreshold = projections.stream()
            .filter(p -> p.getUtilization() >= 80)
            .findFirst()
            .map(ForecastPoint::getMonth)
            .orElse(-1);
        
        return CapacityForecast.builder()
            .resourceType(resourceType)
            .currentUtilization(currentUtilization)
            .growthRate(growthRate)
            .projections(projections)
            .monthsToThreshold(monthsToThreshold)
            .recommendation(generateRecommendation(monthsToThreshold))
            .build();
    }
    
    private double calculateGrowthRate(List<MetricDataPoint> history) {
        // Simple linear regression for growth rate
        // In production, use more sophisticated models
        double firstValue = history.get(0).getValue();
        double lastValue = history.get(history.size() - 1).getValue();
        int months = history.size();
        
        return Math.pow(lastValue / firstValue, 1.0 / months) - 1;
    }
    
    private String generateRecommendation(int monthsToThreshold) {
        if (monthsToThreshold < 0) {
            return "No scaling needed in forecast period";
        } else if (monthsToThreshold <= 1) {
            return "CRITICAL: Scale immediately";
        } else if (monthsToThreshold <= 3) {
            return "WARNING: Begin scaling planning";
        } else {
            return "Monitor: Scale in " + monthsToThreshold + " months";
        }
    }
}
```

### Step 3: Auto-Scaling Configuration

```yaml
# Kubernetes Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

```hcl
# AWS Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  vpc_zone_identifier = var.subnet_ids
  target_group_arns   = [aws_lb_target_group.app.arn]
  
  min_size         = 3
  max_size         = 50
  desired_capacity = 5
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  tag {
    key                 = "Name"
    value               = "app-server"
    propagate_at_launch = true
  }
}

# Target tracking scaling policy
resource "aws_autoscaling_policy" "cpu" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"
  
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 70.0
  }
}

# Scheduled scaling for known events
resource "aws_autoscaling_schedule" "scale_up_black_friday" {
  scheduled_action_name  = "scale-up-black-friday"
  autoscaling_group_name = aws_autoscaling_group.app.name
  
  min_size         = 20
  max_size         = 100
  desired_capacity = 50
  
  # Black Friday: Last Friday of November
  recurrence = "0 0 * 11 5#4"  # Cron for 4th Friday of November
}
```

### Step 4: Capacity Dashboard

```java
// Capacity metrics endpoint
@RestController
@RequestMapping("/api/capacity")
public class CapacityController {
    
    @Autowired
    private CapacityService capacityService;
    
    @GetMapping("/summary")
    public CapacitySummary getCapacitySummary() {
        return CapacitySummary.builder()
            .compute(capacityService.getComputeCapacity())
            .database(capacityService.getDatabaseCapacity())
            .storage(capacityService.getStorageCapacity())
            .network(capacityService.getNetworkCapacity())
            .build();
    }
}

@Service
public class CapacityService {
    
    @Autowired
    private PrometheusClient prometheus;
    
    public ComputeCapacity getComputeCapacity() {
        double cpuUsage = prometheus.query("avg(cpu_usage_percent)");
        double memoryUsage = prometheus.query("avg(memory_usage_percent)");
        int currentPods = prometheus.query("count(kube_pod_status_ready)").intValue();
        int maxPods = 50;  // From HPA config
        
        return ComputeCapacity.builder()
            .cpuUtilization(cpuUsage)
            .memoryUtilization(memoryUsage)
            .currentInstances(currentPods)
            .maxInstances(maxPods)
            .headroomPercent(100 - Math.max(cpuUsage, memoryUsage))
            .status(getStatus(Math.max(cpuUsage, memoryUsage)))
            .build();
    }
    
    public DatabaseCapacity getDatabaseCapacity() {
        double connectionUsage = prometheus.query(
            "pg_stat_activity_count / pg_settings_max_connections * 100"
        );
        double storageUsage = prometheus.query("pg_database_size_bytes / pg_tablespace_size_bytes * 100");
        double cpuUsage = prometheus.query("rds_cpu_utilization");
        
        return DatabaseCapacity.builder()
            .connectionUtilization(connectionUsage)
            .storageUtilization(storageUsage)
            .cpuUtilization(cpuUsage)
            .status(getStatus(Math.max(connectionUsage, Math.max(storageUsage, cpuUsage))))
            .build();
    }
    
    private String getStatus(double utilization) {
        if (utilization < 50) return "GREEN";
        if (utilization < 70) return "YELLOW";
        if (utilization < 85) return "ORANGE";
        return "RED";
    }
}
```

---

## 5️⃣ Cost Optimization

### Right-Sizing

```
┌─────────────────────────────────────────────────────────────────┐
│                    RIGHT-SIZING ANALYSIS                         │
│                                                                  │
│  Current: m5.xlarge (4 vCPU, 16 GB RAM)                        │
│  Cost: $0.192/hour = $140/month                                │
│                                                                  │
│  Actual Usage:                                                  │
│  - CPU: 20% average, 40% peak                                  │
│  - Memory: 30% average, 50% peak                               │
│                                                                  │
│  Recommendation: m5.large (2 vCPU, 8 GB RAM)                   │
│  Cost: $0.096/hour = $70/month                                 │
│                                                                  │
│  Savings: $70/month per instance                               │
│  With 20 instances: $1,400/month = $16,800/year                │
└─────────────────────────────────────────────────────────────────┘
```

### Reserved vs On-Demand vs Spot

```
┌─────────────────────────────────────────────────────────────────┐
│                    INSTANCE PRICING STRATEGY                     │
│                                                                  │
│  Baseline Load (always needed):                                 │
│  → Reserved Instances (1-3 year commitment)                    │
│  → 30-60% discount                                             │
│                                                                  │
│  Variable Load (predictable):                                   │
│  → On-Demand Instances                                         │
│  → Pay as you go                                               │
│                                                                  │
│  Burst Load (interruptible):                                   │
│  → Spot Instances                                              │
│  → 60-90% discount                                             │
│  → Can be terminated with 2 min notice                         │
│                                                                  │
│  Example Mix:                                                   │
│  - 60% Reserved (baseline)                                     │
│  - 30% On-Demand (variable)                                    │
│  - 10% Spot (batch jobs)                                       │
│  - Savings: ~40% vs all On-Demand                              │
└─────────────────────────────────────────────────────────────────┘
```

### Cost per Request Calculation

```java
// Cost tracking service
@Service
public class CostTrackingService {
    
    public CostMetrics calculateCostPerRequest() {
        // Get monthly costs
        double computeCost = cloudCostService.getMonthlyComputeCost();
        double databaseCost = cloudCostService.getMonthlyDatabaseCost();
        double networkCost = cloudCostService.getMonthlyNetworkCost();
        double totalCost = computeCost + databaseCost + networkCost;
        
        // Get monthly requests
        long monthlyRequests = metricsService.getMonthlyRequestCount();
        
        // Calculate cost per request
        double costPerRequest = totalCost / monthlyRequests;
        double costPerMillionRequests = costPerRequest * 1_000_000;
        
        return CostMetrics.builder()
            .totalMonthlyCost(totalCost)
            .monthlyRequests(monthlyRequests)
            .costPerRequest(costPerRequest)
            .costPerMillionRequests(costPerMillionRequests)
            .breakdown(Map.of(
                "compute", computeCost,
                "database", databaseCost,
                "network", networkCost
            ))
            .build();
    }
}
```

---

## 6️⃣ Capacity Reviews

### Monthly Capacity Review Template

```markdown
# Capacity Review - January 2024

## Executive Summary
- Overall capacity status: GREEN
- Key concerns: Database approaching 70% connection utilization
- Actions needed: Plan database scaling in Q2

## Current Utilization

### Compute
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| CPU | 45% | 70% | GREEN |
| Memory | 55% | 80% | GREEN |
| Instances | 15/50 | 40/50 | GREEN |

### Database
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Connections | 65% | 80% | YELLOW |
| Storage | 40% | 70% | GREEN |
| CPU | 35% | 70% | GREEN |

### Network
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Bandwidth | 30% | 70% | GREEN |
| Requests/sec | 5,000 | 10,000 | GREEN |

## Growth Trends
- Traffic growth: 15% month-over-month
- At current growth, will hit 70% CPU in 4 months

## Forecast
| Resource | Current | 3 Months | 6 Months |
|----------|---------|----------|----------|
| CPU | 45% | 60% | 80% |
| DB Connections | 65% | 85% | 110% |
| Storage | 40% | 50% | 65% |

## Recommendations
1. **Database**: Scale to larger instance class by March
2. **Compute**: No action needed, auto-scaling sufficient
3. **Storage**: Monitor, no action needed

## Cost Analysis
- Current monthly cost: $25,000
- Projected cost (6 months): $35,000
- Cost per million requests: $2.50

## Action Items
- [ ] Create ticket for database scaling (Owner: DBA, Due: Feb 15)
- [ ] Review auto-scaling policies (Owner: Platform, Due: Feb 28)
- [ ] Update capacity dashboard (Owner: SRE, Due: Feb 10)
```

---

## 7️⃣ Tradeoffs and Common Mistakes

### Common Mistakes

**1. Not Planning for Peaks**

```
Average traffic: 1,000 RPS
Peak traffic: 10,000 RPS

Provisioned for: 1,500 RPS
Result: Outage during peak
```

**2. Over-Provisioning "Just in Case"**

```
Expected traffic: 1,000 RPS
Provisioned: 10,000 RPS

Monthly cost: $50,000
Needed cost: $5,000
Waste: $45,000/month
```

**3. Ignoring Database Limits**

```
App servers: Auto-scaled to 100
Database connections: Max 100

Result: Each app server gets 1 connection
Performance: Terrible
```

**4. Not Testing at Scale**

```
Load test: 1,000 RPS (passes)
Production: 5,000 RPS (fails)

Issues found in production:
- Connection pool exhaustion
- Memory leaks
- Slow queries
```

**5. Manual Scaling Only**

```
Traffic spike: 10x normal
Manual response time: 30 minutes
Auto-scaling response: 2 minutes

Difference: 28 minutes of degraded service
```

---

## 8️⃣ Interview Follow-Up Questions

### Q1: "How do you approach capacity planning for a new service?"

**Answer**:
Step-by-step approach:

1. **Estimate demand**: Use back-of-envelope calculations
   - Expected users
   - Requests per user
   - Peak vs average ratio

2. **Benchmark**: Load test to find single-instance capacity
   - Max RPS per instance
   - Resource limits (CPU, memory)

3. **Calculate requirements**:
   - Peak RPS / RPS per instance = instances needed
   - Add 30% headroom for safety

4. **Plan for growth**:
   - Expected growth rate
   - When will we need to scale?

5. **Set up auto-scaling**:
   - Scale based on CPU/memory/custom metrics
   - Define min/max limits

6. **Monitor and adjust**:
   - Track actual vs predicted
   - Refine estimates over time

Example: 10,000 expected users, 10 requests/user/day, peak 5x average. Each server handles 100 RPS. Need: (10,000 × 10 / 86,400) × 5 / 100 × 1.3 = 8 servers.

### Q2: "What metrics would you monitor for capacity planning?"

**Answer**:
Key metrics by category:

**Utilization**:
- CPU usage (%)
- Memory usage (%)
- Disk usage (%)
- Network bandwidth (%)

**Saturation**:
- Request queue length
- Connection pool usage
- Thread pool usage
- Database connection usage

**Throughput**:
- Requests per second
- Transactions per second
- Data processed per second

**Latency**:
- Response time (p50, p95, p99)
- Queue wait time

**Business metrics**:
- Active users
- Transactions per hour
- Revenue per hour

I'd set alerts at 70% utilization (plan scaling) and 85% (urgent scaling needed).

### Q3: "How do you handle unexpected traffic spikes?"

**Answer**:
Multiple layers of defense:

**1. Auto-scaling**:
- Scale based on CPU/memory/custom metrics
- Fast scale-up (seconds to minutes)
- Slower scale-down (prevent thrashing)

**2. Load shedding**:
- Rate limiting to protect the system
- Prioritize critical traffic
- Graceful degradation

**3. Caching**:
- Cache responses to reduce backend load
- CDN for static content
- Application-level caching

**4. Circuit breakers**:
- Fail fast if downstream is overloaded
- Prevent cascade failures

**5. Reserved capacity**:
- Keep some headroom for spikes
- Warm standby instances

**6. Monitoring and alerting**:
- Detect spikes early
- Alert on-call if needed

Key: Auto-scaling handles expected spikes. Load shedding protects against unexpected ones.

### Q4: "How do you balance cost and capacity?"

**Answer**:
Strategies for cost optimization:

**1. Right-sizing**:
- Analyze actual usage
- Downsize over-provisioned instances
- Use appropriate instance types

**2. Reserved instances**:
- Commit to baseline capacity
- 30-60% savings vs on-demand

**3. Spot instances**:
- Use for interruptible workloads
- 60-90% savings

**4. Auto-scaling**:
- Scale down during low traffic
- Don't pay for idle capacity

**5. Scheduled scaling**:
- Scale up before known events
- Scale down after

**6. Cost monitoring**:
- Track cost per request
- Set budgets and alerts

Target: 60-70% utilization on average. Lower means over-provisioned (waste). Higher means risk of overload.

### Q5: "How do you plan for a major event like Black Friday?"

**Answer**:
Planning process:

**1. Estimate traffic (weeks before)**:
- Historical data (last year's Black Friday)
- Marketing projections
- Expected growth

**2. Load test (weeks before)**:
- Test at expected peak load
- Test at 2x expected (safety margin)
- Identify bottlenecks

**3. Pre-provision (days before)**:
- Scale up infrastructure
- Warm up caches
- Pre-scale databases

**4. Monitoring (during)**:
- Enhanced monitoring
- War room staffed
- Quick response team ready

**5. Auto-scaling (during)**:
- Aggressive scale-up policies
- Higher max limits

**6. Contingency (during)**:
- Load shedding ready
- Feature flags to disable non-critical features
- Rollback plan if needed

Example: If expecting 10x normal traffic, pre-provision to 5x, auto-scale to 15x max, and have load shedding at 12x.

---

## 9️⃣ One Clean Mental Summary

Capacity planning ensures systems have enough resources to handle current and future demand. The cycle: measure current utilization, forecast growth, plan scaling, provision resources, and review regularly.

Key concepts: utilization thresholds (70% = plan, 85% = scale), headroom for spikes (30%+), and cost optimization (right-sizing, reserved instances, auto-scaling). Back-of-envelope calculations estimate requirements; load testing validates them.

The key insight: Capacity planning is about being proactive, not reactive. Know your limits before you hit them. Measure, forecast, and scale before users notice problems. Balance cost and capacity: too little causes outages, too much wastes money.

