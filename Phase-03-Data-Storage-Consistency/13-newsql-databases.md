# NewSQL Databases: Distributed SQL at Scale

## 0️⃣ Prerequisites

Before diving into NewSQL databases, you should understand:

- **SQL vs NoSQL**: Traditional relational databases vs distributed NoSQL (covered in Topic 1).
- **ACID Transactions**: Atomicity, Consistency, Isolation, Durability (covered in Topic 7).
- **Database Sharding**: Distributing data across multiple servers (covered in Topic 5).
- **Consistency Models**: Strong vs eventual consistency (covered in Topic 6).
- **Consensus Protocols**: Paxos/Raft for distributed agreement (covered in Phase 1).

**Quick refresher on the problem**: Traditional SQL databases don't scale horizontally. NoSQL databases scale but sacrifice SQL and ACID. NewSQL databases aim to provide both: SQL + ACID + horizontal scalability.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine you're building a global financial application:

```
Requirements:
- ACID transactions (can't lose money)
- SQL queries (complex reporting)
- Global distribution (users worldwide)
- Horizontal scalability (growing data)
- Low latency (real-time operations)

Traditional SQL (PostgreSQL, MySQL):
✓ ACID transactions
✓ SQL queries
✗ Global distribution (single region)
✗ Horizontal scalability (vertical only)
✗ Low latency globally (single location)

NoSQL (Cassandra, DynamoDB):
✗ ACID transactions (eventual consistency)
✗ SQL queries (limited query language)
✓ Global distribution
✓ Horizontal scalability
✓ Low latency globally

Neither option works!
```

### The Traditional Trade-off

```
┌─────────────────────────────────────────────────────────────┐
│              THE SCALABILITY DILEMMA                         │
│                                                              │
│  Traditional SQL:                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Single powerful server                               │   │
│  │ ├── All data in one place                           │   │
│  │ ├── ACID is straightforward                         │   │
│  │ ├── Complex queries work                            │   │
│  │ └── But: Can't scale beyond one machine            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Sharded SQL:                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Data split across servers                           │   │
│  │ ├── Each shard is independent                       │   │
│  │ ├── Cross-shard transactions are hard               │   │
│  │ ├── JOINs across shards are slow                   │   │
│  │ └── Application manages routing                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  NoSQL:                                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Distributed by design                               │   │
│  │ ├── Scales horizontally                             │   │
│  │ ├── But: No ACID (eventual consistency)            │   │
│  │ ├── No complex queries                              │   │
│  │ └── Application handles consistency                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What NewSQL Promises

```
┌─────────────────────────────────────────────────────────────┐
│                  NEWSQL PROMISE                              │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Distributed architecture (like NoSQL)               │   │
│  │ ├── Horizontal scalability                          │   │
│  │ ├── Automatic sharding                              │   │
│  │ ├── Geographic distribution                         │   │
│  │                                                      │   │
│  │ PLUS                                                 │   │
│  │                                                      │   │
│  │ SQL and ACID (like traditional SQL)                 │   │
│  │ ├── Full SQL support                                │   │
│  │ ├── ACID transactions                               │   │
│  │ ├── Strong consistency                              │   │
│  │ └── Familiar tooling                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  "The best of both worlds"                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Real Examples

**Google Spanner**: Powers Google's advertising, Google Play, and other critical systems. First to achieve global strong consistency.

**CockroachDB**: Used by companies like Bose, Comcast, and Netflix for globally distributed applications.

**TiDB**: Popular in Asia, used by companies like PingCAP, Xiaomi, and JD.com.

---

## 2️⃣ Intuition and Mental Model

### The Global Bank Analogy

**Traditional SQL = Single Branch Bank**

