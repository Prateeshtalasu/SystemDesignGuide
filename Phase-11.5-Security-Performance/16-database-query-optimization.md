# Database Query Optimization

## 0️⃣ Prerequisites

- Understanding of SQL queries and database fundamentals (covered in Phase 3)
- Knowledge of database indexing (covered in Phase 3: Database Indexing)
- Understanding of N+1 query problem (covered in topic 15)
- Basic familiarity with database connection pooling (covered in topic 13)
- Understanding of EXPLAIN plans and query execution

**Quick refresher**: Database indexing creates data structures (like B-trees) that allow databases to quickly locate rows without scanning entire tables. The N+1 problem occurs when an application makes one query to fetch a list, then N additional queries (one per item) to fetch related data.

## 1️⃣ What problem does this exist to solve?

### The Pain Point

Modern applications query databases millions of times per day. A poorly optimized query can:

1. **Take seconds instead of milliseconds** - Users wait, timeouts occur, systems fail
2. **Consume excessive resources** - One bad query can lock tables, fill memory, exhaust connections
3. **Create cascading failures** - Slow queries back up connection pools, causing other requests to fail
4. **Cost money** - More CPU, more database instances, higher cloud bills

### What Systems Look Like Without Optimization

Without query optimization:

- **Simple user lookup takes 5 seconds** because it scans 100 million rows
- **Dashboard loads in 30 seconds** because it runs 20 sequential queries
- **Database CPU stays at 100%** even during low traffic
- **Connection pool exhaustion** because queries hold connections too long
- **Production incidents** during traffic spikes when slow queries compound

### Real Examples

**Example 1**: E-commerce site during Black Friday. A query to find "products on sale" scans the entire products table (10 million rows) for each request. Result: Database crashes, site goes down, millions in lost revenue.

**Example 2**: Social media app loading a user's feed. The app fetches 50 posts, then makes 50 separate queries to get author info for each post (N+1 problem). Result: Feed takes 8 seconds to load, users abandon.

**Example 3**: Analytics dashboard. A report query joins 5 tables without proper indexes. Result: Query runs for 2 minutes, times out, shows error to users.

## 2️⃣ Intuition and Mental Model

**Think of a library without a card catalog:**

Without optimization, finding a book means walking through every aisle, checking every shelf, opening every book. This is a **full table scan**.

With optimization, you use:
- **Card catalog (index)** - Know exactly which shelf has your book
- **Organization system** - Books sorted alphabetically (clustered index)
- **Reading lists (materialized views)** - Pre-computed popular book combinations
- **Librarian assistance (query planner)** - Expert who knows the fastest route

A database optimizer is like a smart librarian who:
1. Analyzes your question (query)
2. Chooses the best strategy (execution plan)
3. Uses shortcuts (indexes) when available
4. Avoids unnecessary work (filters early, limits results)

**The mental model**: Query optimization is about giving the database the right tools (indexes) and writing queries that let the database use them effectively.

## 3️⃣ How it works internally

### The Query Execution Pipeline

When you send a SQL query, the database goes through these steps:

#### Step 1: Parsing
```
SELECT u.name, p.title 
FROM users u 
JOIN posts p ON u.id = p.user_id 
WHERE u.email = 'user@example.com'
```

The database parses this into an **abstract syntax tree (AST)**:
- Root: SELECT
  - Columns: u.name, p.title
  - FROM: users u
  - JOIN: posts p ON u.id = p.user_id
  - WHERE: u.email = 'user@example.com'

#### Step 2: Query Rewriting
The optimizer rewrites the query for efficiency:
- Predicate pushdown: Move WHERE filters closer to data sources
- Constant folding: Evaluate constant expressions once
- Join reordering: Rearrange joins for optimal order
- Subquery flattening: Convert subqueries to joins when possible

#### Step 3: Cost-Based Optimization
The optimizer generates multiple execution plans and estimates cost:

**Plan A**: Full table scan users (cost: 10,000), then filter
**Plan B**: Index scan on users.email (cost: 10), then join
**Plan C**: Index scan posts.user_id (cost: 5,000), then join users

