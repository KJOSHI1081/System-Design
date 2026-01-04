 # System Design: Global Distributed Cache  

---

## 1. Problem Statement & Requirements
Design a distributed caching system capable of supporting millions of requests per second (RPS) with sub-millisecond latency.

### Functional Requirements
* **API:** `get(key)` and `put(key, value, ttl)`.
* **Eviction:** LRU (Least Recently Used) is the default.
* **Consistency:** Eventual consistency (standard) or Strong consistency (tunable).

### Non-Functional Requirements (The Staff-Level Bar)
* **Scalability:** Horizontal scaling to 100+ nodes.
* **High Availability:** 99.999% uptime; resilient to shard/node failures.
* **Performance:** Sub-millisecond $P99$ latency.
* **Multi-tenancy:** Isolation between different service consumers.

---

## 2. Core Architecture: The Single Node
Each node manages a segment of the global keyspace.

### Data Structures
* **Hash Map:** Maps `Key` to a pointer in the Doubly Linked List for $O(1)$ lookup.
* **Doubly Linked List:** Maintains access order. New/accessed items move to **Head**; **Tail** is evicted when capacity is reached.



### Staff-Level Optimization: Lock Striping
To avoid CPU contention in multi-threaded environments, we do not lock the entire cache.
* **Strategy:** Partition the local memory into $N$ slots (e.g., 16 segments). Each segment has its own mutex. This allows concurrent access to different keys without blocking the whole system.

---

## 3. Distributed Scaling (The Cluster)

### Data Partitioning via Consistent Hashing
Traditional `hash(key) % N` is insufficient because adding/removing a node causes a massive cache miss storm.
* **Solution:** **Consistent Hashing with Virtual Nodes.**
* **Virtual Nodes:** Maps a single physical node to hundreds of points on the hash ring. This ensures "load balancing" even if some keys are slightly more popular than others.



### High Availability & Replication
* **Shard Replication:** Each shard has a Primary and $N$ Replicas.
* **Failure Detection:** Use a gossip protocol or a centralized coordinator (Zookeeper) to detect node death and trigger a Replica promotion within seconds.

---

## 4. Advanced Staff-Level Considerations

### A. The "Hot Key" Problem
When a single key (e.g., a celebrity's profile) resides on one shard and receives 1M RPS.
* **Mitigation 1 (L1 Cache):** Implement a tiny "in-process" cache on the Application server with a 1-second TTL.
* **Mitigation 2 (Key Salting):** Append a random suffix to hot keys (`viral_video_1`, `viral_video_2`) to spread the load across multiple physical shards.

### B. Cache Stampede (Thundering Herd)
Occurs when a popular key expires and all application nodes hit the DB at once.
* **Mitigation:** **Promise Collapsing (SingleFlight).** The cache node detects multiple requests for the same missing key and only allows one request to the DB. Other requests "subscribe" to the result of the first request.

### C. Near-Real-Time Invalidation
How to keep the cache and DB in sync?
* **CDC (Change Data Capture):** Use a service like Debezium to stream DB transaction logs (Binlog). When a row is updated in the DB, the CDC service sends an invalidation signal to the cache cluster.

---

## 5. Trade-off Summary Table

| Feature | Decision | Trade-off / Risk |
| :--- | :--- | :--- |
| **Eviction** | W-TinyLFU | Higher CPU overhead than LRU, but better hit rate for frequency-based access. |
| **Consistency** | Eventual (Write-around) | Small window of stale data; significantly lower write latency. |
| **Network** | Protobuf over gRPC | More complex than JSON/REST, but reduces payload size and serialization time. |
| **Deployment** | Multi-AZ (Availability Zones) | Higher cross-zone networking costs, but survives data center outages. |

---

## 6. Monitoring & Observability
* **Cache Hit Ratio:** The primary metric of health.
* **P99 Latency:** Tracking tail latency to identify "noisy neighbors" in multi-tenant setups.
* **Eviction Rate:** High eviction rates with low hit rates suggest the cache size is too small for the working set.