```
┌─────────────────────────────────────────────────────────────┐
│                 SINGLE BRANCH BANK                           │
│                                                              │
│  All transactions happen at headquarters                    │
│  ├── One ledger book                                        │
│  ├── Transactions are simple (one book)                     │
│  ├── But customers far away have high latency              │
│  └── If HQ is down, everything stops                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**NoSQL = Independent Branch Banks**

```
┌─────────────────────────────────────────────────────────────┐
│              INDEPENDENT BRANCH BANKS                        │
│                                                              │
│  Each branch has its own ledger                             │
│  ├── Fast local transactions                                │
│  ├── Branches sync periodically                             │
│  ├── But: Transfer between branches is complicated         │
│  ├── Balances might be temporarily inconsistent            │
│  └── "Eventually" everything matches up                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**NewSQL = Synchronized Global Bank**

```
┌─────────────────────────────────────────────────────────────┐
│              SYNCHRONIZED GLOBAL BANK                        │
│                                                              │
│  Multiple branches, one synchronized ledger                 │
│  ├── Each branch can accept transactions                    │
│  ├── Branches coordinate in real-time                       │
│  ├── Every branch sees consistent balances                  │
│  ├── Transfer between branches is seamless                  │
│  └── If one branch is down, others continue                │
│                                                              │
│  How? Consensus protocol (all branches agree)               │
│       + Synchronized clocks (ordering transactions)         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Google Spanner: TrueTime

**The Problem**: In a distributed system, how do you order transactions globally?

**Traditional approach**: Logical clocks (Lamport timestamps)
- Works but requires coordination
- Limits throughput

**Spanner's innovation**: TrueTime API

```
┌─────────────────────────────────────────────────────────────┐
│                    TRUETIME API                              │
│                                                              │
│  Every Google datacenter has:                               │
│  ├── GPS receivers (satellite time)                         │
│  ├── Atomic clocks (local precision)                        │
│  └── Time servers combining both                            │
│                                                              │
│  TrueTime returns: [earliest, latest]                       │
│  "The actual time is somewhere in this interval"            │
│                                                              │
│  Example:                                                    │
│  TT.now() → [10:00:00.000, 10:00:00.007]                   │
│  Uncertainty: 7 milliseconds                                │
│                                                              │
│  Commit-wait rule:                                           │
│  Before committing, wait until uncertainty passes           │
│  If commit at [10:00:00.000, 10:00:00.007]:                │
│  Wait until 10:00:00.007 before acknowledging               │
│                                                              │
│  Result: If transaction A commits before B starts,          │
│          A's timestamp < B's timestamp (globally!)          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Spanner Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 SPANNER ARCHITECTURE                         │
│                                                              │
│  Universe (Global):                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  Zone US-East          Zone EU-West                 │   │
│  │  ┌──────────────┐     ┌──────────────┐             │   │
│  │  │ Spanserver 1 │     │ Spanserver 1 │             │   │
│  │  │ Spanserver 2 │     │ Spanserver 2 │             │   │
│  │  │ Spanserver 3 │     │ Spanserver 3 │             │   │
│  │  └──────────────┘     └──────────────┘             │   │
│  │                                                      │   │
│  │  Zone Asia                                           │   │
│  │  ┌──────────────┐                                   │   │
│  │  │ Spanserver 1 │                                   │   │
│  │  │ Spanserver 2 │                                   │   │
│  │  │ Spanserver 3 │                                   │   │
│  │  └──────────────┘                                   │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Each table is split into tablets (shards)                  │
│  Each tablet is replicated across zones (Paxos)             │
│  One replica is the Paxos leader for writes                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### CockroachDB: Raft + Hybrid Logical Clocks

**CockroachDB approach**: Similar to Spanner but without specialized hardware.