The optimizer chooses Plan B (lowest cost).

#### Step 4: Execution Plan Generation
Creates a tree of operations:

```
Index Scan (users.email = 'user@example.com')
  ↓
Nested Loop Join (users.id = posts.user_id)
  ↓
Index Scan (posts.user_id = users.id)
  ↓
Project (name, title)
```

#### Step 5: Execution
The database executes the plan:
1. Use index on users.email to find matching row (1 row)
2. Use users.id to find posts via index on posts.user_id (5 rows)
3. Combine results, project columns
4. Return 5 rows

### EXPLAIN Plan Anatomy

The `EXPLAIN` command shows the execution plan:

```sql
EXPLAIN ANALYZE
SELECT u.name, p.title 
FROM users u 
JOIN posts p ON u.id = p.user_id 
WHERE u.email = 'user@example.com';
```

**Output interpretation**:

```
Nested Loop  (cost=0.85..8.95 rows=5 width=64) (actual time=0.123..0.456 rows=5 loops=1)
  ->  Index Scan using idx_users_email on users u  (cost=0.42..2.62 rows=1 width=36)
        Index Cond: (email = 'user@example.com'::text)
        (actual time=0.089..0.090 rows=1 loops=1)
  ->  Index Scan using idx_posts_user_id on posts p  (cost=0.43..6.25 rows=5 width=36)
        Index Cond: (user_id = u.id)
        (actual time=0.032..0.073 rows=5 loops=1)
Planning Time: 0.234 ms
Execution Time: 0.489 ms
```

**Key metrics**:
- **cost**: Estimated cost (startup..total)
- **rows**: Estimated rows returned
- **actual time**: Real execution time (startup..total per loop)
- **actual rows**: Real rows returned
- **Index Scan**: Using an index (good)
- **Seq Scan**: Sequential scan (usually bad for large tables)

### Index Selection Strategy

The optimizer chooses indexes based on:

1. **Selectivity**: How many rows match
   - High selectivity (few matches) → index is beneficial
   - Low selectivity (many matches) → index might not help

2. **Query patterns**: Which columns appear in WHERE, JOIN, ORDER BY

3. **Index statistics**: Database maintains stats on data distribution

4. **Cost estimation**: Compares index scan vs sequential scan costs

## 4️⃣ Simulation-first explanation

### Scenario: Simple User Lookup

**Start**: 1 application server, 1 database, 1 user table with 1 million rows

**Table structure**:
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(255),
    created_at TIMESTAMP
);

-- No index on email initially
```

**Query**:
```sql
SELECT * FROM users WHERE email = 'alice@example.com';
```

**Without index (Sequential Scan)**:

1. Database receives query
2. Planner decides: "No index on email, must scan all rows"
3. Database reads disk pages sequentially
4. For each row, checks `email = 'alice@example.com'`
5. After checking 1 million rows, finds match at row 847,392
6. Returns result
7. **Total time**: ~500ms, **Rows examined**: 1,000,000

**With index (Index Scan)**:

```sql
CREATE INDEX idx_users_email ON users(email);
```

1. Database receives query
2. Planner decides: "Index exists on email, use it"
3. Database searches B-tree index for 'alice@example.com'
4. Index traversal finds key in 3-4 steps (logarithmic search)
5. Index points to exact row location
6. Database fetches row from table
7. Returns result
8. **Total time**: ~0.5ms, **Rows examined**: 1

**Performance improvement**: 1000x faster (500ms → 0.5ms)

### Scenario: JOIN Query

**Tables**:
```sql
-- 1 million users
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255)
);

-- 10 million posts
CREATE TABLE posts (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    title VARCHAR(255),
    content TEXT
);
```

**Query**:
```sql
SELECT u.email, p.title 
FROM users u 
JOIN posts p ON u.id = p.user_id 
WHERE u.email = 'alice@example.com';
```

**Poor execution plan (without indexes)**:

```
Hash Join
  -> Seq Scan on users (filter: email = 'alice@example.com')
       Rows: 1,000,000 scanned, 1 returned (500ms)
  -> Seq Scan on posts
       Rows: 10,000,000 scanned (2000ms)
       Build hash table (500ms)
