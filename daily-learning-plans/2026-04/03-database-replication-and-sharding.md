# Day 27 — Database Replication & Sharding
**Date:** 2026-04-03
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Database Replication & Sharding — Scaling the Last Single Point of Failure**

You've scaled your application tier horizontally. But everything still writes to one database — the classic bottleneck. Database replication adds read capacity and fault tolerance. Sharding adds write capacity and allows virtually unlimited scale. Together they turn the database from a single point of failure into a distributed data layer. This is Day 17 (indexing) applied at architectural scale.

---

## 2. Core Concepts

### 2.1 Why One Database Isn't Enough

```
Single database:
  Max writes: ~10,000-50,000 TPS (varies by hardware)
  Disk I/O: shared among all reads and writes
  Single point of failure: database down = everything down
  Geographic latency: EU users reading from US database = 100-200ms round trip

Solutions:
  Replication → More read capacity + fault tolerance
  Sharding    → More write capacity + horizontal scale
  Both        → Production-grade data tier
```

---

## 3. Replication

### 3.1 Primary-Replica (Master-Slave) Replication

```
         ┌──────────┐
         │ Primary  │  ← All WRITES go here
         └────┬─────┘
              │ Replication stream (binary log)
    ┌─────────┼──────────┐
    ▼         ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│Replica1│ │Replica2│ │Replica3│  ← All READS distributed here
└────────┘ └────────┘ └────────┘
```

**Replication lag** — replicas are eventually consistent with the primary. A write to the primary may not be visible on replicas for milliseconds to seconds.

#### Spring Boot Read/Write Splitting

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.write")
    public DataSource writeDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.read")
    public DataSource readDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public DataSource routingDataSource(
            @Qualifier("writeDataSource") DataSource write,
            @Qualifier("readDataSource") DataSource read) {
        AbstractRoutingDataSource routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                    ? "read" : "write";
            }
        };
        routing.setTargetDataSources(Map.of("write", write, "read", read));
        routing.setDefaultTargetDataSource(write);
        return routing;
    }
}
```

```java
@Service
public class UserService {

    @Transactional(readOnly = true)  // → routes to replica
    public User findById(Long id) { ... }

    @Transactional  // → routes to primary (readOnly = false by default)
    public User createUser(CreateUserRequest req) { ... }
}
```

### 3.2 Replication Modes

| Mode | Durability | Latency | Use Case |
|------|-----------|---------|---------|
| **Async** (default) | Replica may lag | Low | General purpose — replica lags by ms-seconds |
| **Semi-sync** | At least 1 replica ACKs | Medium | Balance durability and speed |
| **Sync** | All replicas ACK before commit | High | Financial — zero data loss |

### 3.3 MySQL GTID Replication Setup (production snippet)

```sql
-- primary my.cnf
[mysqld]
server-id = 1
log_bin = mysql-bin
gtid_mode = ON
enforce_gtid_consistency = ON
binlog_format = ROW

-- replica my.cnf
[mysqld]
server-id = 2
gtid_mode = ON
enforce_gtid_consistency = ON
read_only = ON

-- Bootstrap replica
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='primary-host',
  SOURCE_USER='replication_user',
  SOURCE_PASSWORD='secret',
  SOURCE_AUTO_POSITION=1;
START REPLICA;
SHOW REPLICA STATUS\G
```

---

## 4. Sharding

### 4.1 What Is Sharding?

```
Sharding = horizontal partitioning at the database level
Each shard holds a subset of data; all shards together hold everything.

Without sharding:
  1 MySQL instance: all 500M user rows
  Write throughput: bounded by single disk/CPU

With sharding (4 shards by userId):
  Shard 0: userId % 4 = 0 → 125M rows
  Shard 1: userId % 4 = 1 → 125M rows
  Shard 2: userId % 4 = 2 → 125M rows
  Shard 3: userId % 4 = 3 → 125M rows