```
┌─────────────────────────────────────────────────────────────┐
│              COCKROACHDB ARCHITECTURE                        │
│                                                              │
│  Key differences from Spanner:                              │
│  ├── Uses Raft instead of Paxos (simpler)                  │
│  ├── Hybrid Logical Clocks (HLC) instead of TrueTime       │
│  ├── Runs on commodity hardware                            │
│  └── Open source                                            │
│                                                              │
│  Data organization:                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Table → Ranges (64MB chunks)                        │   │
│  │ Each range replicated 3x (Raft group)               │   │
│  │ Ranges automatically split/merge based on size      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Hybrid Logical Clock:                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Combines physical time + logical counter            │   │
│  │ HLC = (wall_time, logical_counter)                  │   │
│  │                                                      │   │
│  │ If wall clocks are close: Use physical time         │   │
│  │ If clock skew detected: Increment logical counter   │   │
│  │                                                      │   │
│  │ Provides causal ordering without special hardware   │   │
│  │ But: Requires bounded clock skew (default 500ms)    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### TiDB: MySQL Compatible

```
┌─────────────────────────────────────────────────────────────┐
│                   TIDB ARCHITECTURE                          │
│                                                              │
│  TiDB Server (SQL Layer):                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Stateless SQL processing                            │   │
│  │ ├── Parse SQL                                       │   │
│  │ ├── Optimize query plan                             │   │
│  │ ├── Execute distributed queries                     │   │
│  │ └── MySQL protocol compatible                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  TiKV (Storage Layer):                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Distributed key-value storage                       │   │
│  │ ├── Data split into regions                         │   │
│  │ ├── Each region replicated via Raft                │   │
│  │ ├── Supports ACID transactions                      │   │
│  │ └── RocksDB as local storage engine                │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  PD (Placement Driver):                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Cluster manager                                      │   │
│  │ ├── Stores metadata                                 │   │
│  │ ├── Schedules region placement                      │   │
│  │ └── Generates transaction timestamps               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Distributed Transactions

**How NewSQL handles distributed transactions**:

```
┌─────────────────────────────────────────────────────────────┐
│           DISTRIBUTED TRANSACTION FLOW                       │
│                                                              │
│  Transaction: Transfer $100 from Account A to Account B    │
│  Account A is on Node 1, Account B is on Node 2            │
│                                                              │
│  Step 1: Begin transaction, get timestamp                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Coordinator assigns timestamp: T=100                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Step 2: Read and lock                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Node 1: Read Account A, acquire write lock          │   │
│  │ Node 2: Read Account B, acquire write lock          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Step 3: Prepare (2PC phase 1)                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Node 1: Prepare A = A - 100, vote YES               │   │
│  │ Node 2: Prepare B = B + 100, vote YES               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Step 4: Commit (2PC phase 2)                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ All voted YES → Commit at timestamp T=100           │   │
│  │ Node 1: Commit, release lock                        │   │
│  │ Node 2: Commit, release lock                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Key insight: Timestamp ordering ensures serializability    │
│               Raft/Paxos ensures durability                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

### Scenario 1: Global E-commerce with Spanner

```
Setup:
- Users in US, EU, Asia
- Product catalog and orders
- Spanner with zones in each region

User in Tokyo buys product:

T=0ms:   Request hits Tokyo zone
T=1ms:   Read product (local replica) → Fast!
T=2ms:   Begin transaction
T=3ms:   Write order (Tokyo is Paxos leader for this range)
T=5ms:   Paxos replication to US, EU (parallel)
T=80ms:  Majority acknowledge (Tokyo + 1 other)
T=85ms:  Commit-wait (TrueTime uncertainty)
T=90ms:  Transaction committed, response to user

Total latency: ~90ms (acceptable for purchase)
Read latency: ~2ms (local replica)

Compare to single-region:
- User in Tokyo → US datacenter: 150ms round-trip
- Total latency: ~300ms (unacceptable)
```

### Scenario 2: CockroachDB Multi-Region

```
Setup:
- 3 regions: US-East, US-West, EU
- User data partitioned by region
- Global tables replicated everywhere

User in EU updates profile:

T=0ms:   Request hits EU node
T=1ms:   Route to EU range (user data locality)
T=2ms:   Acquire write intent
T=5ms:   Raft replication within EU (3 nodes)
T=10ms:  Majority in EU acknowledge
T=15ms:  Commit, async replication to US regions