Total: ~3000ms
```

**Optimal execution plan (with indexes)**:

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_posts_user_id ON posts(user_id);
```

```
Nested Loop Join
  -> Index Scan on users using idx_users_email
       Rows: 1 returned (0.5ms)
  -> Index Scan on posts using idx_posts_user_id
       Rows: 50 returned (2ms)
Total: ~2.5ms
```

**Performance improvement**: 1200x faster (3000ms → 2.5ms)

## 5️⃣ How engineers actually use this in production

### Real-World Practices

#### 1. Continuous Query Monitoring

**Uber's approach**: They monitor slow queries in real-time:
- Log all queries taking > 100ms
- Alert on queries taking > 1 second
- Weekly analysis of top 10 slowest queries
- Automated index recommendations based on query patterns

**Implementation**: Use query logging and APM tools (Datadog, New Relic, etc.)

#### 2. Index Strategy at Scale

**Netflix's database optimization**:
- Regular index audits (monthly reviews)
- Remove unused indexes (they slow down writes)
- Multi-column indexes for common query patterns
- Partial indexes for filtered queries

**Example**: Instead of indexing all users, index only active users:
```sql
CREATE INDEX idx_active_users_email 
ON users(email) 
WHERE status = 'active';
```

#### 3. Query Review Process

**Facebook/Meta's approach**:
- All queries reviewed before deployment
- EXPLAIN plans must show index usage
- No queries allowed without WHERE clauses on large tables
- Code review checklist includes query optimization

#### 4. Automated Optimization Tools

**Tools used in production**:
- **pg_stat_statements** (PostgreSQL): Tracks query performance
- **pt-query-digest** (MySQL): Analyzes slow query logs
- **Atlas** (schema migrations with optimization suggestions)
- **EverSQL**: AI-powered query optimization

### Production Workflow

**Step 1: Identify Slow Queries**
```sql
-- PostgreSQL: Enable query logging
ALTER DATABASE mydb SET log_min_duration_statement = 1000; -- Log queries > 1s

-- View slow queries
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```

**Step 2: Analyze Query Plan**
```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name;
```

**Step 3: Optimize**
- Add missing indexes
- Rewrite query if needed
- Consider denormalization for read-heavy patterns

**Step 4: Validate**
- Run EXPLAIN ANALYZE again
- Compare execution times
- Load test with production-like data

**Step 5: Deploy and Monitor**
- Deploy index (during low traffic if possible)
- Monitor query performance
- Watch for index bloat over time

## 6️⃣ How to implement or apply it

### Java/Spring Boot Implementation

#### Example 1: Query Optimization with JPA

**Problem**: N+1 query issue with user posts

**Entity classes**:
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String email;
    private String name;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Post> posts;
    
    // Getters and setters
}

@Entity
@Table(name = "posts")
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String title;
    private String content;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    // Getters and setters
}
```

**Bad query (N+1 problem)**:
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
}

// Service code that triggers N+1
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public List<UserDTO> getUsersWithPosts(String email) {
        List<User> users = userRepository.findByEmail(email);
        // Accessing posts triggers N queries (one per user)
        return users.stream()
            .map(user -> new UserDTO(
                user.getName(),
                user.getPosts().size() // Triggers query for each user!
            ))
            .collect(Collectors.toList());
    }
}
```