Write throughput: 4× (each shard gets 1/4 the writes)
```

### 4.2 Sharding Strategies

#### Hash Sharding
```
shard = hash(userId) % numShards
+ Uniform distribution
- Cross-shard range queries are expensive
```

#### Range Sharding
```
Shard 0: userId 0–10M
Shard 1: userId 10M–20M
+ Range queries within a shard are fast
- Hotspots: new users all go to highest shard
```

#### Geographic Sharding
```
Shard EU: all European users (GDPR compliance)
Shard US: all US users
Shard APAC: all APAC users
+ Data locality, low latency, compliance
- Uneven load if regions have different sizes
```

#### Directory-Based Sharding
```
shard-lookup-service: userId → shardId
+ Flexible: can move data between shards
- Lookup service becomes a dependency
```

### 4.3 Sharding in Application Code (ShardingSphere)

```xml
<dependency>
  <groupId>org.apache.shardingsphere</groupId>
  <artifactId>shardingsphere-jdbc</artifactId>
  <version>5.4.1</version>
</dependency>
```

```yaml
# ShardingSphere config
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:sharding.yaml

# sharding.yaml
dataSources:
  shard0:
    url: jdbc:mysql://shard0-host:3306/orders
  shard1:
    url: jdbc:mysql://shard1-host:3306/orders

rules:
  - !SHARDING
    tables:
      orders:
        actualDataNodes: shard${0..1}.orders
        tableStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: orders-inline
    shardingAlgorithms:
      orders-inline:
        type: INLINE
        props:
          algorithm-expression: shard${user_id % 2}
```

---

## 5. Cross-Shard Challenges

### 5.1 Cross-Shard Joins (Avoid or Denormalise)

```sql
-- IMPOSSIBLE in sharded DB: user is on shard 0, order on shard 1
SELECT u.name, o.total FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.id = 42;

-- Solution 1: Application-level join
User user = userShard.findById(42);
List<Order> orders = orderShard.findByUserId(42);
// Join in application code

-- Solution 2: Denormalise — store user_name in orders table
SELECT user_name, total FROM orders WHERE user_id = 42;
-- All data needed is in one shard
```

### 5.2 Distributed Transactions (Avoid Where Possible)

```
Cross-shard write problem:
  Transfer $100 from user on shard 0 to user on shard 1

Option 1: XA distributed transactions (expensive, avoid)
Option 2: Saga pattern (Day 21) — compensating transactions
Option 3: Redesign — same-shard affinity (put related data on same shard)
```

### 5.3 Global IDs Across Shards

```java
// Twitter Snowflake-style ID: globally unique, time-ordered, shard-aware
// 41-bit timestamp | 10-bit shard ID | 12-bit sequence

public class SnowflakeIdGenerator {
    private final long shardId;
    private long lastTimestamp = -1;
    private long sequence = 0;

    public synchronized long nextId() {
        long now = System.currentTimeMillis();
        if (now == lastTimestamp) {
            sequence = (sequence + 1) & 0xFFF;  // 12-bit max
        } else {
            sequence = 0;
            lastTimestamp = now;
        }
        return (now << 22) | (shardId << 12) | sequence;
    }
}
```

---

## 6. Hands-On Exercise

1. Set up MySQL primary-replica replication locally with Docker Compose.
2. Implement the `AbstractRoutingDataSource` read/write split.
3. Write a test that does a write, then an immediate read — observe replication lag.
4. Configure ShardingSphere with 2 in-memory H2 shards, shard by `user_id % 2`.
5. Insert 10 users and verify 5 land on each shard.

---

## 7. Key Takeaways

1. **Replication adds read capacity and fault tolerance** — replicas handle reads, primary handles writes.
2. **Replication lag is real** — don't read your own writes from replicas without stale-read tolerance.
3. **Sharding adds write capacity** — shard only when you've exhausted all other options (indexing, caching, read replicas).
4. **Choose shard key carefully** — a bad shard key (like date) creates hotspots that defeat the purpose.
5. **Cross-shard joins and transactions are expensive** — design data access patterns first, then choose shard key.

---

## 8. Connection to Previous Days

Day 17 (Database Indexing) was about queries within one database. Today you scale beyond one database. Day 23 (Consistent Hashing) is the routing mechanism for sharding. The Saga pattern from Day 21 (Distributed Systems) handles cross-shard transactions without XA.

---

## 9. Resources

- **[High Performance MySQL, 4th Edition](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)** — Replication deep dive.
- **[Vitess](https://vitess.io/)** — YouTube/Google's MySQL sharding middleware (production-grade).
- **[Apache ShardingSphere](https://shardingsphere.apache.org/)** — Java ecosystem sharding.

---

*Day 27. The database is not the ceiling — it's just another layer you can scale horizontally.*