Total latency: ~15ms (fast, local Raft group)

Cross-region transaction (EU user buys from US seller):

T=0ms:   Begin transaction
T=1ms:   Read EU user (local)
T=150ms: Read US seller (cross-region)
T=155ms: Write to both ranges
T=300ms: 2PC across regions
T=310ms: Commit

Total latency: ~310ms (cross-region is slow)
```

### Scenario 3: TiDB MySQL Migration

```
Existing system:
- MySQL with 500GB data
- 10,000 QPS
- Hitting scaling limits

Migration to TiDB:

Step 1: Set up TiDB cluster
- 3 TiDB servers (SQL layer)
- 3 TiKV nodes (storage)
- 3 PD nodes (coordination)

Step 2: Migrate data
- Use TiDB Data Migration tool
- Replicate from MySQL in real-time

Step 3: Switch traffic
- Point application to TiDB
- MySQL protocol compatible → No code changes!

After migration:
- Same queries work
- Automatic sharding (no manual partitioning)
- Scales to 100,000 QPS by adding nodes
- ACID transactions still work
```

---

## 5️⃣ How Engineers Actually Use This in Production

### At Major Companies

**Google (Spanner)**:
- AdWords: Billions of transactions per day
- Google Play: Global app distribution
- Google Photos: Metadata storage

**Cockroach Labs (CockroachDB)**:
- Bose: Global e-commerce
- Comcast: Customer data platform
- DoorDash: Order management

**PingCAP (TiDB)**:
- JD.com: E-commerce transactions
- Xiaomi: User data storage
- Bank of Beijing: Financial transactions

### When to Choose NewSQL

```
┌─────────────────────────────────────────────────────────────┐
│              NEWSQL DECISION FRAMEWORK                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Choose NewSQL when you need:                               │
│  ├── SQL + ACID (can't compromise)                         │
│  ├── Horizontal scalability (beyond single server)         │
│  ├── Geographic distribution (global users)                │
│  └── High availability (multi-region failover)             │
│                                                              │
│  Stick with traditional SQL when:                           │
│  ├── Single region is sufficient                           │
│  ├── Data fits on one server (< 1TB)                       │
│  ├── Team is small (operational complexity)                │
│  └── Latency requirements are very strict                  │
│                                                              │
│  Consider NoSQL when:                                        │
│  ├── Eventual consistency is acceptable                    │
│  ├── Simple key-value access patterns                      │
│  ├── Extreme write throughput needed                       │
│  └── Schema flexibility is important                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ How to Implement or Apply It

### CockroachDB with Spring Boot

```java
package com.example.newsql;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import jakarta.persistence.*;
import java.math.BigDecimal;

@SpringBootApplication
public class CockroachDBApplication {
    public static void main(String[] args) {
        SpringApplication.run(CockroachDBApplication.class, args);
    }
}

@Entity
@Table(name = "accounts")
public class Account {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal balance;
    
    @Column(name = "region")
    private String region;  // For geo-partitioning
    
    // Getters, setters...
}

@Repository
public interface AccountRepository extends JpaRepository<Account, Long> {
    // CockroachDB handles distribution automatically
}

@Service
public class TransferService {
    
    private final AccountRepository accountRepository;
    
    public TransferService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }
    
    /**
     * Transfer money between accounts.
     * Works across regions with distributed transactions.
     */
    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId)
            .orElseThrow(() -> new RuntimeException("Account not found"));
        Account to = accountRepository.findById(toId)
            .orElseThrow(() -> new RuntimeException("Account not found"));
        
        if (from.getBalance().compareTo(amount) < 0) {
            throw new RuntimeException("Insufficient funds");
        }
        
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
        
        accountRepository.save(from);
        accountRepository.save(to);
        
        // CockroachDB handles:
        // - Distributed locking
        // - 2PC if accounts are on different nodes
        // - Automatic retries on contention
    }
}
```

**Application Configuration**:

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:26257/bank?sslmode=disable
    username: root
    password: ""
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

### CockroachDB Geo-Partitioning

```sql
-- Create geo-partitioned table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email STRING NOT NULL,
    name STRING,
    region STRING NOT NULL,
    created_at TIMESTAMP DEFAULT now()
) LOCALITY REGIONAL BY ROW AS region;

-- Define regions
ALTER DATABASE mydb PRIMARY REGION "us-east1";
ALTER DATABASE mydb ADD REGION "us-west1";
ALTER DATABASE mydb ADD REGION "europe-west1";

-- Data automatically routes to correct region
INSERT INTO users (email, name, region) 
VALUES ('alice@example.com', 'Alice', 'us-east1');

-- Queries in us-east1 for this user are fast (local)
-- Queries from europe-west1 for this user are slower (cross-region)
```

### TiDB with JDBC

```java
package com.example.tidb;

import java.sql.*;
import java.util.Properties;

public class TiDBExample {
    
    private static final String URL = "jdbc:mysql://localhost:4000/test";
    private static final String USER = "root";
    private static final String PASSWORD = "";
    
    public static void main(String[] args) throws SQLException {
        Properties props = new Properties();
        props.setProperty("user", USER);
        props.setProperty("password", PASSWORD);
        props.setProperty("useSSL", "false");
        
        try (Connection conn = DriverManager.getConnection(URL, props)) {
            // Create table (same as MySQL!)
            try (Statement stmt = conn.createStatement()) {
                stmt.execute("""
                    CREATE TABLE IF NOT EXISTS orders (
                        id BIGINT PRIMARY KEY AUTO_INCREMENT,
                        user_id BIGINT NOT NULL,
                        total DECIMAL(10, 2) NOT NULL,
                        status VARCHAR(20) DEFAULT 'pending',
                        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                        INDEX idx_user (user_id),
                        INDEX idx_status (status)
                    )
                    """);
            }
            
            // Insert with transaction
            conn.setAutoCommit(false);
            try {
                try (PreparedStatement ps = conn.prepareStatement(
                        "INSERT INTO orders (user_id, total) VALUES (?, ?)")) {
                    ps.setLong(1, 123);
                    ps.setBigDecimal(2, new java.math.BigDecimal("99.99"));
                    ps.executeUpdate();
                }
                
                // TiDB distributes data automatically
                // No need to specify shard key
                
                conn.commit();
            } catch (SQLException e) {
                conn.rollback();
                throw e;
            }
            
            // Query (same as MySQL!)
            try (Statement stmt = conn.createStatement();
                 ResultSet rs = stmt.executeQuery(
                     "SELECT * FROM orders WHERE user_id = 123")) {
                while (rs.next()) {
                    System.out.println("Order: " + rs.getLong("id") + 
                        ", Total: " + rs.getBigDecimal("total"));
                }
            }
        }
    }
}
```

### Spanner with Java Client

```java
package com.example.spanner;

import com.google.cloud.spanner.*;

public class SpannerExample {
    
    public static void main(String[] args) {
        SpannerOptions options = SpannerOptions.newBuilder().build();
        Spanner spanner = options.getService();
        
        String instanceId = "my-instance";
        String databaseId = "my-database";
        
        DatabaseClient dbClient = spanner.getDatabaseClient(
            DatabaseId.of(options.getProjectId(), instanceId, databaseId));
        
        // Read-write transaction
        dbClient.readWriteTransaction().run(transaction -> {
            // Read account balance
            Struct row = transaction.readRow(
                "Accounts",
                Key.of("account-123"),
                Arrays.asList("balance"));
            
            long balance = row.getLong("balance");
            
            // Update balance
            transaction.buffer(
                Mutation.newUpdateBuilder("Accounts")
                    .set("account_id").to("account-123")
                    .set("balance").to(balance - 100)
                    .build());
            
            return null;
        });
        
        // Read-only transaction (uses snapshot)
        try (ReadOnlyTransaction readTx = 
                dbClient.readOnlyTransaction(TimestampBound.strong())) {
            
            ResultSet resultSet = readTx.executeQuery(
                Statement.of("SELECT * FROM Accounts WHERE region = 'us-east'"));
            
            while (resultSet.next()) {
                System.out.println("Account: " + resultSet.getString("account_id"));
            }
        }
        
        spanner.close();
    }
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistake 1: Ignoring Latency

```
Scenario: Single-region mindset in multi-region deployment

WRONG assumption:
"Transactions will be as fast as PostgreSQL"

Reality:
- Local transactions: 5-20ms (similar to PostgreSQL)
- Cross-region transactions: 100-300ms (network latency)
- Global consistency has a cost

Solution:
- Design for data locality (partition by region)
- Use read replicas for read-heavy workloads
- Accept eventual consistency where possible
```

### Common Mistake 2: Wrong Data Model

```sql
-- WRONG: Frequent cross-region JOINs
SELECT o.*, u.name 
FROM orders o 
JOIN users u ON o.user_id = u.id
WHERE o.created_at > NOW() - INTERVAL '1 day';

-- If orders and users are in different regions: SLOW

-- RIGHT: Denormalize or co-locate
-- Option 1: Store user_name in orders table
-- Option 2: Partition both tables by same key (user_id)
```

### Common Mistake 3: Over-Engineering

```
Scenario: Small startup, 10,000 users

WRONG: Deploy CockroachDB cluster across 3 regions
- High operational complexity
- Unnecessary cost
- Team doesn't have expertise

RIGHT: Start with PostgreSQL
- Simpler to operate
- Migrate to NewSQL when you actually need it
- Most startups never reach that scale
```

### Performance Gotchas

```
┌─────────────────────────────────────────────────────────────┐
│              NEWSQL PERFORMANCE GOTCHAS                      │
│                                                              │
│  1. Transaction contention                                   │
│     - High contention = many retries                        │
│     - Design to minimize conflicts                          │
│     - Use optimistic locking where possible                 │
│                                                              │
│  2. Clock skew (CockroachDB)                                │
│     - Default max skew: 500ms                               │
│     - Higher skew = longer commit wait                      │
│     - Use NTP, monitor clock drift                          │
│                                                              │
│  3. Range hotspots                                           │
│     - Sequential IDs create hot ranges                      │
│     - Use UUIDs or hash-prefixed keys                       │
│                                                              │
│  4. Cross-region queries                                     │
│     - Always slower than local                              │
│     - Use follower reads for stale-OK queries              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8️⃣ When NOT to Use This

### When NOT to Use NewSQL

1. **Single-region applications**
   - Traditional SQL is simpler and faster
   - No benefit from distribution overhead

2. **Small data volumes**
   - Under 1TB, PostgreSQL scales fine
   - NewSQL adds complexity without benefit

3. **Extreme low-latency requirements**
   - Sub-millisecond latency needs
   - Consensus adds unavoidable latency

4. **Simple key-value workloads**
   - NoSQL (Redis, DynamoDB) is more efficient
   - SQL overhead not needed

5. **Budget constraints**
   - NewSQL requires more infrastructure
   - Operational expertise is expensive

### Signs You Might Need NewSQL

- Outgrowing single PostgreSQL instance
- Need ACID across multiple regions
- Sharding application logic is too complex
- Global users need low latency
- High availability across regions is critical

---

## 9️⃣ Comparison with Alternatives

### Database Type Comparison

| Feature | Traditional SQL | NoSQL | NewSQL |
|---------|----------------|-------|--------|
| SQL Support | Full | Limited/None | Full |
| ACID | Yes | Usually No | Yes |
| Horizontal Scale | Limited | Yes | Yes |
| Strong Consistency | Yes | Usually No | Yes |
| Complexity | Low | Medium | High |
| Latency | Lowest | Low | Medium |

### NewSQL Database Comparison

| Database | Protocol | Consistency | Open Source | Best For |
|----------|----------|-------------|-------------|----------|
| Spanner | gRPC | External | No | Google Cloud |
| CockroachDB | PostgreSQL | Serializable | Yes | Multi-region |
| TiDB | MySQL | Snapshot | Yes | MySQL migration |
| YugabyteDB | PostgreSQL | Serializable | Yes | PostgreSQL migration |
| VoltDB | Custom | Serializable | Partial | High throughput |

### Cost Comparison

```
Rough comparison for 10TB, 3 regions:

Traditional SQL (PostgreSQL):
- Single region: $2,000/month
- Multi-region (manual): $8,000/month + engineering time

NoSQL (DynamoDB):
- Global tables: $5,000/month
- No ACID for complex transactions

NewSQL (CockroachDB):
- Self-hosted: $6,000/month + operations
- Managed: $10,000/month

NewSQL (Spanner):
- Managed: $15,000/month
- Includes operations
```

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: What is NewSQL and how is it different from traditional SQL databases?**

**Answer:**
NewSQL databases combine the ACID guarantees and SQL interface of traditional databases with the horizontal scalability of NoSQL systems.

**Traditional SQL (PostgreSQL, MySQL):**
- Strong ACID transactions
- Full SQL support
- Limited to single server (vertical scaling)
- Can't distribute data across regions easily

**NewSQL (Spanner, CockroachDB):**
- Same ACID transactions
- Same SQL interface
- Distributes data across many servers (horizontal scaling)
- Works across multiple regions with strong consistency

The key innovation is using consensus protocols (like Raft or Paxos) to coordinate distributed transactions while maintaining ACID guarantees.

**Q2: What is TrueTime and why is it important for Spanner?**

**Answer:**
TrueTime is Google's globally synchronized clock API that returns a time interval [earliest, latest] instead of a single timestamp.

**Why it matters:**
In distributed systems, ordering transactions correctly is hard because server clocks can drift. If Server A's clock is ahead of Server B's, a transaction on B might get a "later" timestamp even though it happened first.

**How TrueTime solves this:**
1. Uses GPS receivers and atomic clocks in every datacenter
2. Returns a range: "The real time is between X and Y"
3. Before committing, waits until the uncertainty passes
4. Guarantees: If transaction A commits before B starts, A's timestamp < B's timestamp

This enables Spanner to provide "external consistency" (linearizability) globally, which was previously thought impossible at scale.

### L5 (Mid-Level) Questions

**Q3: How does CockroachDB achieve distributed transactions without specialized hardware like Spanner?**

**Answer:**
CockroachDB uses several techniques:

**1. Hybrid Logical Clocks (HLC):**
- Combines physical time with a logical counter
- If clocks are synchronized: Uses physical time
- If clock skew detected: Increments logical counter
- Requires bounded clock skew (default 500ms)

**2. Raft Consensus:**
- Each data range is a Raft group
- 3+ replicas per range
- Leader handles writes, replicates to followers
- Majority must acknowledge before commit

**3. Distributed Transactions:**
- Uses 2PC (two-phase commit) across ranges
- Transaction coordinator manages the protocol
- Write intents mark pending changes
- Commit or rollback based on all participants

**Trade-offs vs Spanner:**
- Doesn't need special hardware (runs anywhere)
- Slightly weaker guarantees (bounded staleness possible)
- Requires careful clock synchronization (NTP)

**Q4: When would you choose CockroachDB over PostgreSQL?**

**Answer:**
Choose CockroachDB when:

1. **Scale beyond single server:**
   - Data > 1TB and growing
   - Need to distribute across machines

2. **Multi-region requirements:**
   - Users in multiple continents
   - Need data locality for low latency
   - Require cross-region disaster recovery

3. **High availability:**
   - Can't afford downtime for failover
   - Need automatic recovery from failures

4. **Distributed transactions:**
   - Business logic requires ACID across shards
   - Can't handle eventual consistency

Stick with PostgreSQL when:
- Single region is sufficient
- Data fits on one server
- Team is small (simpler operations)
- Latency requirements are very strict

### L6 (Senior) Questions

**Q5: Design a globally distributed payment system using NewSQL.**

**Answer:**
**Requirements:**
- ACID transactions (can't lose money)
- Global users (low latency everywhere)
- High availability (99.99% uptime)
- Regulatory compliance (data residency)

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│              GLOBAL PAYMENT SYSTEM                           │
│                                                              │
│  Regions: US-East, US-West, EU, Asia                        │
│                                                              │
│  Data Partitioning:                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Accounts: Partitioned by user's home region         │   │
│  │ - US users → US region (data residency)             │   │
│  │ - EU users → EU region (GDPR compliance)            │   │
│  │                                                      │   │
│  │ Transactions: Partitioned by sender's region        │   │
│  │ - Local transactions: Fast (single region)          │   │
│  │ - Cross-region: Slower but still ACID              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Database: CockroachDB or Spanner                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ - 3 replicas per region (local HA)                  │   │
│  │ - Cross-region replication for DR                   │   │
│  │ - Geo-partitioning for data residency              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Transaction Flow (same region):**
1. Request hits local region
2. Read account balance (local, fast)
3. Distributed transaction within region
4. Raft consensus (3 nodes)
5. Commit in ~20ms

**Transaction Flow (cross-region):**
1. Request hits sender's region
2. 2PC coordinator in sender's region
3. Read/lock sender account (local)
4. Read/lock receiver account (cross-region, 100ms)
5. Prepare both participants
6. Commit if both ready
7. Total: ~300ms

**Optimizations:**
- Cache frequently accessed data
- Use follower reads for balance checks (stale OK)
- Batch small transactions
- Async notification after commit

**Q6: How would you migrate from a sharded MySQL setup to TiDB?**

**Answer:**
**Current state:**
- 10 MySQL shards
- Application-level routing
- Cross-shard transactions handled manually

**Migration strategy:**

**Phase 1: Preparation**
1. Set up TiDB cluster (3 TiDB, 3 TiKV, 3 PD)
2. Create schema in TiDB (same as MySQL)
3. Set up TiDB Data Migration (DM) for replication

**Phase 2: Data sync**
1. Full snapshot of all shards to TiDB
2. Enable binlog replication from all shards
3. TiDB merges data from all shards
4. Verify data consistency

**Phase 3: Shadow traffic**
1. Send read traffic to both MySQL and TiDB
2. Compare results
3. Fix any discrepancies
4. Measure TiDB performance

**Phase 4: Cutover**
1. Stop writes to MySQL
2. Wait for replication to catch up
3. Switch application to TiDB
4. Monitor closely

**Phase 5: Cleanup**
1. Remove sharding logic from application
2. Decommission MySQL shards
3. Optimize TiDB configuration

**Benefits after migration:**
- No more application-level sharding
- Cross-shard transactions work automatically
- Easier to scale (add TiKV nodes)
- Familiar MySQL interface

---

## 1️⃣1️⃣ One Clean Mental Summary

NewSQL databases provide the SQL interface and ACID transactions of traditional databases with the horizontal scalability of NoSQL. They achieve this through distributed consensus (Raft/Paxos) and innovative clock synchronization (TrueTime or HLC).

Key players: Google Spanner (pioneered the space, uses TrueTime), CockroachDB (open-source, PostgreSQL-compatible), TiDB (MySQL-compatible, popular in Asia).

Use NewSQL when you need: SQL + ACID + horizontal scale + global distribution. Don't use when: single region is enough, data is small, or team can't handle operational complexity.

The main trade-off is latency: local transactions are fast, but cross-region transactions add network latency. Design for data locality (partition by region) to minimize cross-region operations.

Migration path: Most NewSQL databases are wire-compatible with PostgreSQL or MySQL, making migration easier. Start with a shadow deployment, replicate data, verify consistency, then cut over.