**Optimized query (JOIN FETCH)**:
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u " +
           "LEFT JOIN FETCH u.posts " +
           "WHERE u.email = :email")
    List<User> findByEmailWithPosts(@Param("email") String email);
    
    // Alternative: Using EntityGraph
    @EntityGraph(attributePaths = {"posts"})
    List<User> findByEmail(String email);
}
```

**Service code (optimized)**:
```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public List<UserDTO> getUsersWithPosts(String email) {
        // Single query with JOIN
        List<User> users = userRepository.findByEmailWithPosts(email);
        return users.stream()
            .map(user -> new UserDTO(
                user.getName(),
                user.getPosts().size() // Already loaded, no extra queries
            ))
            .collect(Collectors.toList());
    }
}
```

#### Example 2: Batch Loading

**Problem**: Loading many entities with relationships

**Batch loading configuration**:
```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
```

**Repository with batch fetch**:
```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    
    @Query("SELECT p FROM Post p " +
           "WHERE p.user.id IN :userIds")
    List<Post> findByUserIds(@Param("userIds") List<Long> userIds);
    
    @Query(value = "SELECT * FROM posts " +
                   "WHERE user_id IN :userIds",
           nativeQuery = true)
    List<Post> findByUserIdsNative(@Param("userIds") List<Long> userIds);
}
```

#### Example 3: Database Index Management

**Migration with Flyway**:
```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- V2__add_email_index.sql
CREATE INDEX idx_users_email ON users(email);

-- V3__add_composite_index.sql
CREATE INDEX idx_users_email_created ON users(email, created_at);

-- V4__add_partial_index.sql
CREATE INDEX idx_active_users_email 
ON users(email) 
WHERE status = 'active';
```

**Spring Boot configuration**:
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost/mydb");
        config.setUsername("user");
        config.setPassword("password");
        
        // Connection pool optimization
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        
        // Query optimization
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        
        return new HikariDataSource(config);
    }
}
```

#### Example 4: Query Performance Monitoring

**Custom repository interceptor**:
```java
@Component
public class QueryPerformanceInterceptor implements Interceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(QueryPerformanceInterceptor.class);
    private static final long SLOW_QUERY_THRESHOLD_MS = 1000;
    
    @Override
    public boolean onLoad(Object entity, Serializable id, Object[] state, 
                         String[] propertyNames, Type[] types) {
        long startTime = System.currentTimeMillis();
        try {
            return false;
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            if (duration > SLOW_QUERY_THRESHOLD_MS) {
                logger.warn("Slow query detected: {} ms for entity {} with id {}", 
                           duration, entity.getClass().getSimpleName(), id);
            }
        }
    }
}
```

**Using Micrometer for metrics**:
```java
@Service
public class UserService {
    
    private final MeterRegistry meterRegistry;
    private final UserRepository userRepository;
    
    public UserService(MeterRegistry meterRegistry, UserRepository userRepository) {
        this.meterRegistry = meterRegistry;
        this.userRepository = userRepository;
    }
    
    public List<User> findUsers(String email) {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            return userRepository.findByEmail(email);
        } finally {
            sample.stop(Timer.builder("db.query.duration")
                .tag("query", "findByEmail")
                .tag("table", "users")
                .register(meterRegistry));
        }
    }
}
```

#### Example 5: Native Query Optimization

