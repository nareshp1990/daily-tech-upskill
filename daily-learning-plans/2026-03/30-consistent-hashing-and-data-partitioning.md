# Day 23 — Consistent Hashing & Data Partitioning
**Date:** 2026-03-30
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Consistent Hashing & Data Partitioning — Distributing Data Without Catastrophic Reshuffling**

You know from Day 16 that horizontal scaling means adding more machines. But how do you decide which machine holds which data? Naive modulo hashing (`key % N`) means that adding even one node invalidates almost every existing mapping — catastrophic for a cache or distributed database. Consistent hashing solves this: adding or removing a node only remaps `1/N` of keys, not all of them. Today you master the theory and the Java implementation.

---

## 2. Core Concepts

### 2.1 The Problem with Simple Hash Partitioning

```
3 nodes: key % 3
key "user:42" → hash 4200 % 3 = 0 → Node 0

Add a 4th node: key % 4
key "user:42" → hash 4200 % 4 = 0 → Node 0  (lucky)
key "user:99" → hash 9900 % 3 = 0, % 4 = 3   (moved!)

→ ~75% of all keys get remapped when adding 1 node
→ Cache invalidation, data movement cost = enormous
```

### 2.2 Consistent Hashing — The Ring

```
                  0
              ┌───────┐
         315  │       │  45
              │  Hash │
         270  │  Ring │  90
              │       │
         225  │       │  135
              └───────┘
                  180

Place nodes on the ring by hashing their name/ID:
  Node A → hash("nodeA") % 360 = 45°
  Node B → hash("nodeB") % 360 = 135°
  Node C → hash("nodeC") % 360 = 225°

Place keys on the ring by hashing the key:
  key "user:42" → hash("user:42") % 360 = 80°

Rule: A key is served by the first node clockwise from its position.
  "user:42" at 80° → first clockwise node = Node B at 135° ✅

Add Node D at 100°:
  Only keys between 45° and 100° move from Node B → Node D
  All other keys stay where they are ✅
```

### 2.3 Virtual Nodes (VNodes) — Solving Uneven Distribution

```
Without VNodes: 3 nodes → each gets ~33% of keyspace
But hash collisions create hot spots.

With VNodes: Each physical node gets K virtual positions on the ring.
  Node A → positions at 45°, 180°, 300° (K=3)
  Node B → positions at 90°, 210°, 330°
  Node C → positions at 135°, 270°,  15°

Benefits:
1. More uniform distribution — statistical averaging
2. Graceful capacity weighting — stronger nodes get more VNodes
3. Faster rebalancing — more fine-grained movement
```

### 2.4 Implementation in Java

```java
import java.security.MessageDigest;
import java.util.SortedMap;
import java.util.TreeMap;

public class ConsistentHashRing<T> {

    private final int virtualNodes;
    private final SortedMap<Long, T> ring = new TreeMap<>();

    public ConsistentHashRing(int virtualNodes) {
        this.virtualNodes = virtualNodes;
    }

    public void addNode(T node) {
        for (int i = 0; i < virtualNodes; i++) {
            long hash = hash(node.toString() + "#" + i);
            ring.put(hash, node);
        }
    }

    public void removeNode(T node) {
        for (int i = 0; i < virtualNodes; i++) {
            long hash = hash(node.toString() + "#" + i);
            ring.remove(hash);
        }
    }

    public T getNode(String key) {
        if (ring.isEmpty()) return null;
        long hash = hash(key);
        // Find the first node at or after this hash
        SortedMap<Long, T> tailMap = ring.tailMap(hash);
        Long nodeHash = tailMap.isEmpty() ? ring.firstKey() : tailMap.firstKey();
        return ring.get(nodeHash);
    }

    private long hash(String key) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(key.getBytes());
            // Use first 8 bytes as long
            long hash = 0;
            for (int i = 0; i < 8; i++) {
                hash = (hash << 8) | (digest[i] & 0xFF);
            }
            return hash;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

Usage:
```java
ConsistentHashRing<String> ring = new ConsistentHashRing<>(150); // 150 VNodes
ring.addNode("redis-node-1:6379");
ring.addNode("redis-node-2:6379");
ring.addNode("redis-node-3:6379");