**When JPA isn't enough, use native queries**:
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query(value = 
        "SELECT u.id, u.name, COUNT(p.id) as post_count " +
        "FROM users u " +
        "LEFT JOIN posts p ON u.id = p.user_id " +
        "WHERE u.created_at > :since " +
        "GROUP BY u.id, u.name " +
        "HAVING COUNT(p.id) > :minPosts " +
        "ORDER BY post_count DESC " +
        "LIMIT :limit",
        nativeQuery = true)
    List<Object[]> findActiveUsersWithPostCount(
        @Param("since") LocalDateTime since,
        @Param("minPosts") int minPosts,
        @Param("limit") int limit
    );
}
```

**Index to support this query**:
```sql
CREATE INDEX idx_users_created_at ON users(created_at);
CREATE INDEX idx_posts_user_id_created ON posts(user_id, created_at);
```

## 7️⃣ Tradeoffs, pitfalls, and common mistakes

### Tradeoffs

#### 1. Indexes: Read Speed vs Write Speed
- **Benefit**: Dramatically faster reads
- **Cost**: Slower writes (indexes must be updated)
- **Rule of thumb**: Index if reads >> writes (10:1 ratio or more)

#### 2. Query Complexity vs Readability
- Complex optimized queries are harder to maintain
- Sometimes denormalization is better than complex joins
- Consider materialized views for expensive aggregations

#### 3. Early Optimization vs Premature Optimization
- Don't optimize until you have performance data
- But do design with optimization in mind (proper indexes from start)

### Common Pitfalls

#### Pitfall 1: Over-Indexing
**Problem**: Too many indexes slow down writes
```sql
-- Bad: Indexing every column
CREATE INDEX idx1 ON users(id);
CREATE INDEX idx2 ON users(email);
CREATE INDEX idx3 ON users(name);
CREATE INDEX idx4 ON users(created_at);
CREATE INDEX idx5 ON users(status);
-- ... and 10 more
```

**Solution**: Index only columns used in WHERE, JOIN, ORDER BY clauses

#### Pitfall 2: Index on Low-Selectivity Columns
**Problem**: Indexing boolean or enum columns with few values
```sql
-- Bad: Only 2 values (true/false)
CREATE INDEX idx_users_active ON users(is_active);
```

**Solution**: Use partial indexes instead
```sql
-- Good: Index only active users
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
```

#### Pitfall 3: Missing Composite Indexes
**Problem**: Multiple single-column indexes don't help multi-column queries
```sql
-- Bad: Two separate indexes
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_status ON users(status);

-- Query uses both, but can't use indexes efficiently
SELECT * FROM users WHERE email = 'x' AND status = 'active';
```

**Solution**: Create composite index
```sql
-- Good: Composite index matches query pattern
CREATE INDEX idx_email_status ON users(email, status);
```

#### Pitfall 4: SELECT * on Large Tables
**Problem**: Fetching unnecessary columns
```java
// Bad: Fetches all columns including large TEXT fields
List<User> users = userRepository.findAll();
```

**Solution**: Select only needed columns
```java
// Good: Project only needed fields
@Query("SELECT u.id, u.name, u.email FROM User u")
List<UserSummary> findAllSummaries();
```

#### Pitfall 5: Functions on Indexed Columns
**Problem**: Using functions prevents index usage
```sql
-- Bad: Index can't be used
SELECT * FROM users WHERE UPPER(email) = 'ALICE@EXAMPLE.COM';

-- Bad: Index can't be used
SELECT * FROM posts WHERE EXTRACT(YEAR FROM created_at) = 2024;
```

**Solution**: Restructure query or use expression index
```sql
-- Good: Index can be used
SELECT * FROM users WHERE email = 'alice@example.com';

-- Good: Expression index
CREATE INDEX idx_posts_year ON posts(EXTRACT(YEAR FROM created_at));
SELECT * FROM posts WHERE EXTRACT(YEAR FROM created_at) = 2024;
```

#### Pitfall 6: OR Conditions Breaking Indexes
**Problem**: OR conditions often prevent index usage
```sql
-- Bad: May not use index efficiently
SELECT * FROM users WHERE email = 'x' OR name = 'y';
```

**Solution**: Use UNION or restructure
```sql
-- Good: Each query can use index
SELECT * FROM users WHERE email = 'x'
UNION
SELECT * FROM users WHERE name = 'y';
```

### Performance Gotchas

#### Gotcha 1: Statistics Stale
**Problem**: Query planner uses outdated statistics
**Solution**: Run `ANALYZE` regularly (PostgreSQL) or `OPTIMIZE TABLE` (MySQL)

#### Gotcha 2: Parameter Sniffing
**Problem**: Query plan optimized for first parameter value, bad for others
```java
// First call with rare value creates plan optimized for that
findByStatus("rare_status"); // Plan optimized for rare status
findByStatus("common_status"); // Uses wrong plan!
```

**Solution**: Use query hints or separate methods for different patterns

#### Gotcha 3: Implicit Type Conversions
**Problem**: Type mismatch prevents index usage
```sql
-- Bad: String compared to integer (no index usage)
SELECT * FROM users WHERE id = '123';
```

**Solution**: Match types
```sql
-- Good: Integer compared to integer (index used)
SELECT * FROM users WHERE id = 123;
```

## 8️⃣ When NOT to use this

### When Optimization Isn't Needed

1. **Prototypes and MVPs**: Don't optimize until you have real data and usage patterns

2. **Small datasets**: Tables with < 10,000 rows don't need complex optimization

3. **One-time queries**: Ad-hoc reports don't need optimization (but do need to complete!)

4. **Write-only operations**: If data is only inserted and never queried, skip indexes

### Anti-Patterns

1. **Optimizing without metrics**: Don't guess what's slow, measure first

2. **Premature denormalization**: Normalized design first, denormalize only when needed

3. **Micro-optimizations**: Optimizing a query from 10ms to 5ms while ignoring 5-second queries

## 9️⃣ Comparison with Alternatives

### Query Optimization vs Caching

**Query Optimization**: Makes the query itself fast
- Pros: Always up-to-date data, works for all queries
- Cons: Limited by database performance

**Caching**: Stores query results in memory
- Pros: Extremely fast (sub-millisecond)
- Cons: Stale data, memory usage, cache invalidation complexity

**Best practice**: Optimize queries first, then cache hot queries

### Query Optimization vs Read Replicas

**Query Optimization**: Makes queries faster on same database
- Pros: No infrastructure changes, always consistent
- Cons: Still limited by single database capacity

**Read Replicas**: Distribute read load across multiple databases
- Pros: Horizontal scaling, isolation of read load
- Cons: Replication lag, more infrastructure, eventual consistency

**Best practice**: Optimize queries first, use replicas when single DB is maxed out

### Query Optimization vs Materialized Views

**Query Optimization**: Optimizes the query execution
- Pros: Always current data, flexible queries
- Cons: Still executes query every time

**Materialized Views**: Pre-computed query results
- Pros: Instant results for complex aggregations
- Cons: Stale data, storage overhead, refresh complexity

**Best practice**: Use materialized views for expensive aggregations that don't need real-time data

## 🔟 Interview follow-up questions WITH answers

### Question 1: How do you identify slow queries in production?

**Answer**:
1. **Enable query logging**: Configure database to log queries above threshold (e.g., > 1 second)
2. **Use APM tools**: Datadog, New Relic track database query performance automatically
3. **Database-specific tools**:
   - PostgreSQL: `pg_stat_statements` extension
   - MySQL: Slow query log and `pt-query-digest`
   - SQL Server: Extended events
4. **Application-level monitoring**: Add timing around database calls, use Micrometer
5. **User reports**: Monitor p95/p99 latencies, alert on spikes

**Follow-up**: "What if you can't enable query logging?"
- Use connection pool monitoring (connection hold times)
- APM tools with database instrumentation
- Database activity monitoring tools
- Analyze application logs for timeout patterns

### Question 2: A query that was fast yesterday is slow today. What could cause this?

**Answer**:
1. **Data growth**: Table grew significantly, indexes no longer efficient
2. **Statistics stale**: Query planner using outdated statistics, run `ANALYZE`
3. **Index bloat**: Indexes fragmented, rebuild indexes
4. **Lock contention**: Other queries locking tables, check for long-running transactions
5. **Missing index**: Index was dropped or became invalid
6. **Query plan changed**: Planner chose different execution plan (parameter sniffing)
7. **Database load**: Higher overall load affecting query performance
8. **Network issues**: Database server network problems

**Debugging steps**:
1. Compare EXPLAIN plans from yesterday vs today
2. Check table/index sizes
3. Review database statistics
4. Check for locks and long-running queries
5. Review database server metrics (CPU, memory, disk I/O)

### Question 3: How do you optimize a query with multiple JOINs?

**Answer**:
1. **Analyze execution plan**: Run EXPLAIN to see join order and methods
2. **Add indexes**: Index join columns (foreign keys) and WHERE clause columns
3. **Join order matters**: Database optimizer usually handles this, but can hint if needed
4. **Filter early**: Push WHERE conditions as early as possible
5. **Consider denormalization**: If joins are expensive and frequent, denormalize
6. **Use covering indexes**: Index includes all columns needed (avoids table lookups)
7. **Materialized views**: For expensive joins that don't need real-time data

**Example**:
```sql
-- Query
SELECT u.name, p.title, c.content
FROM users u
JOIN posts p ON u.id = p.user_id
JOIN comments c ON p.id = c.post_id
WHERE u.status = 'active' AND p.created_at > '2024-01-01';