String node = ring.getNode("user:12345");
System.out.println("Key 'user:12345' → " + node);
```

### 2.5 Replication Factor

```
Ring with replication factor R=3:
Each key is stored on the next R nodes clockwise.

"user:42" at 80° → stored on Node B (135°), Node C (225°), Node D (300°)

Benefits: fault tolerance — if Node B dies, C and D still have the data.
This is how Cassandra, DynamoDB, and Riak work.
```

---

## 3. Data Partitioning Strategies

### 3.1 Range Partitioning

```
Key ranges assigned to shards:
  Shard 0: user IDs 0–1,000,000
  Shard 1: user IDs 1,000,001–2,000,000
  Shard 2: user IDs 2,000,001–3,000,000

Pros: Range queries are efficient (scan one shard)
Cons: Hotspots — new users all go to the last shard
```

### 3.2 Hash Partitioning

```
shard = hash(userId) % numShards

Pros: Uniform distribution
Cons: Range queries span all shards (full scatter-gather)
```

### 3.3 Directory-Based Partitioning

```
Lookup table: userId → shardId
  Centralised service or replicated config

Pros: Flexible resharding without changing hash function
Cons: Single point of failure if lookup service goes down
```

### 3.4 Partitioning in Kafka

```yaml
spring:
  kafka:
    producer:
      # Default: hash(key) % numPartitions
      # Custom partitioner for business-aware routing:
      properties:
        partitioner.class: com.example.UserRegionPartitioner
```

```java
public class UserRegionPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int numPartitions = cluster.partitionCountForTopic(topic);
        String userId = (String) key;
        // Route EU users to first half, US users to second half
        return userId.startsWith("EU") ?
            Math.abs(userId.hashCode()) % (numPartitions / 2) :
            (numPartitions / 2) + Math.abs(userId.hashCode()) % (numPartitions / 2);
    }
}
```

---

## 4. Resharding — Moving Data Between Partitions

### Online Resharding Strategy (Avoid Downtime)

```
Phase 1: Dual-write (write to old AND new sharding scheme)
Phase 2: Backfill — migrate existing data from old → new
Phase 3: Verify — compare old and new, resolve conflicts
Phase 4: Cut-over — switch reads to new scheme
Phase 5: Drain — stop writing to old scheme

Timeline:
  Old: ████████████████░░░░░░░░░░░
  New:         ████████████████████
  Dual-write:  ████████
```

---

## 5. Hands-On Exercise

1. Implement the `ConsistentHashRing<T>` from section 2.4.
2. Add 3 nodes and hash 10,000 keys. Measure distribution per node (should be ~3333 each).
3. Add a 4th node — measure how many keys moved (should be ~2500, i.e. 25%).
4. Compare: do the same with `key.hashCode() % N` naive hashing — count keys that moved.
5. Try different VNode counts (1, 10, 150) and chart the distribution variance.

---

## 6. Key Takeaways

1. **Simple modulo hashing remaps ~(N-1)/N keys when a node is added** — catastrophic for large caches.
2. **Consistent hashing remaps ~1/N keys** — proportional and predictable.
3. **Virtual nodes fix uneven distribution** — more VNodes = better balance at cost of memory.
4. **Replication factor determines fault tolerance** — standard is 3 (tolerate 2 failures).
5. **Know when to use range vs hash partitioning** — range for time-series/ordered queries, hash for uniform distribution.

---

## 7. Connection to Previous Days

Caching (Day 18) assumed "cache aside" but didn't explain how to route requests to the right Redis node — consistent hashing is that mechanism. Kafka partitioning (Day 19) uses a similar concept. Database sharding (coming Day 27) builds on these partitioning strategies.

---

## 8. Resources

- **[Consistent Hashing — MIT lecture notes](https://pdos.csail.mit.edu/6.824/papers/consistent-hashing.pdf)** — Original Karger et al. paper.
- **[Amazon Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)** — How DynamoDB uses consistent hashing with VNodes.
- **[Ketama — libmemcached consistent hashing](https://github.com/RJ/ketama)** — Classic production implementation.

---

*Day 23. Where data lives determines how fast you can find it — choose the ring over the remainder.*