-- Optimization
CREATE INDEX idx_users_status ON users(status) WHERE status = 'active';
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at);
CREATE INDEX idx_comments_post ON comments(post_id);
```

**Follow-up**: "What if the query planner chooses wrong join order?"
- Use query hints (database-specific)
- Restructure query to force order
- Update statistics so planner makes better decisions
- Consider materialized view as last resort

### Question 4: When would you denormalize a database instead of optimizing queries?

**Answer**:
Denormalize when:
1. **Read-to-write ratio very high**: 100:1 or more reads vs writes
2. **Query patterns fixed**: Same queries repeated millions of times
3. **Joins are expensive**: Even with optimization, joins are too slow
4. **Real-time not required**: Can tolerate slightly stale denormalized data
5. **Storage cost acceptable**: Denormalization increases storage

**Example**: User profile page shows user info + post count + follower count
- **Normalized**: 3 queries or 1 query with 2 JOINs
- **Denormalized**: Add `post_count` and `follower_count` columns to users table, update on writes
- **Tradeoff**: Writes become more expensive, reads become instant

**Anti-pattern**: Don't denormalize "just in case" - measure first, optimize queries, then consider denormalization if needed.

### Question 5: How do you handle query optimization in a microservices architecture?

**Answer**:
1. **Service-level optimization**: Each service optimizes its own database queries
2. **API aggregation**: Service aggregates data from multiple sources to reduce client queries
3. **CQRS pattern**: Separate read and write models, optimize read model specifically
4. **Event sourcing**: Build read-optimized views from events
5. **Data locality**: Cache frequently accessed foreign data locally
6. **GraphQL/BFF**: Backend-for-frontend aggregates multiple service calls

**Example**: Order service needs user info
- **Naive**: Query user service for each order (N+1)
- **Optimized**: Batch fetch users, cache user data locally, or denormalize user name in orders table

**Challenge**: Can't use database JOINs across services
**Solution**: Application-level joins, caching, or event-driven denormalization

### Question 6 (L5/L6): How would you design a system to automatically optimize queries?

**Answer**:
**System components**:
1. **Query collector**: Capture all queries with execution plans and timings
2. **Analyzer**: Identify slow queries, missing indexes, suboptimal plans
3. **Recommendation engine**: Suggest index creation, query rewrites, statistics updates
4. **Testing framework**: Validate recommendations on staging before production
5. **Deployment pipeline**: Automatically create indexes, update statistics

**Architecture**:
- **Real-time monitoring**: Stream query metrics to analysis system
- **Machine learning**: Learn query patterns, predict which indexes help
- **A/B testing**: Test optimization recommendations safely
- **Rollback mechanism**: Automatically revert if performance degrades

**Considerations**:
- Safety: Never automatically drop indexes or change data
- Validation: Test all recommendations thoroughly
- Monitoring: Track optimization impact over time
- Human review: Critical changes require approval

## 1️⃣1️⃣ One clean mental summary

Database query optimization makes queries fast by using indexes (like a library card catalog), choosing efficient execution plans (like a smart librarian), and writing queries that let the database use its tools effectively. The process involves identifying slow queries, analyzing execution plans, adding appropriate indexes, and monitoring results. Key tradeoffs include read speed vs write speed (indexes slow writes), query complexity vs maintainability, and when to optimize vs when to use alternatives like caching or denormalization. Always measure first, optimize systematically, and validate improvements.

