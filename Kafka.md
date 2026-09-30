# Apache Kafka: Deep Interview Notes (Kafka 3.x / KRaft, Spring for Apache Kafka 3.x)

> Audience: Java developer with about 5 years of experience preparing for senior-level interviews.
> Honesty note: none of the commands or code in this file were executed while writing it. Kafka CLI tools and the Kafka client jars are not installed in this workspace. Config keys and APIs are written from documented behavior. Verify defaults against the docs of the exact version you run. Where a default changed between versions, the note says so.

## Table of contents
1. 60-second mental model and analogy
2. Why Kafka is fast
3. Storage internals (topic, partition, segment, index, retention, compaction)
4. Replication (leader, follower, ISR, high watermark, acks matrix, leader epoch)
5. Controller and metadata (ZooKeeper vs KRaft)
6. Producer internals (batching, ordering, idempotence, transactions)
7. Consumer internals (poll loop, timeouts, commits, offset reset)
8. Consumer groups and rebalancing
9. Ordering, partition planning, lag monitoring
10. Failure scenario tables (what is lost, what is duplicated)
11. Spring Kafka: config, producer, consumer, error handling, retry topics
12. Serialization and schema evolution
13. Streams, ksqlDB, Connect, Debezium, outbox pattern end to end
14. Event-driven design, security, multi-DC, capacity and ops
15. Kafka vs RabbitMQ vs SQS/SNS vs Pulsar
16. Testing
17. docker-compose (KRaft) and CLI cheat sheet
18. Production war stories
19. Interview questions (50) with follow-ups and common wrong answers
20. One-page cheat sheet

---

# 1. The 60-second mental model

Kafka is a **distributed, replicated, append-only commit log** that many independent readers can read at their own pace.

```
                      Topic "orders"  (3 partitions, replication factor 3)

 Producer ---key=orderId---> [ P0 ]  0 1 2 3 4 5 ...   <- append at the end only
                             [ P1 ]  0 1 2 3 ...
                             [ P2 ]  0 1 2 3 4 ...
                                     ^         ^
                                     |         +-- consumer group "billing" is at offset 4
                                     +------------ consumer group "email"   is at offset 1
```

Five facts explain about 80 percent of every interview question:

1. A topic is split into **partitions**. A partition is an ordered, immutable sequence of records. Each record has an **offset** (its index in that partition).
2. **Order is guaranteed only inside one partition.** There is no global order across partitions.
3. Records are **not deleted when consumed**. They are deleted by **retention** (time or size) or **compaction** (keep the latest per key).
4. A **consumer group** shares the partitions of a topic. Inside a group, one partition is read by at most one consumer at a time. Different groups are fully independent and each gets every record.
5. Each partition is **replicated**. One replica is the leader (all reads and writes go to it), the others are followers. Durability is a trade-off you choose with `acks`, `min.insync.replicas` and replication factor.

**Analogy.** RabbitMQ is a post office: a letter goes to one recipient and is then gone. Kafka is a **CCTV recording room with numbered tapes**. The camera (producer) only records forward. Each viewer (consumer group) keeps their own bookmark (committed offset). A viewer can rewind, another can be live. Tapes are erased by age or size, not because someone watched them.

**Smart consumer, dumb broker.** The broker does not track per-message acknowledgements. It only stores the log and, for each group, one number per partition (the committed offset). That is why it scales so well.

---

# 2. Why Kafka is fast

| Technique | What it means | Why it helps |
|---|---|---|
| Sequential append-only log | Writes always go to the end of a file; reads are sequential scans | Sequential disk I/O is orders of magnitude faster than random I/O, even on HDD; SSD still benefits from fewer write amplification effects |
| OS page cache | Kafka does not keep its own big in-heap cache; it writes to files and lets the OS cache them in free RAM | Recent data is served from RAM; broker JVM heap can be small (commonly around 6 GB), so little GC pressure; cache survives a broker process restart |
| Zero-copy (`sendfile`) | For plaintext connections, the kernel copies from page cache directly to the socket buffer | Skips copying into user space and back (avoids 2 copies and 2 context switches). **Not applicable with TLS**, because data must pass through user space for encryption |
| Batching | Producer batches records per partition; broker stores the batch as is; consumer fetches many records per request | Amortizes network round trips, syscalls, and per-request overhead |
| Compression | Whole batch is compressed (gzip, snappy, lz4, zstd) by the producer, stored compressed, decompressed by the consumer | Less network and disk; the broker usually does not recompress (it does if the topic-level `compression.type` differs from what producer used) |
| Partitioning | Load spreads across brokers and disks | Horizontal scale for both writes and reads |
| Binary protocol, same format on wire and disk | Record batch format is identical in network and log | The broker can pass bytes through without parse and re-serialize |

Trace of a consumer fetch with zero-copy (plaintext):

```
1. Consumer sends FetchRequest(topic-partition, offset=1000, maxBytes)
2. Broker looks up offset 1000 in .index -> file position in the .log segment
3. Broker calls sendfile(logFileChannel, position, count, socketChannel)
4. Kernel: page cache --DMA--> NIC   (data never enters the JVM heap)
5. If the pages are not in page cache, the OS reads from disk first (this is the slow "cold read" case,
   which is why a consumer that replays old data hurts a broker that also serves real-time traffic)
```

Interview-grade nuance: "Kafka is fast because it writes to disk" is a half-truth. It is fast **because** it writes sequentially and relies on the page cache. It does not `fsync` each write by default; durability comes from **replication**, not from flushing (`log.flush.interval.messages` and `log.flush.interval.ms` exist but are normally left at defaults).

---

# 3. Storage internals

## 3.1 Hierarchy

```
Topic
 +-- Partition 0  (directory: /var/kafka-logs/orders-0/)
 |     +-- 00000000000000000000.log        segment 1: records at offsets 0..4199
 |     +-- 00000000000000000000.index      sparse offset -> byte position
 |     +-- 00000000000000000000.timeindex  sparse timestamp -> offset
 |     +-- 00000000000000004200.log        segment 2: base offset 4200 (file name = first offset)
 |     +-- 00000000000000004200.index
 |     +-- 00000000000000004200.timeindex  <- the ACTIVE segment (only one being appended)
 |     +-- leader-epoch-checkpoint
 +-- Partition 1 ...
```

- A partition is a directory of **segments**. The segment file name is the **base offset** (first offset in that segment).
- Only the newest (**active**) segment is written. Older segments are immutable.
- `.index` is **sparse**: one entry roughly every `log.index.interval.bytes` (default 4096) bytes, not one per record. To find offset N: binary search the index for the largest entry at or below N, then scan the `.log` forward from that position.
- `.timeindex` maps timestamps to offsets so that `offsetsForTimes` (used by "reset to datetime") works.

## 3.2 Segment roll

A new segment is created when the active one:
- reaches `log.segment.bytes` (broker default 1 GiB; topic config `segment.bytes`), or
- is older than `log.roll.hours` / `log.roll.ms` (default 7 days; topic config `segment.ms`), or
- its index is full (`segment.index.bytes`).

## 3.3 Retention (cleanup.policy=delete)

- Deletion is **per whole segment**, never per record. The active segment is never deleted.
- Time: `log.retention.hours` (default 168 = 7 days) or `log.retention.ms`; topic config `retention.ms`. Segment expiry is based on the **largest timestamp in the segment**.
- Size: `log.retention.bytes` / topic `retention.bytes` is **per partition**, default -1 (unlimited). Not per topic.
- A background check runs every `log.retention.check.interval.ms` (default 5 minutes).
- Consequence: data older than retention can remain readable for a while if the segment still contains newer records or is still active. Low-traffic topics with a 1 GiB segment size may keep data far longer than `retention.ms` says. Fix with a smaller `segment.ms`.

## 3.4 Log compaction (cleanup.policy=compact)

Keeps **at least the latest record per key**. Use for "current state per key" topics (changelog, table-like data, `__consumer_offsets` itself, Kafka Streams state store changelogs, config topics).

```
Before compaction (key:value at offset):
 0 A:1   1 B:1   2 A:2   3 C:1   4 B:2   5 A:3   6 C:null(tombstone)   7 B:3
After compaction:
 5 A:3   6 C:null   7 B:3        <- offsets are preserved, with gaps; order preserved
Later, after delete.retention.ms (default 24h), the tombstone C:null is also removed.
```

- **Tombstone** = a record with a key and a **null value**. It marks the key deleted. Consumers must see it before it is removed, hence `delete.retention.ms`.
- The active segment is not compacted. The cleaner works when the dirty ratio exceeds `min.cleanable.dirty.ratio` (default 0.5). `min.compaction.lag.ms` guarantees a minimum time before a record may be compacted.
- Compaction does **not** guarantee that older duplicates vanish immediately. A consumer reading from the beginning may see several versions of a key. It only guarantees the latest is retained.
- Can combine: `cleanup.policy=compact,delete` (compact and also expire old data).
- Records must have keys. A null-key record cannot be written to a compacted topic (rejected).

## 3.5 Offsets vs timestamps

| | Offset | Timestamp |
|---|---|---|
| Scope | per partition, monotonic, assigned by leader | per record; `CreateTime` (producer clock, default) or `LogAppendTime` (broker clock; topic config `message.timestamp.type`) |
| Meaning | position | when (approximately) |
| Use | commit positions, seek | time-based retention, "replay from 10:00" via `offsetsForTimes` |
| Caveat | gaps happen (compaction, transaction markers) | not monotonic with `CreateTime` because producers have different clocks |

Do not compute "number of messages = end offset - start offset" as exact: transaction control markers and compaction create gaps.

## 3.6 Partition log with high watermark (diagram)

```
Leader log of orders-0 (records with offsets):

 offset:   0   1   2   3   4   5   6   7   8
         +---+---+---+---+---+---+---+---+---+
         | r | r | r | r | r | r | r | r | r |
         +---+---+---+---+---+---+---+---+---+
                               ^           ^
                               |           +-- LEO (log end offset) of leader = 9 (next offset to write)
                               +-------------- High Watermark (HW) = 6
                                               (offsets 0..5 are replicated to all ISR)

 Consumers (read_uncommitted default) can read only offsets < HW  ->  0..5
 Records 6..8 exist on the leader but are "not committed" yet; a leader crash may lose them.
 Follower A: LEO = 8    Follower B: LEO = 6   -> HW = min(LEO of ISR members) = 6
```

Key rule: **a consumer never sees a record that is not yet on all in-sync replicas.** That is why a consumer never observes data that disappears after a leader failover (with `acks=all` semantics the producer also is not told "success" before that).

---

# 4. Replication

## 4.1 Concepts

- **Replication factor (RF)**: number of copies of each partition. Typical production value: 3.
- **Leader**: the one replica that serves all produce and (by default) fetch requests. Followers pull from the leader like a consumer (fetch).
- **ISR (in-sync replicas)**: the leader plus followers that are caught up. A follower falls out of the ISR if it has not caught up to the leader's log end within `replica.lag.time.max.ms` (default 30 s since Kafka 2.5).
- **High watermark**: the offset up to which all ISR members have replicated. Consumers read only below it.
- **`min.insync.replicas` (topic/broker config)**: with `acks=all`, the broker rejects a write with `NotEnoughReplicasException` if the current ISR size is below this number. It is a **producer-visible availability vs durability dial**.
- **Leader epoch**: an integer incremented at each leadership change, stored with the log. Followers use it to know where to truncate after a leader change (before leader epochs, they truncated to the high watermark, which could cause divergence or loss in rare cases: KIP-101).
- **Preferred leader**: the first replica in the assignment list. With `auto.leader.rebalance.enable=true` (default) the controller periodically moves leadership back to preferred replicas when imbalance exceeds `leader.imbalance.per.broker.percentage` (default 10), checked every `leader.imbalance.check.interval.seconds` (default 300). Manual: `kafka-leader-election.sh --election-type preferred`.
- **Unclean leader election** (`unclean.leader.election.enable`, default false): allow an **out-of-sync** replica to become leader when all ISR replicas are gone. Availability over consistency: you lose committed records. Keep false for business data.

## 4.2 The acks matrix

Assume RF=3, `min.insync.replicas=2`, and leader L, followers F1 and F2.

| acks | Producer waits for | Loses data when | Latency | Notes |
|---|---|---|---|---|
| `0` | nothing (does not even read a response) | broker down, leader change, network hiccup: send is silently lost; no retries have any effect because no error is seen | lowest | metrics, lossy telemetry only |
| `1` | leader wrote to its local log (not fsync, page cache) | leader crashes **after acking but before followers replicate**; new leader (an ISR follower) never had that record; producer believed success | low | the classic silent-loss setting |
| `all` (or `-1`) | all **current ISR** members have the record | Only if **every** ISR member is lost together (or an unclean election happens). Acked data exists on every ISR member; with `min.insync.replicas=2`, at least 2 copies exist at ack time | highest | the default in clients 3.0+ is `acks=all` (together with idempotence) |

Important subtlety: `acks=all` alone with `min.insync.replicas=1` and ISR shrunk to just the leader means "all" = 1 replica. Then a leader crash loses data. **Durability = acks=all AND min.insync.replicas >= 2 AND unclean election off.**

The classic formula: with RF=N and `min.insync.replicas=M`, you tolerate **N - M** broker failures for **writes** to continue, and **M - 1** simultaneous failures without acked data loss. RF=3, min ISR=2: writes survive 1 broker down; data survives 1 broker loss.

## 4.3 ISR shrink and expand timeline

```
t0   ISR = {L, F1, F2}   HW = LEO = 100      all healthy
t1   F2 has a GC pause / slow disk. F2 stops fetching.
t2   Producer sends offsets 101..110 with acks=all. Leader appends. HW cannot advance beyond 100
     because it waits for F2 (still in ISR).  Producers see latency rising.
t3   t1 + replica.lag.time.max.ms elapsed. Leader shrinks ISR: {L, F1}.  (IsrShrinksPerSec metric)
     Leader informs controller; ISR change is persisted in metadata.
t4   HW advances to 110 (needs only L and F1). Producers unblocked, acked.
t5   ISR size 2 >= min.insync.replicas 2 -> writes continue. If F1 also died: ISR = {L} (size 1 < 2)
     -> acks=all writes fail with NotEnoughReplicasException; reads still work. (consistency over availability)
t6   F2 recovers, fetches from the leader from its own LEO (after truncating divergent tail using leader epoch),
     catches up to the leader LEO.
t7   Leader adds F2 back: ISR = {L, F1, F2}.  (IsrExpandsPerSec)
```

## 4.4 Leader failure trace (acks=1 loss vs acks=all safe)

```
Setup: RF=3, ISR={L,F1,F2}. Producer writes offset 50.
acks=1:
 1. Leader L appends offset 50 to its log, replies ACK.
 2. L crashes before F1/F2 fetch offset 50.
 3. Controller elects F1 (in ISR). F1's log ends at offset 49. New leader epoch = e+1.
 4. Producer already thought 50 was written. It is LOST. Later writes reuse offset 50 with different data.
 5. When old L returns as follower, it sees the higher epoch, truncates offset 50 away, and follows F1.
acks=all (min.insync.replicas=2):
 1. Leader appends offset 50; waits until both F1 and F2 (the ISR) fetched it.
 2. Only then acks. If L crashes before that, producer gets no ack -> retries -> (idempotent) no duplicate.
 3. Any ISR member elected as leader has offset 50 whenever it was acked.
```

## 4.5 Replication factor sizing

- RF=1: dev only. RF=2: survives 1 failure but with `min.insync.replicas=2` loses write availability on any failure; with min ISR 1 risks loss. Rarely a good choice.
- **RF=3, min ISR=2**: the standard. Tolerates 1 broker down with full availability, and survives a rolling restart.
- RF=5 for very critical metadata or cross-AZ needs; costs 5x storage and replication traffic.
- RF cannot exceed the number of brokers. Spread across **racks/AZs** with `broker.rack` so replicas of one partition live in different AZs. Consumers can fetch from the closest replica (follower fetching, `replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector` on brokers and `client.rack` on consumers) to cut cross-AZ cost.
- Storage math: disk = ingest MB/s x retention seconds x RF (plus 20 to 30 percent headroom). Network: each produced byte is written RF times over the cluster network plus once per consumer group read.

---

# 5. Controller and metadata

**Legacy (ZooKeeper mode)**: metadata (brokers, topics, partition assignments, ISR, configs) lives in ZooKeeper. One broker is elected **controller** and pushes metadata changes to other brokers. On controller failover the new controller must reload all metadata from ZooKeeper, which is slow at very high partition counts.

**KRaft (Kafka Raft)**: metadata is stored in an internal Raft-replicated log topic `__cluster_metadata`. A small quorum of **controller** nodes (typically 3, or 5) holds it. One is the **active controller** (Raft leader); others are hot standbys that already have the metadata in memory, so failover is fast. Brokers fetch metadata updates from the controller quorum like consumers.

| | ZooKeeper mode | KRaft |
|---|---|---|
| Metadata store | external ZooKeeper ensemble | internal Raft log `__cluster_metadata` |
| Controller failover | reload state from ZK (slow) | standby has state (fast) |
| Ops | run and secure two systems | one system |
| Partition scalability | limited by ZK and failover | designed for many more partitions per cluster |
| Status | deprecated in 3.5; **removed in Kafka 4.0** | production-ready since 3.3; the only mode in 4.0 |

Node roles (`process.roles`): `broker`, `controller`, or `broker,controller` (combined, fine for dev/small clusters; separate dedicated controllers are recommended for large production). Relevant configs: `process.roles`, `node.id`, `controller.quorum.voters`, `controller.listener.names`. Storage must be formatted once with `kafka-storage.sh format` using a cluster id from `kafka-storage.sh random-uuid`.

Kafka 4.0 also made the new consumer group protocol (KIP-848) generally available (see section 8) and requires Java 17 on brokers, Java 11 for clients.

---

# 6. Producer internals

## 6.1 What happens on `send()`

```
Application thread                                   Sender (I/O) thread
------------------                                   -------------------
send(record)
 1. serialize key, value
 2. partitioner picks partition
 3. append to RecordAccumulator
    (per-partition deque of batches,      ------->   4. drains ready batches (batch full OR linger.ms elapsed)
     memory from buffer.memory)                         grouped by destination broker
    if buffer full: block up to                      5. sends ProduceRequest (up to max.in.flight per broker)
    max.block.ms, then throw                         6. on response: complete Future / callback,
 returns Future<RecordMetadata>                         or retry on retriable error
```

| Config | Default | Meaning |
|---|---|---|
| `batch.size` | 16384 bytes | max bytes per partition batch. It is an upper bound, not a wait target |
| `linger.ms` | 0 (3.x); 5 in 4.0 | how long the sender waits for more records to fill a batch. Trading latency for throughput. Even with 0, records arriving while a request is in flight get batched together |
| `buffer.memory` | 33554432 (32 MB) | total memory for unsent batches. When full, `send()` blocks up to `max.block.ms` (default 60000) then throws `TimeoutException` |
| `compression.type` | none | `gzip`, `snappy`, `lz4`, `zstd`. Compression is per batch, so bigger batches compress better |
| `acks` | `all` (3.0+) | see section 4.2 |
| `retries` | Integer.MAX_VALUE (2.1+) | bounded in practice by `delivery.timeout.ms` |
| `delivery.timeout.ms` | 120000 | upper bound on the time from `send()` returning until success or failure, including batching, retries and in-flight. Must be >= `linger.ms + request.timeout.ms` |
| `request.timeout.ms` | 30000 | how long to wait for one request's response before considering it failed |
| `retry.backoff.ms` | 100 | pause between retries |
| `max.in.flight.requests.per.connection` | 5 | unacknowledged requests per broker connection |
| `enable.idempotence` | true (3.0+, provided compatible settings) | PID and sequence numbers, see 6.4 |
| `max.request.size` | 1048576 | largest request; also caps a single record size on the producer side. Broker/topic side: `message.max.bytes` / `max.message.bytes` |

Rule of thumb for throughput tuning: raise `linger.ms` to 5 to 20 ms, raise `batch.size` to 64 to 256 KB, use `lz4` or `zstd`. Check the metrics `batch-size-avg`, `records-per-request-avg`, `compression-rate-avg`, `buffer-available-bytes`.

## 6.2 Partitioner

Order of decision in the default partitioner (`partitioner.class` default, Kafka 3.x):
1. If the record specifies a partition explicitly, use it.
2. If the record has a key: `Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions`.
3. If the key is null: **sticky** behavior. Fill one batch for one partition, then switch to another (KIP-480, Kafka 2.4). In 3.3+ (KIP-794) the choice is weighted so that slow or unavailable brokers receive fewer records (`partitioner.adaptive.partitioning.enable`, `partitioner.availability.timeout.ms`). Older behavior (pre 2.4) was pure round robin per record, which produced many tiny batches.

Changing the partition count changes `hash % numPartitions`, so **the same key goes to a different partition after you add partitions**. Ordering across the change is not preserved, and compacted topics can end up with two "latest" versions of one key in different partitions.

**Custom partitioner** (implement `org.apache.kafka.clients.producer.Partitioner`, set `partitioner.class`):

```java
public class TenantPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int n = cluster.partitionsForTopic(topic).size();
        String k = String.valueOf(key);
        if (k.startsWith("BIGTENANT-")) {
            // spread one hot tenant over a few partitions using a sub-key: ordering per (tenant, entity) only
            String entity = k.substring(k.indexOf('|') + 1);
            return Utils.toPositive(Utils.murmur2(entity.getBytes(StandardCharsets.UTF_8))) % n;
        }
        return Utils.toPositive(Utils.murmur2(keyBytes)) % n;
    }
    @Override public void close() {}
    @Override public void configure(Map<String, ?> configs) {}
}
```

**Hot partition problem**: one key (a big tenant, a celebrity, `null` mapped to a constant) receives most traffic, so one partition and one consumer become the bottleneck while others are idle. Mitigations: (a) choose a higher-cardinality key (`orderId` rather than `customerId` if ordering per customer is not needed); (b) key salting (`tenant#0..N`) only when ordering across the salted parts is not needed; (c) a custom partitioner as above; (d) more partitions do not help if it is one key. Detect via per-partition bytes-in and per-partition lag.

## 6.3 Ordering and retries

Without idempotence, `retries > 0` with `max.in.flight.requests.per.connection > 1` can reorder: batch 1 fails, batch 2 succeeds, batch 1 is retried and lands after batch 2. Solutions:
- Use `enable.idempotence=true` (default in 3.0+). Then ordering is preserved with up to **5** in-flight requests, because the broker checks sequence numbers per partition.
- Or (legacy) `max.in.flight.requests.per.connection=1` (kills throughput).

## 6.4 Idempotent producer

```
On startup the broker (transaction coordinator path for InitProducerId) assigns:  PID (producer id) + epoch
For each partition the producer numbers batches with a monotonically increasing sequence number:

 Producer(PID=7)  batch seq=0 ----> Leader appends, acks lost on the network
 Producer(PID=7)  batch seq=0 ----> RETRY. Broker sees (PID=7, partition, seq=0) already written
                                    -> does NOT append again, returns success (duplicate suppressed)
 Producer(PID=7)  batch seq=2 (gap, seq=1 missing) -> OutOfOrderSequenceException (broker rejects)
```

- The broker remembers the last **5** batches of sequences per PID per partition (hence the max of 5 in-flight).
- Scope: **one producer instance, one session, one partition**. If the producer process restarts, it gets a **new PID** and the broker cannot recognise re-sent data as a duplicate. The application-level duplicate (send called twice by your code, for example after a crash and replay from the DB) is not detected.
- Requirements: `acks=all`, `retries > 0`, `max.in.flight <= 5`. In 3.0+ these are the defaults; if you explicitly set an incompatible value (for example `acks=1`) with idempotence explicitly on, the client throws a config exception.

## 6.5 Transactions (exactly-once inside Kafka)

Setting `transactional.id` gives: (1) idempotence across restarts (the id maps to a stable PID with an **epoch**; a new instance with the same id bumps the epoch and **fences** the old "zombie" with `ProducerFencedException`), and (2) atomic multi-partition, multi-topic writes plus offset commits.

```java
Properties p = new Properties();
p.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
p.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
p.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
p.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-processor-1"); // unique per instance, stable across restarts

KafkaProducer<String,String> producer = new KafkaProducer<>(p);
KafkaConsumer<String,String> consumer = /* enable.auto.commit=false, isolation.level=read_committed */ null;

producer.initTransactions();                       // once: registers with transaction coordinator, fences zombies
while (true) {
    ConsumerRecords<String,String> recs = consumer.poll(Duration.ofMillis(500));
    if (recs.isEmpty()) continue;
    producer.beginTransaction();
    try {
        Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
        for (ConsumerRecord<String,String> r : recs) {
            producer.send(new ProducerRecord<>("orders-enriched", r.key(), enrich(r.value())));
            offsets.put(new TopicPartition(r.topic(), r.partition()),
                        new OffsetAndMetadata(r.offset() + 1));   // NEXT offset to read
        }
        producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata()); // offsets committed atomically with output
        producer.commitTransaction();
    } catch (ProducerFencedException | OutOfOrderSequenceException | AuthorizationException e) {
        producer.close();  // fatal: cannot recover, must recreate the producer
        throw e;
    } catch (KafkaException e) {
        producer.abortTransaction();  // then rewind consumer to last committed offsets and retry
    }
}
```

How it works:
1. `initTransactions` contacts the **transaction coordinator** (a broker that leads a partition of the internal `__transaction_state` topic chosen by hash of `transactional.id`).
2. Each partition written in the transaction is registered with the coordinator; data is appended to the partition logs **immediately** (not buffered on the broker).
3. `commitTransaction` (two-phase): the coordinator writes PREPARE_COMMIT to `__transaction_state`, then writes a **COMMIT control marker** into every involved partition (it takes one offset each), then marks the transaction complete. Abort writes ABORT markers.
4. Consumers with `isolation.level=read_committed` only return records of committed transactions. They read up to the **LSO (last stable offset)**, which is the offset of the first still-open transaction. One long or hung transaction **blocks all read_committed consumers** on that partition at the LSO. `read_uncommitted` (default) sees everything including aborted data.
5. `transaction.timeout.ms` (producer, default 60000) makes the coordinator abort a transaction that does not finish in time; it must be <= broker `transaction.max.timeout.ms`.

**What "exactly-once" means**: read from Kafka -> process -> write to Kafka, with offset commits, is atomic. It says nothing about side effects outside Kafka. If your consumer calls a payment API or inserts into a DB in the middle, that action can run twice. Pitfalls:
- **DB write in the middle**: DB commit and Kafka transaction are two separate commits; no distributed transaction between them. Use idempotent writes (unique key / upsert) or the outbox pattern.
- Consumers of the output topic must set `read_committed`, otherwise they can see aborted records.
- Transaction cost: latency of the commit round trips; do not use one transaction per record at high rate. Batch per poll.
- Use a **stable, unique `transactional.id` per producer instance** (Spring: `transaction-id-prefix` plus a suffix per instance). Two live instances with the same id fence each other. Since Kafka 2.5 (KIP-447) the producer, with `consumer.groupMetadata()`, no longer needs one transactional producer per input partition.
- Requires broker settings `transaction.state.log.replication.factor` (default 3) and `transaction.state.log.min.isr` (default 2), so a single-broker dev cluster must lower them.
- Kafka Streams enables it with `processing.guarantee=exactly_once_v2`.

---

# 7. Consumer internals

## 7.1 Poll loop

```java
Properties c = new Properties();
c.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
c.put(ConsumerConfig.GROUP_ID_CONFIG, "billing");
c.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
c.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
c.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
c.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

try (KafkaConsumer<String,String> consumer = new KafkaConsumer<>(c)) {
    consumer.subscribe(List.of("orders"));
    while (running) {
        ConsumerRecords<String,String> records = consumer.poll(Duration.ofMillis(500));
        for (ConsumerRecord<String,String> r : records) {
            process(r);                                    // must finish quickly (see max.poll.interval.ms)
        }
        consumer.commitSync();                             // AFTER processing -> at-least-once
    }
}
```

`KafkaConsumer` is **not thread-safe** (only `wakeup()` may be called from another thread). One consumer per thread.

What `poll()` really does: joins the group and handles rebalances (callbacks run inside poll), sends heartbeats via a **background heartbeat thread**, fetches records (pre-fetched into memory), returns up to `max.poll.records` from the buffer, and auto-commits if enabled.

## 7.2 Fetch tuning

| Config | Default | Effect |
|---|---|---|
| `fetch.min.bytes` | 1 | broker holds the fetch response until this many bytes are available... |
| `fetch.max.wait.ms` | 500 | ...or until this time passes. Raise `fetch.min.bytes` for throughput, at the cost of latency |
| `fetch.max.bytes` | 52428800 (50 MB) | max per fetch response (soft limit) |
| `max.partition.fetch.bytes` | 1048576 (1 MB) | per partition per fetch (soft limit; the first batch is always returned) |
| `max.poll.records` | 500 | max records returned from a single `poll()`; it does **not** change how much is fetched from the network |

## 7.3 The three timeouts (a favorite interview topic)

| Config | Default | What it detects |
|---|---|---|
| `heartbeat.interval.ms` | 3000 | how often the background thread pings the group coordinator. Should be <= 1/3 of session timeout |
| `session.timeout.ms` | 45000 (3.0+; earlier 10000) | **consumer process dead or unreachable**: no heartbeat for this long -> coordinator removes member -> rebalance |
| `max.poll.interval.ms` | 300000 (5 min) | **consumer alive but stuck / too slow**: no `poll()` call for this long -> the client itself leaves the group -> rebalance |

Because heartbeats run in a separate thread, a consumer stuck in a slow `process()` **keeps heartbeating** (so `session.timeout.ms` never fires), but stops calling `poll()`. After `max.poll.interval.ms` it is evicted.

**Classic incident: "consumer kicked out of the group because processing is slow."**

```
1. poll() returns 500 records; each takes 1 s to process (calls a slow downstream API) -> 500 s > 300 s.
2. No poll() for 300 s -> the client marks itself failed and leaves the group (log: "consumer poll timeout has expired... time between subsequent calls to poll() was longer than the configured max.poll.interval.ms").
3. Rebalance: its partitions go to another consumer, which starts from the last COMMITTED offset (the batch was not committed).
4. The slow consumer finishes and tries to commit -> CommitFailedException (or in Spring, the container logs a rebalance / commit failure).
5. The new owner re-processes the same records -> duplicates -> also slow -> it gets evicted too -> endless rebalance loop, lag grows, throughput ~0.
Fixes: lower max.poll.records (e.g. 10-50) so one batch finishes well within the interval; raise max.poll.interval.ms
       (last resort); make processing faster or async with pause()/resume() pattern; move slow work to a queue; idempotent consumers.
```

## 7.4 Offsets and commit strategies

Committed offsets live in the internal compacted topic **`__consumer_offsets`** (default 50 partitions, RF 3 in production). Key = (group, topic, partition), value = offset. The **group coordinator** for a group is the leader of partition `hash(groupId) % 50` of that topic.

The committed offset means **"the next record I will read"**, that is, last processed offset + 1.

| Strategy | How | Semantics | Downside |
|---|---|---|---|
| Auto commit | `enable.auto.commit=true` (default true in the raw client), `auto.commit.interval.ms=5000`; commit happens inside `poll()` for records returned by the previous poll | can be at-least-once, but you can lose messages if you process asynchronously (offset committed before background work is done) and can reprocess up to 5 s worth after crash | no control |
| Sync per batch | `commitSync()` after processing the batch | at-least-once; blocks and retries | throughput cost; on crash mid-batch, the whole batch is redone |
| Async | `commitAsync(callback)` | at-least-once; no retry (a retry could commit an older offset after a newer one) | a failure may go unnoticed; use `commitSync()` on shutdown/rebalance |
| Manual per record | `commitSync(Map.of(tp, new OffsetAndMetadata(offset+1)))` after each record | smallest replay window | slowest |
| Commit **before** processing | | **at-most-once** (a crash between commit and processing loses the record) | data loss |
| Commit **after** processing | | **at-least-once** (crash between processing and commit reprocesses) | duplicates; make the consumer idempotent |

**`auto.offset.reset`** applies **only when there is no committed offset** for the group and partition (a new group, or the committed offset expired or was deleted, or the offset is out of range). Values: `earliest` (start of log), `latest` (default; only new records), `none` (throw exception). It does **not** apply when a valid committed offset exists. Offsets of an idle group expire after `offsets.retention.minutes` (default 7 days, since 2.0) once the group has no active members. A classic bug: a consumer down for 8 days restarts with `latest` and silently skips a week of data.

**Seek / replay / reset:**
- In code: `consumer.seek(tp, offset)`, `seekToBeginning`, `seekToEnd`, `offsetsForTimes` (timestamp lookup via `.timeindex`).
- CLI (group must be **inactive**, that is, all consumers stopped): `kafka-consumer-groups.sh --bootstrap-server ... --group billing --topic orders --reset-offsets --to-earliest --execute` (without `--execute` it is a dry run; also `--to-datetime`, `--to-offset`, `--shift-by`, `--by-duration`, `--to-latest`, `--to-current`).
- Spring: implement `ConsumerSeekAware` in the listener for replay-on-demand.

---

# 8. Consumer groups and rebalancing

## 8.1 Who does what
- **Group coordinator**: a broker (leader of the relevant `__consumer_offsets` partition). Tracks membership, receives heartbeats and offset commits.
- **Group leader**: one consumer of the group (the first to join). It runs the **partition assignor** locally and sends the result to the coordinator.

## 8.2 Rebalance sequence (eager protocol)

```
Trigger: a consumer joins/leaves/dies, a topic's partition count changes, or subscription changes.

 C1        C2 (new)      Coordinator
 |            |               |
 |            |--JoinGroup--->|      1. everyone (re)sends JoinGroup with subscribed topics
 |--JoinGroup(already member)->|     2. (eager) ALL members first REVOKE ALL their partitions  <-- "stop the world"
 |<--JoinGroup resp (C1 = leader, gets member list)--|
 |  3. leader runs assignor (Range/RoundRobin/Sticky/CooperativeSticky) -> assignment map
 |--SyncGroup(assignment)----->|
 |            |--SyncGroup---->|      4. everyone sends SyncGroup; coordinator returns each its assignment
 |<--SyncGroup resp: [P0,P1]---|
 |            |<--resp: [P2]---|
 5. Consumers resume from committed offsets; onPartitionsAssigned callbacks.
Consumption paused cluster-wide for the whole duration.
```

Cost: during the pause, no records from the group are processed, lag grows, and uncommitted in-flight work is redone by the new owners.

## 8.3 Assignors (`partition.assignment.strategy`)

| Assignor | Behavior |
|---|---|
| `RangeAssignor` | per topic, contiguous ranges; can be uneven across many topics. Eager |
| `RoundRobinAssignor` | evenly spreads all partitions. Eager |
| `StickyAssignor` | balanced and minimizes movement. Eager |
| `CooperativeStickyAssignor` | same as sticky, but **incremental cooperative rebalancing**: only partitions that must move are revoked, in two rounds; others keep processing. Default list since 3.0 is `[RangeAssignor, CooperativeStickyAssignor]` (the group upgrades to cooperative after Range is removed from all members) |

Do not mix eager and cooperative assignors carelessly during rolling upgrades: follow the documented two-step rollout.

## 8.4 Static membership
Set `group.instance.id` (unique and stable per consumer, for example the pod ordinal from a StatefulSet). If the member disappears and comes back within `session.timeout.ms`, it gets its old partitions back **without a rebalance**. Trade-off: with a large session timeout, a truly dead member's partitions stay unassigned until timeout. Useful for rolling restarts of Kubernetes deployments.

## 8.5 Rebalance storms and how to avoid them
Causes: slow processing exceeding `max.poll.interval.ms`; autoscaling flapping (HPA adding/removing pods); deployments restarting all pods at once; too-short `session.timeout.ms`; long GC pauses; network flaps.
Mitigations: cooperative sticky assignor, static membership, tune `max.poll.records` and `max.poll.interval.ms`, rolling deployments with `maxUnavailable` small, avoid scaling on CPU with short periods, keep consumer count <= partitions.

## 8.6 KIP-848 (next-gen consumer group protocol)
GA in Kafka 4.0 (opt-in from the client with `group.protocol=consumer`). Moves the assignment logic to the broker-side coordinator, removes the global "sync barrier", makes rebalances incremental and per-member, and removes the client-side `partition.assignment.strategy` (server-side `group.consumer.assignors` instead). Knowing its purpose is enough for interviews: faster, less disruptive rebalances at scale.

---

# 9. Ordering, partition planning, lag

## 9.1 What guarantees ordering
- Same key -> same partition -> ordered **as long as the partition count is constant**.
- Producer: idempotence on (or in-flight=1).
- Consumer: process one partition's records sequentially in offset order.

## 9.2 What breaks ordering
| Cause | Why |
|---|---|
| More than one partition with no key or a key that varies | different partitions have no relative order |
| Retries with `max.in.flight > 1` and idempotence off | reorder of batches |
| Consumer thread pool over one partition's records | tasks finish out of order. Fix: hand off per key (hash key to one single-thread executor) and commit only contiguous completed offsets |
| Changing the partition count | key -> partition mapping changes |
| Non-blocking retry topics (`@RetryableTopic`) | a failed record is delayed while later records go ahead |
| Producer calling `send` from multiple threads/instances with no external sequencing | order = arrival order |
| Different event types on different topics | no ordering between topics. Put related events of one aggregate in one topic, keyed by aggregate id |

## 9.3 Partition count planning
Formula: `partitions = max( target throughput / per-partition producer throughput , target throughput / per-partition consumer throughput )`, and at least the number of consumers you want to run in parallel.

Worked example: target 200 MB/s; a single partition sustains about 20 MB/s on the producer side and a consumer instance processing your logic handles 5 MB/s per partition. `max(200/20, 200/5) = max(10, 40) = 40`. Round up and add headroom (for example 48), since you cannot reduce later. Measure with `kafka-producer-perf-test.sh` and your real consumer logic rather than trusting generic figures.

Facts:
- Max consumer parallelism in a group = number of partitions. Extra consumers sit idle (but are useful as hot standby).
- **Partitions can be increased, never decreased** (create a new topic and migrate).
- Increasing partitions breaks key -> partition mapping for existing keys.
- More partitions cost: file handles, memory, longer leader failover and end-to-end latency (replication per partition), more open segments, larger controller metadata. Roughly, keep to low thousands of partitions per broker in ZooKeeper mode; KRaft raises the ceiling but the cost per partition is not zero.

## 9.4 Lag monitoring
**Lag** = log end offset - committed offset, per partition. Rising lag means consumers are slower than producers or stuck.

- CLI: `kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group billing` shows CURRENT-OFFSET, LOG-END-OFFSET, LAG, CONSUMER-ID, HOST per partition.
- Client metric: `records-lag-max` and `records-lag` in `kafka.consumer:type=consumer-fetch-manager-metrics`. With Spring Boot Actuator and Micrometer, Kafka client metrics are bound automatically as `kafka.consumer.*`, and listener timing as `spring.kafka.listener` (needs `spring.kafka.listener.observation-enabled` or the default timer). Verify names in your Boot version.
- External: **Burrow** (LinkedIn; evaluates lag trend and status rather than a simple threshold), Prometheus `kafka_exporter`, Grafana, Cruise Control, AKHQ/Kafka UI.
- Alert on **trend and time lag** (how many seconds behind), not just an absolute number. A high but shrinking lag after a deployment is fine.
- Broker health: `UnderReplicatedPartitions` (should be 0), `UnderMinIsrPartitionCount`, `OfflinePartitionsCount`, `ActiveControllerCount` (exactly 1 across cluster), `IsrShrinksPerSec` vs `IsrExpandsPerSec`, request handler idle percent, disk usage.

---

# 10. Failure scenarios: event sequence, loss or duplication, mitigation

Assume RF=3, `min.insync.replicas=2`, unless stated.

| # | Event sequence | Result | Mitigation |
|---|---|---|---|
| 1 | Producer `acks=1`; leader acks, then crashes before followers copy | **Loss** of acked record | `acks=all`, min ISR 2 |
| 2 | Producer `acks=all`; leader crashes before all ISR fetch it; producer gets `NotLeaderOrFollowerException` or timeout and retries | No loss. Without idempotence a **duplicate** is possible if the first write did land on the leader and got replicated to the new leader before the ack was lost | idempotent producer (default), consumer-side idempotency |
| 3 | One broker down (of 3), ISR=2 | Writes continue; **no loss, no dup**; under-replicated partitions alert | fix or replace broker; RF=3, min ISR 2 |
| 4 | Two brokers down (of 3), ISR=1 | `acks=all` writes fail with `NotEnoughReplicasException`; reads OK; **no loss of acked data**; availability lost for writes | restore brokers; producers buffer and retry until `delivery.timeout.ms`, then fail: application needs its own retry or outbox |
| 5 | All ISR down, `unclean.leader.election.enable=true`, a stale replica becomes leader | **Loss** of committed data; possible offset divergence; consumers with higher offsets get `OffsetOutOfRange` and reset per `auto.offset.reset` | keep false; accept unavailability rather than loss |
| 6 | Network partition between producer and broker; request written, response lost; producer retries | **Duplicate** (non-idempotent) or **no dup** (idempotent, same PID+seq) | idempotence on |
| 7 | Network partition isolates the leader from followers and the controller | Leader is removed from ISR (after `replica.lag.time.max.ms`), a new leader is elected. With acks=all, the isolated old leader cannot get acks and shrinks ISR to itself; if ISR < min ISR it rejects writes. Fencing by leader epoch stops it from accepting divergent data once it learns of the new epoch | acks=all, min ISR 2. `acks=1` may accept writes on the old leader that are later truncated: **loss** |
| 8 | Producer timeout: `delivery.timeout.ms` expires while retrying | Callback gets `TimeoutException`. The record **may or may not** have been written (unknown outcome) | If app retries by re-sending: duplicates possible unless consumers are idempotent; outbox with retry |
| 9 | Consumer processes a record, then crashes before commit | Restart from last commit: record **processed twice** (at-least-once) | idempotent processing (dedupe table on event id), commit after processing |
| 10 | Consumer commits first (auto-commit or commit before processing), then crashes during processing | **Loss** (at-most-once) | commit after processing; never auto-commit with async processing |
| 11 | Rebalance during processing: partition revoked mid-batch, new owner starts from last commit | **Duplicates** for uncommitted records; old owner's commit fails | commit in `onPartitionsRevoked`, cooperative assignor, small batches, idempotency |
| 12 | Full disk on a broker: log dir goes offline or broker shuts down (`IOException`) | Partitions on that dir go offline; leaders move if replicas exist (RF=3 no loss). With JBOD, only that log dir goes offline | alert on disk at 70 to 80 percent, set retention, add disks/brokers, reassign partitions |
| 13 | Transactional producer crashes mid-transaction | Transaction aborted after `transaction.timeout.ms` (or fenced by the restarted producer with same `transactional.id`). `read_committed` consumers see none of it; `read_uncommitted` consumers **see the aborted data** | consumers use `read_committed`; do not run two instances with the same id |
| 14 | Consumer group offset expired (`offsets.retention.minutes`) then restart with `auto.offset.reset=latest` | **Loss** (skipped data) | `earliest` for must-not-miss consumers; alert before retention; keep groups alive |
| 15 | Poison pill: a record that always throws (or cannot be deserialized) | Consumer blocked, retries forever, lag grows on that partition | `ErrorHandlingDeserializer`, `DefaultErrorHandler` with bounded backoff and DLT |
| 16 | Topic retention shorter than consumer outage | **Loss** (offsets deleted; `OffsetOutOfRange`) | retention >= worst-case outage, lag alerts |

---

# 11. Spring Kafka (Spring Boot 3.x, spring-kafka 3.x)

## 11.1 Dependency and application.yml

```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
<dependency>  <!-- test -->
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka-test</artifactId>
  <scope>test</scope>
</dependency>
```

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      acks: all                                   # wait for all in-sync replicas
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      compression-type: lz4                       # batch compression
      batch-size: 32768                           # max bytes per partition batch
      properties:
        enable.idempotence: true                  # PID + sequence; default already true in 3.x clients
        linger.ms: 10                             # wait up to 10 ms to fill batches
        delivery.timeout.ms: 120000               # total budget for one send incl. retries
        max.in.flight.requests.per.connection: 5  # safe together with idempotence
        spring.json.add.type.headers: false       # do not add __TypeId__ header if consumers are not Java
    consumer:
      group-id: billing
      auto-offset-reset: earliest                 # only when the group has no committed offset
      enable-auto-commit: false                   # Spring commits per its AckMode
      isolation-level: read_committed             # ignore aborted transactional records
      max-poll-records: 50                        # smaller batch = finishes before max.poll.interval.ms
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: com.example.events   # never "*" in production
        spring.json.value.default.type: com.example.events.OrderPlaced
        max.poll.interval.ms: 300000
        session.timeout.ms: 45000
        heartbeat.interval.ms: 15000
    listener:
      ack-mode: manual                            # we call Acknowledgment.acknowledge()
      concurrency: 3                              # 3 consumer threads in this instance (<= partitions)
```

Line-by-line notes on the important ones:
- `acks: all` + `enable.idempotence` + broker/topic `min.insync.replicas=2` = the durable, no-duplicate-on-retry combination.
- `linger.ms` and `batch-size`: throughput knobs (section 6.1).
- `ErrorHandlingDeserializer`: wraps the real deserializer. A record that fails to deserialize does **not** throw out of `poll()` (which would loop forever on the same offset); instead the failure is put in a header and handed to the error handler (`DeserializationException`), which routes it to the DLT. This is the fix for **poison pills** at the deserializer level.
- `spring.json.trusted.packages`: `JsonDeserializer` only instantiates classes from trusted packages (security: prevents deserialization gadget attacks).
- `ack-mode: manual`: `Acknowledgment.acknowledge()` marks the record done; the container commits (for `MANUAL` it batches commits at the end of the poll batch, `MANUAL_IMMEDIATE` commits right away).
- `concurrency`: creates N `KafkaMessageListenerContainer` threads, each with its own consumer, all in the same group. Beyond the number of partitions they are idle.

### AckMode values (`ContainerProperties.AckMode`)
| Mode | Commit happens |
|---|---|
| `RECORD` | after each record's listener returns |
| `BATCH` (default) | after all records from one `poll()` are processed |
| `TIME` / `COUNT` / `COUNT_TIME` | after a time / record count |
| `MANUAL` | when you call `acknowledge()`; commits queued and applied after the batch |
| `MANUAL_IMMEDIATE` | when you call `acknowledge()`, immediately |

Spring's default for `enable.auto.commit` is set to false by the container (it manages commits), which is why the raw client default (true) surprises people who compare.

## 11.2 Java configuration (producer, topics, error handler)

```java
@Configuration
public class KafkaConfig {

    @Bean
    NewTopic ordersTopic() {
        return TopicBuilder.name("orders")
                .partitions(6)
                .replicas(3)
                .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2")
                .config(TopicConfig.RETENTION_MS_CONFIG, String.valueOf(Duration.ofDays(7).toMillis()))
                .build();          // KafkaAdmin creates it at startup if it does not exist
    }

    @Bean
    DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
        // After retries are exhausted, publish to <topic>.DLT on the SAME partition number
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template,
                (rec, ex) -> new TopicPartition(rec.topic() + ".DLT", rec.partition()));
        // 3 retries, 1 s apart, then recover (blocking retries: the partition waits)
        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3L));
        // exceptions that will never succeed: skip retries, go to DLT immediately
        handler.addNotRetryableExceptions(IllegalArgumentException.class, DeserializationException.class);
        return handler;
    }
}
```

Spring Boot auto-detects a `CommonErrorHandler` bean and applies it to the auto-configured `ConcurrentKafkaListenerContainerFactory`. The DLT topic must exist with a partition count at least as high as the source topic if you keep `rec.partition()` (otherwise pass a resolver returning a valid partition or `-1` to let the producer choose).

Note on defaults: `DefaultErrorHandler` with no arguments retries 9 more times (10 deliveries total) with no delay, then logs and **skips** the record. Do not rely on that in production.

## 11.3 Producer code

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderPlaced> kafkaTemplate;

    public CompletableFuture<SendResult<String, OrderPlaced>> publish(OrderPlaced event) {
        // key = orderId -> all events of one order land in one partition -> ordered
        return kafkaTemplate.send("orders", event.orderId(), event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        // unknown outcome after delivery.timeout.ms: must be handled (retry from outbox, alert)
                        log.error("publish failed orderId={}", event.orderId(), ex);
                    } else {
                        RecordMetadata md = result.getRecordMetadata();
                        log.debug("sent partition={} offset={}", md.partition(), md.offset());
                    }
                });
    }
}
```

In Spring Kafka 3.x `KafkaTemplate.send` returns `CompletableFuture<SendResult<K,V>>` (it was `ListenableFuture` in 2.x). `send()` is asynchronous: an exception thrown here is only for synchronous problems (serialization, buffer full after `max.block.ms`); broker errors arrive in the future.

## 11.4 Consumer code (manual ack, idempotent)

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class BillingListener {

    private final BillingService billingService;

    @KafkaListener(topics = "orders", groupId = "billing")
    public void onOrder(ConsumerRecord<String, OrderPlaced> record, Acknowledgment ack) {
        OrderPlaced event = record.value();
        billingService.chargeOnce(record.topic(), record.partition(), record.offset(), event);
        ack.acknowledge();   // AFTER the work -> at-least-once
    }
}

@Service
@RequiredArgsConstructor
class BillingService {
    private final ProcessedEventRepository processed;   // table with UNIQUE(event_id)
    private final InvoiceRepository invoices;

    @Transactional
    public void chargeOnce(String topic, int partition, long offset, OrderPlaced e) {
        // idempotent consumer: insert dedupe row and business row in ONE DB transaction
        try {
            processed.saveAndFlush(new ProcessedEvent(e.eventId()));   // unique key violation => already done
        } catch (DataIntegrityViolationException dup) {
            return;                                                    // duplicate delivery: ignore
        }
        invoices.save(Invoice.from(e));
    }
}
```

Choosing the dedupe key: a business or event id created by the producer (`eventId` UUID in the payload). `topic+partition+offset` also works for replays of the same log but breaks if the event is re-published (a new offset). Clean the table with a retention job aligned to how long duplicates can occur.

## 11.5 Error handling design

Failure types and treatment:

| Failure | Nature | Handling |
|---|---|---|
| Deserialization failure | permanent | `ErrorHandlingDeserializer` -> DLT immediately |
| Validation / bad data | permanent | not-retryable exceptions -> DLT |
| Downstream temporarily down (DB, HTTP 503) | transient | retry with backoff (blocking short retries, or retry topics for long delays) |
| Bug throwing NPE | permanent until fixed | DLT after N attempts, fix, then replay from DLT |

### Blocking retries (`DefaultErrorHandler` + BackOff)
The container **seeks back** to the failed record and redelivers it after the backoff, while the partition's later records wait. Preserves ordering, blocks the partition. Keep the total backoff far below `max.poll.interval.ms` (the container calls `pause` semantics only when it needs to; long sleeps inside the error handler count against `max.poll.interval.ms`: Spring warns if the backoff exceeds it). Use `ExponentialBackOffWithMaxRetries(5)` with `setInitialInterval`, `setMultiplier`, `setMaxInterval`.

### Non-blocking retries (`@RetryableTopic`)

```java
@Component
@Slf4j
public class PaymentListener {

    @RetryableTopic(
        attempts = "4",                                   // 1 original + 3 retries
        backoff = @Backoff(delay = 2000, multiplier = 2.0, maxDelay = 30000),
        dltStrategy = DltStrategy.FAIL_ON_ERROR,
        exclude = { IllegalArgumentException.class },     // no retry for permanent errors, straight to DLT
        autoCreateTopics = "true"
    )
    @KafkaListener(topics = "payments", groupId = "payments-svc")
    public void handle(PaymentRequested event) {
        gateway.charge(event);                            // throws on transient failure
    }

    @DltHandler
    public void dlt(PaymentRequested event, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        log.error("gave up on {} from {}", event, topic);  // alert, store for manual replay
    }
}
```

Topics created: `payments`, `payments-retry-0`, `payments-retry-1`, `payments-retry-2` (or with delay suffixes such as `-retry-2000` depending on `topicSuffixingStrategy`), and `payments-dlt`. The failed record is published to the next retry topic, the main partition **continues immediately**, and the retry listener waits until the record's due time (using pause/resume, no thread sleep beyond the container's own timing) before invoking your method.

**Ordering vs retry trade-off**: non-blocking retries **break per-key ordering** (a later event for the same key can succeed before the earlier one is retried). Use blocking retries when order matters and failures are short; use retry topics when throughput matters more and events are independent or handled commutatively. Never retry forever on a hot partition.

### DLT operations
Headers added by `DeadLetterPublishingRecoverer`: `kafka_dlt-original-topic`, `kafka_dlt-original-partition`, `kafka_dlt-original-offset`, `kafka_dlt-exception-fqcn`, `kafka_dlt-exception-message`, `kafka_dlt-exception-stacktrace`. Build a small tool or a scheduled job to republish DLT records once the bug is fixed. Alert on DLT depth.

## 11.6 Kafka transactions in Spring (Kafka-to-Kafka EOS)

```yaml
spring.kafka.producer.transaction-id-prefix: order-tx-       # enables transactions; Spring appends a suffix per producer
spring.kafka.consumer.properties.isolation.level: read_committed
```

```java
@KafkaListener(topics = "orders", groupId = "enricher")
public void listen(OrderPlaced e) {           // container starts a Kafka transaction around the listener
    kafkaTemplate.send("orders-enriched", e.orderId(), enrich(e));   // in the same transaction
}   // on normal return: sendOffsetsToTransaction + commit. On exception: abort and redeliver.
```

When the container is given a `KafkaAwareTransactionManager` (Boot wires `KafkaTransactionManager` when `transaction-id-prefix` is set), the offsets are sent to the transaction automatically. Combining with a JPA `@Transactional` gives **best-effort one-phase commit**, chained transactions: DB and Kafka commit one after the other, not atomically. A crash between them leaves one done and the other not. This is not exactly-once across DB and Kafka.

## 11.7 Useful listener features
- `@KafkaListener(topicPartitions = @TopicPartition(topic="orders", partitions={"0","1"}))` for manual assignment (no group rebalance).
- `@KafkaListener(batch = "true")` with `List<ConsumerRecord<..>>` (set `spring.kafka.listener.type: batch`); batch error handling is different (`BatchListenerFailedException` to indicate which record failed).
- `ConsumerSeekAware` for replay; `KafkaListenerEndpointRegistry` to start, stop, pause, resume listeners (`id` attribute, `autoStartup=false`).
- `@SendTo` for reply topics; `ReplyingKafkaTemplate` for request/reply (rarely the right choice; prefer HTTP/gRPC for RPC).
- `ContainerProperties#setPauseImmediate` and `pause()` for backpressure.

---

# 12. Serialization and schema evolution

| Format | Pros | Cons |
|---|---|---|
| JSON | human-readable, easy debugging, no extra infra | large, no enforced schema, breaking changes discovered at runtime |
| Avro | compact binary, schema evolution rules well defined, tight Schema Registry support | tooling learning curve; needs the schema to read |
| Protobuf | compact, strongly typed, multi-language, field numbers give safe evolution | schema and code generation management |

**Schema Registry** (Confluent, Apicurio, etc.): schemas are stored centrally under a **subject** (default `<topic>-value`). The serializer writes a **magic byte + 4-byte schema id** before the payload; the deserializer fetches the schema by id (cached). Compatibility mode is enforced **at registration time**, so an incompatible producer deploy fails in CI or startup rather than breaking consumers in production.

## 12.1 Compatibility types (Avro terminology)

| Mode | Guarantee | Allowed changes (Avro) | Upgrade order |
|---|---|---|---|
| `BACKWARD` (Confluent default) | consumers using the **new** schema can read data written with the **previous** schema | delete a field; add a field **with a default** | consumers first, then producers |
| `FORWARD` | consumers using the **previous** schema can read data written with the **new** schema | add a field (old readers ignore it); delete a field **only if it had a default** | producers first, then consumers |
| `FULL` | both directions | add or delete only fields that have defaults | any order |
| `*_TRANSITIVE` | checked against **all** previous versions, not just the last | same rules | |
| `NONE` | no check | anything | |

Example. v1: `{id:string, amount:double}`.
- v2 adds `currency:string default "EUR"`. New reader reading old data uses the default (**backward OK**). Old reader reading new data ignores `currency` (**forward OK**). This is FULL-compatible.
- v2 adds `currency` with **no default**: new reader cannot fill it from old data (**breaks backward**); old readers still fine (forward OK).
- v2 renames `amount` to `total`: breaks both unless you use an alias. Treat rename as add + deprecate + remove over multiple releases.
- v2 changes `amount` from double to string: breaks both.

Practical rules: always give new fields a default; never reuse a removed field's name or (Protobuf) number; use an **envelope with an event version**; deploy in the compatibility mode's required order; since Kafka retains history, a replaying consumer meets **all old versions**, so prefer `*_TRANSITIVE` for topics that are replayed.

Spring configuration for Avro with Confluent (sketch, class names from Confluent's serdes):
```yaml
spring.kafka:
  producer.value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
  consumer.value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
  properties:
    schema.registry.url: http://localhost:8081
    specific.avro.reader: true
```

---

# 13. Kafka Streams, ksqlDB, Connect, Debezium, outbox

## 13.1 Overview
- **Kafka Streams**: a Java library (not a cluster) for stateful stream processing: `KStream`, `KTable`, joins, windowed aggregations, local state stores backed by compacted changelog topics, exactly-once via `processing.guarantee=exactly_once_v2`. Scale by running more instances of the same `application.id` (which is the consumer group id); parallelism is limited by input partitions. Co-partitioning is required for joins (same partition count and key partitioning on both topics).
- **ksqlDB**: SQL-like streaming layer on top of Kafka Streams (Confluent-licensed server).
- **Kafka Connect**: framework for **source** connectors (DB, files, queues to Kafka) and **sink** connectors (Kafka to Elasticsearch, S3, JDBC). Runs in distributed mode with offsets/config stored in Kafka topics. **Single Message Transforms (SMT)** for light changes. No code needed for standard integrations.
- **Debezium**: a set of Connect source connectors for **change data capture**. It reads the database transaction log (Postgres WAL, MySQL binlog) and emits a Kafka record per row change. No polling load on the DB, captures every committed change in order.

## 13.2 Outbox pattern end to end

Problem: "save order in the DB, then publish to Kafka" is a **dual write**. Either can fail independently: DB commits but the publish fails (lost event), or the publish succeeds and the DB rolls back (phantom event).

Solution: write the event to an **outbox table in the same DB transaction** as the business change. A separate relay publishes it.

```
 Order Service (@Transactional)                          Relay                     Kafka
 +----------------------------------+
 | INSERT INTO orders ...           |    (same commit)
 | INSERT INTO outbox (event) ...   | ---------------> poller OR Debezium ------> topic "order.events"
 +----------------------------------+                  (at-least-once)              |
                                                                                    v
                                                          consumers dedupe by event id
```

Schema:
```sql
CREATE TABLE outbox (
  id            UUID PRIMARY KEY,
  aggregate_type VARCHAR(100) NOT NULL,   -- "order"
  aggregate_id   VARCHAR(100) NOT NULL,   -- used as Kafka key -> ordering per aggregate
  event_type     VARCHAR(100) NOT NULL,   -- "OrderPlaced"
  payload        TEXT NOT NULL,           -- JSON
  created_at     TIMESTAMP NOT NULL DEFAULT now(),
  published_at   TIMESTAMP NULL           -- used by the polling variant
);
CREATE INDEX idx_outbox_unpublished ON outbox (created_at) WHERE published_at IS NULL;  -- PostgreSQL partial index
```

Write side:
```java
@Entity @Table(name = "outbox")
public class OutboxEvent {
    @Id private UUID id = UUID.randomUUID();
    private String aggregateType, aggregateId, eventType;
    @Column(columnDefinition = "text") private String payload;
    private Instant createdAt = Instant.now();
    private Instant publishedAt;
    // constructors, getters
}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orders;
    private final OutboxRepository outbox;
    private final ObjectMapper mapper;

    @Transactional                               // ONE local DB transaction: order + outbox row
    public Order place(PlaceOrder cmd) throws JsonProcessingException {
        Order order = orders.save(Order.create(cmd));
        OrderPlaced event = new OrderPlaced(UUID.randomUUID().toString(), order.getId().toString(), order.getTotal());
        outbox.save(new OutboxEvent("order", order.getId().toString(), "OrderPlaced",
                                    mapper.writeValueAsString(event)));
        return order;
    }
}
```

Variant A: **polling publisher**
```java
public interface OutboxRepository extends JpaRepository<OutboxEvent, UUID> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2")) // -2 = SKIP LOCKED in Hibernate 6
    @Query("select o from OutboxEvent o where o.publishedAt is null order by o.createdAt")
    List<OutboxEvent> lockBatch(Pageable page);
}

@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxRelay {
    private final OutboxRepository outbox;
    private final KafkaTemplate<String, String> kafka;

    @Scheduled(fixedDelay = 500)
    @Transactional
    public void publishPending() throws Exception {
        List<OutboxEvent> batch = outbox.lockBatch(PageRequest.of(0, 100));   // several relay instances do not clash
        for (OutboxEvent e : batch) {
            // wait for the broker ack before marking as published. Key = aggregateId keeps per-order ordering.
            kafka.send("order.events", e.getAggregateId(), e.getPayload()).get(10, TimeUnit.SECONDS);
            e.setPublishedAt(Instant.now());
        }
    }   // crash after send but before commit -> event re-sent next run: DUPLICATE, so consumers must dedupe
}
```
Verify the lock hint semantics for your Hibernate version (`jakarta.persistence.lock.timeout` = `-2` means skip locked in Hibernate 6, or use a native query with `FOR UPDATE SKIP LOCKED`). Ordering caveat: if one row keeps failing, it blocks later rows for that aggregate; with multiple relays and SKIP LOCKED, strict global order is not guaranteed, only if you shard by aggregate.

Variant B: **Debezium with the Outbox Event Router** (no polling code; the DB log is the source):
```json
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "***",
    "database.dbname": "orders",
    "topic.prefix": "orders-db",
    "table.include.list": "public.outbox",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.id": "id",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.by.field": "aggregate_type",
    "transforms.outbox.route.topic.replacement": "outbox.event.${routedByValue}",
    "value.converter": "org.apache.kafka.connect.storage.StringConverter"
  }
}
```
Delete or archive outbox rows periodically (an insert followed by a delete in the same transaction still produces a WAL event, so many teams insert and delete immediately). Debezium is also at-least-once: after a connector restart it may re-emit recent changes, so **consumers still dedupe** (the event id is in a header).

Delivery guarantee summary: the outbox turns a possible *loss* into a possible *duplicate*. Idempotent consumers turn that into effectively-once processing.

---

# 14. Event-driven design, security, multi-DC, capacity

## 14.1 Design vocabulary
- **Event**: a fact in the past, immutable, named in past tense (`OrderPlaced`). Producer does not care who listens.
- **Command**: a request to do something (`PlaceOrder`), addressed to one handler; can be rejected. Commands usually go over a queue or HTTP, events over a log.
- **Event-carried state transfer**: the event carries enough data (for example the order's full snapshot) so consumers need not call back the producer. Reduces coupling and runtime dependency, at the cost of duplicated data and schema versioning.
- **Event notification (thin event)**: only an id; the consumer calls back for details. Simple, but couples runtime availability.
- **Event sourcing**: the **state of an aggregate is derived from its event log** (the log is the source of truth). Kafka can be the transport or a store for this, but it is **not by itself** an event store: it lacks per-aggregate optimistic concurrency ("append only if version = N") and cheap load-by-aggregate-id. "Using Kafka" does not equal "event sourcing".
- **CQRS**: separate write model and read model(s). Kafka often feeds read models (Elasticsearch, Redis, SQL views) built by consumers or Kafka Streams. Read models are eventually consistent.
- Topic design: one topic per event type family and aggregate (`order.events`); key by aggregate id; version schemas; avoid one giant "everything" topic and avoid a topic per tenant at scale.

## 14.2 Security
- **Encryption**: `security.protocol=SSL` or `SASL_SSL`. TLS disables zero-copy for those connections (CPU cost).
- **Authentication**: SASL mechanisms `PLAIN`, `SCRAM-SHA-256/512`, `GSSAPI` (Kerberos), `OAUTHBEARER`; or mutual TLS.
- **Authorization**: ACLs (`kafka-acls.sh`), for example allow principal `User:billing` to `Read` topic `orders` and `Read` group `billing`; producers need `Write` (and `Describe`), transactional producers also need `Write` on the `TransactionalId`, idempotent producers `IdempotentWrite` on the cluster (in newer versions the `Write` on a topic suffices).
- Client config example:
```yaml
spring.kafka:
  properties:
    security.protocol: SASL_SSL
    sasl.mechanism: SCRAM-SHA-512
    sasl.jaas.config: org.apache.kafka.common.security.scram.ScramLoginModule required username="billing" password="${KAFKA_PASSWORD}";
  ssl:
    trust-store-location: file:/etc/kafka/truststore.jks
    trust-store-password: ${TRUSTSTORE_PASSWORD}
```
- Never expose brokers publicly; control `advertised.listeners` (see the war story on clients that connect to bootstrap and then fail).

## 14.3 Multi-DC and MirrorMaker 2
- **Stretch cluster** across 3 AZs in one region (low latency): one cluster, `broker.rack`, RF=3. Across regions with high latency, a single stretched cluster is usually a bad idea (acks=all latency, Raft/ISR issues).
- **MirrorMaker 2** (built on Connect): asynchronous replication of topics, configs, ACLs and consumer group offsets between clusters. Remote topics are prefixed with the source cluster alias (`us-east.orders`) by default to avoid loops. Patterns: active-passive (DR), active-active (each region writes local topics and reads the union). Replication is **asynchronous**, so failover has an RPO greater than zero and offsets need translation (MM2 checkpoints and `RemoteClusterUtils`/offset sync).
- Alternatives: Confluent Cluster Linking (byte-for-byte offsets), Uber uReplicator (historic).

## 14.4 Capacity and ops
- **Brokers**: start with at least 3 (RF=3). Add brokers for disk, network, or partition count, then **reassign partitions** (`kafka-reassign-partitions.sh` or Cruise Control). New brokers get no existing partitions automatically.
- **Disk**: `ingest MB/s x retention seconds x RF`, plus 20 to 30 percent headroom. Use several disks (JBOD) or RAID10; XFS or ext4. Sequential throughput matters more than IOPS.
- **Network**: often the first bottleneck. Egress per broker = consumers x ingest + replication. 10 GbE is common.
- **Memory**: **small JVM heap (about 4 to 8 GB)**; leave the rest of RAM to the page cache. Rule of thumb: enough page cache to hold the "hot" data that consumers read in real time (a few minutes of ingest per partition set).
- **G1GC** is the norm; watch GC pauses (they cause ISR shrink).
- **OS**: raise open file limit (`nofile`), `vm.max_map_count` for many segments (each segment has index files mapped into memory), `vm.swappiness` low.
- **Rack awareness**: `broker.rack=az-a`; replicas of one partition spread across racks.
- **Rolling upgrade**: one broker at a time, wait for `UnderReplicatedPartitions=0` before the next; `controlled.shutdown.enable=true` (default) so leadership moves gracefully.
- `auto.create.topics.enable=false` in production (typos create topics with default settings). Set `default.replication.factor=3`, `min.insync.replicas=2`, `offsets.topic.replication.factor=3`.

---

# 15. Kafka vs RabbitMQ vs SQS/SNS vs Pulsar

| | Kafka | RabbitMQ | SQS / SNS | Pulsar |
|---|---|---|---|---|
| Model | distributed log, pull | smart broker queues + exchanges, push | SQS: managed queue; SNS: pub/sub fanout | log/queue hybrid, segment storage on BookKeeper |
| After consume | retained until retention | removed on ack (classic queues; streams and quorum queues differ) | SQS removes on delete after visibility timeout | retained per policy, plus subscription cursors |
| Replay | yes | classic no (RabbitMQ Streams yes) | no (SQS); SNS no | yes |
| Ordering | per partition | per queue (single consumer) | FIFO queues per message group; standard queues best effort | per partition/key (Key_Shared) |
| Throughput | very high (100s of MB/s per cluster) | moderate | managed, scales automatically | high |
| Routing | topic and partition only; filter in consumer | rich (direct, topic, fanout, headers) | SNS filter policies | flexible subscriptions |
| Per-message features | none built in (no per-message TTL, priority, delay) | TTL, priority, delayed via plugin, DLX | delay, visibility timeout, DLQ | delayed delivery, DLQ, negative ack |
| Ops burden | high (self-managed), or managed (MSK, Confluent Cloud) | moderate | none | high (broker + BookKeeper + metadata store) |
| Best for | event streaming, CDC, analytics, event-driven microservices, replay, high volume | task queues, workflows, complex routing, RPC-ish, low latency per message | simple decoupling on AWS, serverless, no ops | multi-tenancy, geo-replication built in, mixed queue+stream needs |

Decision guide:
- "Do this job once with one of N workers, with retry and delay, rich routing" -> RabbitMQ or SQS.
- "Something happened; several systems react; I may need to replay, audit, or rebuild a read model" -> Kafka.
- "AWS only, small team, no ops, at-least-once is fine" -> SQS+SNS (or Kinesis/MSK when you need a log).
- "Need millions of small topics, or a built-in tiered storage and geo-replication with queue and stream semantics" -> evaluate Pulsar.

---

# 16. Testing

| | `@EmbeddedKafka` (spring-kafka-test) | Testcontainers |
|---|---|---|
| What runs | an in-JVM Kafka broker (KRaft in current spring-kafka 3.x versions) | a real Kafka Docker container |
| Startup | fast, no Docker needed | slower, needs Docker |
| Fidelity | good for logic and serialization; not identical to production broker version and config | closest to production; can test with a specific image tag, multi-broker, SASL, Schema Registry container |
| Use for | fast listener and error handler tests in CI without Docker | integration tests, transaction and rebalance behaviors, upgrade rehearsals |

```java
@SpringBootTest
@EmbeddedKafka(partitions = 3, topics = { "orders", "orders.DLT" },
               bootstrapServersProperty = "spring.kafka.bootstrap-servers")
class OrderFlowEmbeddedTest {
    @Autowired KafkaTemplate<String, OrderPlaced> template;
    @Autowired ProcessedEventRepository processed;

    @Test
    void consumesAndDedupes() {
        OrderPlaced e = new OrderPlaced("evt-1", "o-1", BigDecimal.TEN);
        template.send("orders", "o-1", e);
        template.send("orders", "o-1", e);       // duplicate delivery
        Awaitility.await().atMost(Duration.ofSeconds(10))
                  .untilAsserted(() -> assertThat(processed.count()).isEqualTo(1));
    }
}
```

```java
@SpringBootTest
@Testcontainers
class OrderFlowContainerTest {
    @Container
    @ServiceConnection                          // Spring Boot 3.1+: wires spring.kafka.bootstrap-servers automatically
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.1"));
    // ... same test body
}
```
Testing tips: never `Thread.sleep`, use Awaitility; use unique group ids or topics per test; set `auto-offset-reset: earliest` in tests, otherwise a consumer that subscribes after the send misses the message; test the DLT path by sending an undeserializable payload; test idempotence by sending twice.

---

# 17. docker-compose (single-node KRaft) and CLI cheat sheet

## 17.1 docker-compose.yml (official `apache/kafka` image, combined broker+controller)

```yaml
services:
  kafka:
    image: apache/kafka:3.8.0
    container_name: kafka
    ports:
      - "9092:9092"                       # clients on the host connect here
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller                 # one node plays both roles (dev only)
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092 # what clients are told to reconnect to; must be reachable BY THE CLIENT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093       # id@host:port of controller quorum
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1              # single broker: RF cannot be 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1                 # otherwise transactions hang on one broker
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
```
Not executed here (Docker exists on this machine but the stack was not started while writing). If your app also runs in Docker, add a second listener (for example `INTERNAL://kafka:29092`) and advertise it, otherwise you hit the "advertised.listeners" trap (war story 8). Tools inside the container live in `/opt/kafka/bin/` (use `docker exec -it kafka /opt/kafka/bin/kafka-topics.sh ...`). With a non-Docker install, format storage once: `kafka-storage.sh random-uuid` then `kafka-storage.sh format -t <uuid> -c config/kraft/server.properties`.

## 17.2 CLI cheat sheet (from the Kafka distribution `bin/` folder; `.sh` on Linux/macOS, `bin/windows/*.bat` on Windows)

```bash
BS=localhost:9092

# --- topics ---
kafka-topics.sh --bootstrap-server $BS --create --topic orders --partitions 6 --replication-factor 3 \
                --config min.insync.replicas=2 --config retention.ms=604800000
kafka-topics.sh --bootstrap-server $BS --list
kafka-topics.sh --bootstrap-server $BS --describe --topic orders          # leader, replicas, ISR per partition
kafka-topics.sh --bootstrap-server $BS --describe --under-replicated-partitions
kafka-topics.sh --bootstrap-server $BS --alter --topic orders --partitions 12   # increase only
kafka-topics.sh --bootstrap-server $BS --delete --topic orders

# --- topic/broker configs ---
kafka-configs.sh --bootstrap-server $BS --entity-type topics --entity-name orders --describe
kafka-configs.sh --bootstrap-server $BS --entity-type topics --entity-name orders \
                 --alter --add-config retention.ms=86400000,cleanup.policy=compact

# --- console producer / consumer ---
kafka-console-producer.sh --bootstrap-server $BS --topic orders \
                 --property parse.key=true --property key.separator=:        # type  o-1:{"id":1}
kafka-console-consumer.sh --bootstrap-server $BS --topic orders --from-beginning \
                 --property print.key=true --property print.offset=true --property print.partition=true
kafka-console-consumer.sh --bootstrap-server $BS --topic orders --group debug-1 --isolation-level read_committed

# --- consumer groups ---
kafka-consumer-groups.sh --bootstrap-server $BS --list
kafka-consumer-groups.sh --bootstrap-server $BS --describe --group billing            # lag per partition
kafka-consumer-groups.sh --bootstrap-server $BS --describe --group billing --members --verbose
kafka-consumer-groups.sh --bootstrap-server $BS --describe --group billing --state

# --- reset offsets (group must have NO active members). Without --execute it is a dry run. ---
kafka-consumer-groups.sh --bootstrap-server $BS --group billing --topic orders --reset-offsets --to-earliest --execute
kafka-consumer-groups.sh --bootstrap-server $BS --group billing --topic orders:0 --reset-offsets --to-offset 1500 --execute
kafka-consumer-groups.sh --bootstrap-server $BS --group billing --topic orders --reset-offsets --to-datetime 2026-01-15T10:00:00.000 --execute
kafka-consumer-groups.sh --bootstrap-server $BS --group billing --topic orders --reset-offsets --shift-by -1000 --execute
kafka-consumer-groups.sh --bootstrap-server $BS --group billing --delete

# --- inspect log files, elections, quorum, perf ---
kafka-dump-log.sh --files /var/kafka-logs/orders-0/00000000000000000000.log --print-data-log
kafka-leader-election.sh --bootstrap-server $BS --election-type preferred --all-topic-partitions
kafka-metadata-quorum.sh --bootstrap-server $BS describe --status                    # KRaft controller quorum
kafka-reassign-partitions.sh --bootstrap-server $BS --generate ...                   # move partitions between brokers
kafka-producer-perf-test.sh --topic perf --num-records 1000000 --record-size 1024 --throughput -1 \
                 --producer-props bootstrap.servers=$BS acks=all linger.ms=10 batch.size=65536
kafka-consumer-perf-test.sh --bootstrap-server $BS --topic perf --messages 1000000
kafka-acls.sh --bootstrap-server $BS --add --allow-principal User:billing --operation Read --topic orders --group billing
```

---

# 18. Production war stories (symptom -> diagnosis -> fix)

**1. Consumer group in an endless rebalance loop**
- Symptom: lag climbing, logs full of "Attempt to heartbeat failed since group is rebalancing" and "poll timeout expired". CPU low.
- Diagnosis: `kafka-consumer-groups --describe --state` shows the group flipping between `PreparingRebalance` and `CompletingRebalance`. A downstream API got slower; a batch of 500 records now takes more than `max.poll.interval.ms`.
- Fix: `max.poll.records` from 500 to 20; timeouts on the HTTP client; `max.poll.interval.ms` raised only with a bound; static membership and cooperative sticky assignor; idempotent consumer because duplicates were created during the loop.

**2. Silent data loss after a leader failover**
- Symptom: a few orders missing downstream after a broker restart. Producer logs show no errors.
- Diagnosis: producers were configured `acks=1` (an old library config). The broker restart moved leadership before followers had copied the latest records.
- Fix: `acks=all`, `min.insync.replicas=2`, `unclean.leader.election.enable=false`, `enable.idempotence=true`; add a reconciliation job (count events per hour at source vs sink).

**3. Producers fail with NotEnoughReplicasException**
- Symptom: after one broker crashed, writes fail although two brokers are alive.
- Diagnosis: `--describe` shows ISR size 1 for many partitions. RF=3, min ISR=2, and a second broker had been dropped from the ISR by a long GC pause plus disk saturation from a big replay by another consumer group (page cache evicted).
- Fix: fix the slow broker (GC, disk), throttle the replaying consumer (quotas: `consumer_byte_rate`), keep RF/min ISR as is (the protection worked), alert on `UnderMinIsrPartitionCount`.

**4. Duplicate charges after a consumer restart**
- Symptom: customers charged twice after a deployment.
- Diagnosis: the listener called the payment gateway and committed offsets afterwards with `enable.auto.commit=true` and a 5 s interval. Pods were killed mid-batch (SIGKILL after grace period), so uncommitted records were replayed.
- Fix: manual ack after work; idempotency key sent to the gateway (event id); dedupe table; graceful shutdown (the listener container stops on context close; set `terminationGracePeriodSeconds` above the worst-case batch time).

**5. One partition has huge lag, others zero (hot partition)**
- Symptom: lag on partition 4 only; adding consumers did not help.
- Diagnosis: the key was `tenantId`, one tenant produced 60 percent of traffic. Per-partition bytes-in confirmed it. Consumers cannot share one partition.
- Fix: key by `orderId` (ordering per order is enough); or salt the key for the hot tenant; for an urgent drain, a consumer that fans out per key into worker threads with contiguous-offset commits.

**6. Poison pill blocks a partition**
- Symptom: same stack trace repeating every few milliseconds; lag stuck; one offset never advances.
- Diagnosis: JSON with a wrong type was published; `JsonDeserializer` threw in `poll()`, so the same record was fetched again and again.
- Fix: `ErrorHandlingDeserializer`; `DefaultErrorHandler` + `DeadLetterPublishingRecoverer`; producers validate schema (Schema Registry with a compatibility check); to unblock immediately, reset the offset by +1 with `--reset-offsets --shift-by 1` (inactive group) and save the bad record.

**7. Disk full on a broker**
- Symptom: broker offline, `No space left on device`, under-replicated partitions.
- Diagnosis: a topic with `retention.ms=-1` by mistake, and a consumer stuck so nobody noticed growth; also a large `segment.bytes` delayed deletion.
- Fix: temporarily reduce `retention.ms` on the big topic via `kafka-configs --alter`, free space, restart; alert at 70 percent; per-topic ownership and retention review; add capacity and rebalance with Cruise Control.

**8. "Connection to node -1 could not be established" or client connects then times out**
- Symptom: `bootstrap-servers` is correct and reachable, but produce or consume times out (typical in Docker or Kubernetes).
- Diagnosis: the client connects to bootstrap, receives cluster metadata containing each broker's **`advertised.listeners`**, and reconnects to that address. If it is `localhost` or an internal hostname, the client cannot reach it.
- Fix: correct advertised listeners for each network (internal and external listener names), DNS reachable from clients.

**9. Consumers lose a week of data after a holiday shutdown**
- Symptom: after restarting a rarely used service, it processes only new events.
- Diagnosis: the group was inactive beyond `offsets.retention.minutes`, so its committed offsets were deleted; `auto.offset.reset` was `latest` (the default).
- Fix: `earliest` for consumers that must not skip; raise `offsets.retention.minutes`; keep a heartbeat consumer; alert on group age.

**10. Ordering violated between "OrderCreated" and "OrderPaid"**
- Symptom: `OrderPaid` processed before `OrderCreated` for a few orders.
- Diagnosis: the events were on two topics (`order-created`, `order-paid`), so there is no ordering guarantee between them. Another cause seen: `@RetryableTopic` delayed the failed `OrderCreated`.
- Fix: one topic per aggregate stream keyed by orderId; or make consumers tolerate reordering (state machine, park events until prerequisites arrive).

**11. Transactional producer "Producer fenced"**
- Symptom: `ProducerFencedException` in a rolling deploy.
- Diagnosis: a new pod started with the same `transactional.id` while the old pod was still running. The new pod's `initTransactions` bumped the epoch and fenced the old one, which is the correct behavior but caused failures during the overlap.
- Fix: `transactional.id` unique per instance derived from a stable identity, treat `ProducerFencedException` as fatal for that instance, and shorten overlap.

**12. read_committed consumers stall at a point in time**
- Symptom: consumers of a topic stop advancing though producers keep writing.
- Diagnosis: a transactional producer had an open, hung transaction. The LSO does not advance beyond it; `read_committed` consumers wait until the coordinator aborts it at `transaction.timeout.ms`.
- Fix: keep transactions short, set an appropriate `transaction.timeout.ms`, find open transactions with `kafka-transactions.sh` (`list`, `describe`).

---

# 19. Interview questions (50)

Legend: E = Easy, M = Medium, H = Hard. Each has a model answer, follow-up chain, and where useful a common wrong answer.

## Easy

**Q1 (E). What is Kafka and what problem does it solve?**
A distributed, replicated, partitioned commit log for high-throughput event streaming. It decouples producers from consumers, retains data for replay, and scales horizontally. Typical uses: event-driven microservices, log and metrics pipelines, CDC, stream processing.
Follow-ups: How is it different from a queue? (Records are retained, many groups read independently, and consumers track their position.) Is it a database? (It can store data durably, but has no ad-hoc querying or per-record updates.)
Wrong: "Kafka is a message queue like RabbitMQ that is just faster."

**Q2 (E). Explain topic, partition, offset.**
A topic is a named stream. It is split into partitions, each an ordered immutable log. An offset is a record's sequence number within one partition.
Follow-ups: Is the offset unique across the topic? (No, per partition; identity is topic-partition-offset.) Can offsets have gaps? (Yes, compaction and transaction markers.)

**Q3 (E). What is a consumer group?**
Consumers sharing a `group.id`. Each partition is assigned to one consumer in the group; multiple groups each get all data.
Follow-ups: 10 consumers, 6 partitions? (4 idle.) 3 consumers, 6 partitions? (2 each.) Can two groups consume the same topic? (Yes, independently.)

**Q4 (E). How does Kafka decide which partition a record goes to?**
Explicit partition if given; else murmur2 hash of the key modulo partition count; else sticky for null keys (batch-wise).
Follow-ups: What happens when you add partitions? (Mapping changes, ordering per key across the change breaks.) How do you keep events of one order ordered? (Key = orderId.)
Wrong: "Round robin per record" (that was pre-2.4 behavior for null keys) or "hash of the value".

**Q5 (E). Where is ordering guaranteed?**
Only within a partition. Not across partitions or topics.
Follow-ups: How do you get a global order? (One partition, sacrificing parallelism; usually you redesign to need only per-key order.)

**Q6 (E). What is a broker, and what is the replication factor?**
A broker is a Kafka server storing partitions. RF is the number of copies of each partition across brokers (1 leader + RF-1 followers).
Follow-ups: Can RF exceed broker count? (No.) Typical production RF? (3.)

**Q7 (E). What is retention? Are messages deleted after consumption?**
No. Deletion is by time or size (`retention.ms`, `retention.bytes` per partition), by whole segments, or by compaction.
Follow-ups: What if a consumer is slower than retention? (It loses data; `OffsetOutOfRange`.)

**Q8 (E). What does `acks` mean?**
0: no ack; 1: leader ack; all: all in-sync replicas ack (subject to `min.insync.replicas`).
Follow-ups: Which loses data on leader crash? (1.) Default in Kafka 3.x clients? (all.)

**Q9 (E). What is an offset commit and where is it stored?**
Saving the next-to-read offset for (group, topic, partition) in the internal topic `__consumer_offsets`.
Follow-ups: Committed offset = last processed or last processed + 1? (+1.) Who stores it in old versions? (ZooKeeper, before 0.9.)

**Q10 (E). Auto commit vs manual commit?**
Auto commits periodically inside `poll()`; manual gives control (after processing). Use manual (or Spring `AckMode`) for correctness.
Follow-ups: Risk of auto commit? (Loss with async processing, replay window.)

**Q11 (E). What is a Dead Letter Topic?**
A topic where records that cannot be processed after retries are published, so they do not block the partition and can be inspected/replayed.
Follow-ups: How does Spring name it? (`<topic>.DLT` for `DeadLetterPublishingRecoverer`; `-dlt` suffix for `@RetryableTopic`.)

**Q12 (E). Kafka vs RabbitMQ in one paragraph.**
Kafka: log, pull, retention, replay, huge throughput, ordering per partition. RabbitMQ: smart broker with exchanges, push, per-message ack and deletion, flexible routing, TTL/priority/delay features. Choose Kafka for event streams and fan-out with replay; RabbitMQ for task queues and rich routing.
Wrong: "Kafka is always better since it is faster."

## Medium

**Q13 (M). Why is Kafka fast?**
Sequential append-only writes, page cache reliance (durability from replication, not fsync), zero-copy `sendfile` for reads (not with TLS), batching, compression, partition parallelism, same format on wire and disk.
Follow-ups: Why does TLS hurt? (Disables zero-copy, adds CPU.) Why a small heap? (Data lives in page cache, not heap.) Does Kafka fsync every write? (No, by default it leaves flushing to the OS.)

**Q14 (M). Explain ISR, high watermark, LEO.**
LEO: next offset to be written in a replica. ISR: leader plus followers caught up within `replica.lag.time.max.ms`. HW: smallest LEO within ISR; consumers see only offsets below the HW.
Follow-ups: What happens to a slow follower? (Removed from ISR, re-added after catching up.) What if ISR shrinks to 1 and `acks=all` with min ISR 2? (Writes rejected.)

**Q15 (M). Configure Kafka for no data loss.**
Producer: `acks=all`, idempotence on, retries with `delivery.timeout.ms`. Topic: RF=3, `min.insync.replicas=2`, `unclean.leader.election.enable=false`. Consumer: commit after processing. Also handle send failures in application code (outbox).
Follow-ups: What still can be lost? (Data from `send()` failures nobody handled; if all replicas die; disk corruption without replicas; consumer commit-before-process.) What does RF=3 with min ISR=3 do? (No broker can fail without write outage.)
Wrong: "acks=all is enough."

**Q16 (M). What is `min.insync.replicas` and how does it interact with `acks`?**
It applies only when `acks=all`. If ISR size < the value, the leader rejects writes with `NotEnoughReplicasException` (or `...AfterAppend`). Trades availability for durability.
Follow-ups: RF=3, min ISR=2: how many broker failures for writes? (1.) For no loss? (Data survives 1 loss at ack time; two losses can lose recent writes if both ISR members die.)

**Q17 (M). Explain the idempotent producer.**
Broker assigns PID; producer sequences batches per partition; broker dedupes retries and rejects gaps. Default on in 3.0+. Scope: one producer session, not across restarts.
Follow-ups: How many in-flight requests are safe? (Up to 5.) Does it prevent application-level duplicates? (No.) What changes with `transactional.id`? (Stable PID across restarts plus atomic writes and fencing.)

**Q18 (M). What is `max.in.flight.requests.per.connection` and how does it affect ordering?**
Number of unacked requests per broker connection. With retries and no idempotence, >1 can reorder. With idempotence, up to 5 is safe.
Follow-ups: Why 5? (Broker keeps metadata for the last 5 batches per PID/partition.)

**Q19 (M). Describe `linger.ms`, `batch.size`, `buffer.memory`.**
`linger.ms` waits to fill batches; `batch.size` max bytes per partition batch; `buffer.memory` total unsent memory; when exhausted `send()` blocks up to `max.block.ms`.
Follow-ups: How to raise throughput? (linger 5 to 20 ms, batch 64 to 256 KB, compression.) Does linger 0 mean no batching? (Records queued during in-flight requests still batch.)

**Q20 (M). Difference between `session.timeout.ms`, `heartbeat.interval.ms`, `max.poll.interval.ms`.**
See table in section 7.3. Heartbeat thread detects dead processes (session); poll interval detects stuck or slow processing.
Follow-ups: A consumer with a slow handler: which fires? (`max.poll.interval.ms`, since heartbeats continue.) What is the fix? (smaller `max.poll.records`, faster processing, async with pause/resume.)
Wrong: "session.timeout.ms is how long processing may take."

**Q21 (M). What does `auto.offset.reset` do?**
Only applies when no valid committed offset exists. `earliest`, `latest`, `none`.
Follow-ups: Committed offset expired? (Treated as none.) Changing it for an existing group with offsets? (No effect.)

**Q22 (M). At-most-once, at-least-once, exactly-once: how in Kafka?**
Commit before processing = at-most-once. Commit after = at-least-once. Exactly-once = idempotent producer + transactions + read_committed, only within Kafka (consume-process-produce). External side effects need idempotence.
Follow-ups: Can you have EOS with a DB write? (Only effectively-once via idempotent upsert or store offsets in the DB transactionally and seek on startup.)

**Q23 (M). What is log compaction? Give a use case.**
Keeps the latest value per key; tombstones delete keys. Use: table-like state, KTable changelogs, configuration, `__consumer_offsets`, CDC snapshots.
Follow-ups: Does compaction delete old records immediately? (No, background, never the active segment.) Ordering preserved? (Yes.) Null-key record? (Rejected on compacted topics.)

**Q24 (M). What is a consumer rebalance and what triggers it?**
Reassignment of partitions among group members. Triggers: member joins/leaves/crashes, missed heartbeat or poll interval, partition count change, subscription change.
Follow-ups: Eager vs cooperative? (Eager revokes all first; cooperative moves only affected partitions.) What is static membership for? (Restarts without rebalance.)

**Q25 (M). How do you choose the number of partitions?**
Target throughput divided by per-partition throughput of the slower side (producer or consumer), at least desired consumer parallelism, plus headroom; consider the cost of too many partitions.
Follow-ups: Can you reduce? (No.) Effect of increasing? (key mapping changes.) Why not 10,000? (File handles, failover time, metadata, memory.)

**Q26 (M). How do you handle poison pills in Spring Kafka?**
`ErrorHandlingDeserializer` for deserialization failures; `DefaultErrorHandler` with bounded `BackOff` and `DeadLetterPublishingRecoverer`; classify non-retryable exceptions.
Follow-ups: Default behavior of `DefaultErrorHandler`? (10 attempts, no delay, then log and skip.) Why bounded backoff? (blocks partition, `max.poll.interval.ms`.)

**Q27 (M). Blocking retry vs `@RetryableTopic`?**
Blocking: seek back and retry in place; preserves order; blocks partition. Non-blocking: publish to retry topics with delays; partition proceeds; breaks ordering; more topics.
Follow-ups: When do you pick which? (Order-critical and short failures: blocking; independent events and long downtime: retry topics.)

**Q28 (M). How do you make a consumer idempotent?**
Dedupe on a unique event id (unique constraint in a `processed_events` table, in the same DB transaction as the effect), or natural idempotence (upsert, set-not-increment), or pass idempotency keys downstream.
Follow-ups: How long to keep dedupe rows? (Longer than the maximum duplicate window: retention plus replay policy.) What if the effect is an email? (Track "sent" before/with idempotency key; accept rare duplicates.)

**Q29 (M). What is the outbox pattern and why?**
Avoids the dual-write problem: write event to an outbox table in the same DB transaction as the state change; a relay (poller or Debezium) publishes to Kafka. At-least-once; consumers dedupe.
Follow-ups: Polling vs CDC? (CDC: lower latency, no polling load, needs Connect/Debezium and DB config; polling: simple, more load and latency.) How to preserve order? (Key by aggregate id, single-writer per aggregate, order by created_at/sequence.)

**Q30 (M). Schema Registry and compatibility modes.**
Central schema store; messages carry schema id; compatibility check on registration. BACKWARD: new readers read old data (upgrade consumers first). FORWARD: old readers read new data (producers first). FULL: both.
Follow-ups: Add a field without default in BACKWARD? (Not allowed.) Why TRANSITIVE? (Replay meets all old versions.)

**Q31 (M). How do you monitor consumer lag?**
`kafka-consumer-groups --describe`, `records-lag-max` metric, Burrow, exporters. Alert on lag trend and time lag, plus broker health metrics.
Follow-ups: Lag zero but data missing? (Committed before processing, or consumer looking at the wrong topic/group.) What does growing lag with idle CPU suggest? (Blocked on downstream, rebalance loop, or stuck partition.)

**Q32 (M). What are the limits of "exactly-once" in Kafka?**
Only for Kafka-to-Kafka read-process-write with transactions. Not for DB writes, HTTP calls, emails. Also requires `read_committed` consumers downstream.
Follow-ups: Cost? (Latency, coordination.) Difference of `exactly_once_v2` in Streams? (Uses the KIP-447 group-metadata approach, fewer producers.)

**Q33 (M). Difference between Kafka Streams and Kafka Connect and a plain consumer.**
Consumer: raw API. Streams: library for stateful processing with state stores, joins, windows, exactly-once. Connect: config-driven integration of external systems.
Follow-ups: How does Streams scale? (More instances of the same `application.id`, up to partition count.)

## Hard

**Q34 (H). Walk through what happens when a leader crashes with `acks=all`, RF=3, min ISR=2.**
Controller detects broker loss (session/Raft heartbeat), picks a new leader from the ISR (bumps leader epoch), followers truncate divergent tails using epoch, producers receive `NotLeaderOrFollowerException`, refresh metadata, and retry to the new leader (idempotence dedupes). Consumers were only reading below HW, so they saw nothing that vanished. Recovering old leader rejoins as follower, truncates using the epoch, and catches up.
Follow-ups: What if ISR = {leader} only when it crashes? (No ISR replica: partition offline unless unclean election is enabled.) Why leader epoch rather than HW truncation? (KIP-101: HW-based truncation could lose or diverge data in rare crash sequences.)

**Q35 (H). Explain the classic failure where the consumer is kicked out because processing is slow. How do you diagnose and fix?**
See section 7.3. Diagnose from logs (poll timeout), `--describe --state`, rebalance metrics; fix batch size and timeouts, decouple slow work, ensure idempotency. Commit failures (`CommitFailedException`) are the symptom, duplicates the consequence.
Follow-ups: Why does the heartbeat thread not save it? (Separate concern: liveness of process vs progress.) Consumer thread pool designs? (pause partitions, process async, resume, commit contiguous offsets.)

**Q36 (H). How do transactions work end to end? What is the LSO and fencing?**
See 6.5: coordinator, `__transaction_state`, control markers, epoch fencing, LSO for `read_committed`.
Follow-ups: What happens to a hung transaction? (Aborted after timeout; blocks read_committed consumers until then.) Why unique `transactional.id` per instance? (Same id fences the other.) Do transactions span DB? (No.)

**Q37 (H). Design an exactly-once pipeline from Kafka topic A to a PostgreSQL table.**
Two options: (1) idempotent upsert keyed by event id or business key, with at-least-once consumption; (2) store the consumed offset in the same DB transaction as the data, disable Kafka commits, on startup/rebalance `seek` to the stored offset (`ConsumerSeekAware` / `onPartitionsAssigned`). Both give effectively-once.
Follow-ups: What about rebalances? (On assign, read offsets from DB and seek.) Why not two-phase commit? (Kafka does not support XA.)

**Q38 (H). How does rebalancing work at protocol level and how to reduce its impact?**
JoinGroup, leader selection, assignor on leader, SyncGroup; eager revokes everything. Reduce: cooperative sticky, static membership, tune timeouts, fewer restarts. KIP-848 moves to broker-side incremental assignment (GA in 4.0).
Follow-ups: What is onPartitionsRevoked used for? (Commit offsets, flush state.) What is the "stop-the-world" in eager? (All members give up all partitions before reassign.)

**Q39 (H). What are the ways ordering can be violated even with a key and one partition per key?**
Producer retries without idempotence and in-flight>1; consumer thread pool; retry topics; partition count change; multiple producers to the same key with no sequencing; consumer-side reprocessing after rebalance (order retained but duplicates appear); different topics for related events.
Follow-ups: How would you parallelize consumption while preserving per-key order? (One queue or thread per key hash bucket; commit only the lowest contiguous completed offset.)

**Q40 (H). You need to increase partitions on a live keyed topic. What are the risks and the safe procedure?**
Risks: key->partition mapping changes, so ordering across the boundary is lost; compacted topics may hold duplicates of a key in two partitions; consumers with per-partition state (Streams, local caches) become inconsistent.
Procedure: create a new topic with the target count, dual-write or mirror with a consumer that re-keys, drain and switch consumers at a cutover offset, or pause producers, drain consumers, alter partitions, resume, and accept a controlled ordering blip.
Follow-ups: For Streams? (Repartition topics and state stores require reset; usually new application id.)

**Q41 (H). Explain how log compaction handles deletes and what pitfalls exist.**
Tombstones (key + null value) retained for `delete.retention.ms` so slow consumers see them; then removed. Pitfalls: consumers starting late might never see a delete if they read after tombstone removal (treat absence as delete only via snapshots), compaction timing is not guaranteed, the active segment is not compacted, offsets have gaps, and keys must be non-null. GDPR "right to erase" uses tombstones, but old segments in backups and consumers' stores remain.
Follow-ups: How to force earlier compaction? (`min.cleanable.dirty.ratio` lower, `segment.ms` smaller, `max.compaction.lag.ms`.)

**Q42 (H). ZooKeeper vs KRaft: what changed and why?**
Metadata moved into an internal Raft log on a controller quorum; faster controller failover, one system to operate, higher partition limits; ZooKeeper deprecated in 3.5 and removed in 4.0. Migration path: ZK to KRaft migration mode available in 3.x bridge releases.
Follow-ups: Combined vs dedicated controller nodes? (Dedicated for large production.) How many controllers? (3 or 5; majority quorum.)

**Q43 (H). A consumer group has lag only on some partitions after a broker restart. Diagnose.**
Check per-partition leader and ISR (`--describe`), preferred leader imbalance (leaders concentrated on few brokers causing overload), under-replicated partitions, consumer assignment skew, slow keys. Fix: preferred leader election, Cruise Control rebalance, check consumers on the affected partitions.
Follow-ups: What does `auto.leader.rebalance.enable` do? (Periodically restores preferred leaders.)

**Q44 (H). How do you design a DLT and replay strategy so no message is lost and order is respected where needed?**
Route failures with headers preserving origin metadata; alert on DLT growth; tool to replay into the original topic (or a retry topic) with original key; for order-sensitive keys, block the key (park subsequent events for that key) or use blocking retries; ensure idempotent consumers; include a poison pill tag to prevent replay loops (max replay count header).
Follow-ups: DLT partition count? (Same or higher than source if mapping partitions one to one.) What if the DLT publish fails? (The recoverer failure makes the error handler redeliver; do not silently skip.)

**Q45 (H). How would you migrate a service from ZooKeeper-era consumer group offsets or from another cluster with minimal downtime?**
Within a cluster: stop group, reset offsets by timestamp. Across clusters: MirrorMaker 2 with checkpoint/offset sync (`sync.group.offsets.enabled`) and translation, run consumers in the target with translated offsets; accept duplicates since replication is asynchronous, so consumers must be idempotent.
Follow-ups: RPO? (>0, async.) Active-active loops? (Remote topic prefixing prevents.)

**Q46 (H). Why can `acks=all` still lose data and how does `min.insync.replicas=1` matter?**
With ISR shrunk to the leader, "all" means the leader only; a crash loses. Also `unclean.leader.election.enable=true`, or losing all replicas, or producer-side failures ignored. Set min ISR 2.
Follow-ups: How do you detect ISR shrink? (`IsrShrinksPerSec`, `UnderReplicatedPartitions`.)

**Q47 (H). Design topic and partition strategy for an order platform with 50k events/s, 1 KB each, needing per-order ordering and 3 consumer services.**
Throughput 50 MB/s in; with RF=3 the cluster writes 150 MB/s plus 3 consumer groups reading 150 MB/s (page cache). Key by orderId; partitions roughly 48 (50 MB/s / ~5 MB/s per consumer partition = 10, then headroom for growth and parallelism; validate by load test). RF=3, min ISR 2, 3 to 6 brokers across 3 AZs, retention 7 days: 50 MB/s x 604800 s x 3 = about 90 TB, so size disks accordingly (or reduce retention, compress with lz4/zstd, use tiered storage if available). One topic per event family, schema registry with backward-transitive compatibility.
Follow-ups: What if one order gets 1000x more events? (Not likely; but for hot keys, restructure.) What compress ratio? (Measure with `compression-rate-avg`.)

**Q48 (H). Explain event sourcing vs using Kafka as an event log. Can Kafka be an event store?**
Event sourcing derives aggregate state from its event stream and needs optimistic concurrency per aggregate and efficient load of one aggregate's stream. Kafka lacks per-aggregate append conditions and cheap reads by key without scanning, so most use a DB-based event store and publish to Kafka; Kafka with compaction can hold latest state (event-carried state), which is not event sourcing.
Follow-ups: How to rebuild read models? (Reset consumer group offset to earliest or new group.)

**Q49 (H). How would you handle a "retry storm": downstream is down and all consumers retry, filling retry topics?**
Use bounded retries with exponential backoff; circuit breaker; pause the container (`pause()`) instead of consuming and failing; long delays with retry topics; alert; DLT when exhausted; throttle producers upstream. Avoid unbounded blocking retries (rebalance risk).
Follow-ups: How does Spring pause? (`MessageListenerContainer.pause()`; error handler `DefaultErrorHandler` can be combined with a circuit breaker; `ContainerProperties` pause options.)

**Q50 (H). What would you check before saying a Kafka cluster is healthy?**
Exactly one active controller, zero offline partitions, zero under-replicated and under-min-ISR partitions, ISR shrink and expand balanced, request queue and handler idle > 30 percent, disk below 70 percent, network below saturation, leader distribution balanced, produce and fetch latency p99, consumer lag trend, DLT depth, certificates expiry, and ability to fail over (rolling restart test).

## Common wrong answers (quick list)
- "Kafka guarantees ordering" (only per partition).
- "Exactly-once means my DB write happens once" (only within Kafka).
- "Adding partitions is free and safe" (breaks key mapping).
- "More consumers than partitions increases throughput" (idle consumers).
- "`session.timeout.ms` limits processing time" (that is `max.poll.interval.ms`).
- "`acks=all` guarantees no loss" (needs min ISR >= 2 and no unclean election).
- "Kafka deletes a message once all consumers read it" (retention/compaction only).
- "`max.poll.records` changes how much is fetched from the broker" (it caps what poll returns).
- "Committed offset is the last processed offset" (it is the next to read).
- "Retries are safe" (duplicates and reordering unless idempotent).

---

# 20. One-page cheat sheet

```
MODEL      topic -> partitions (ordered logs) -> segments (.log .index .timeindex); offset = position in partition
ORDER      per partition only; key -> murmur2 % partitions; null key -> sticky; adding partitions breaks mapping
STORAGE    retention by whole segment (time/size per partition); compaction keeps latest per key; tombstone = null value
SPEED      sequential I/O + page cache + sendfile (no TLS) + batching + compression; small heap, big page cache
REPLICATE  leader + followers; ISR (replica.lag.time.max.ms 30s); HW = min LEO(ISR); leader epoch on failover
DURABLE    acks=all + min.insync.replicas=2 + RF=3 + unclean.leader.election.enable=false + idempotence
ACKS       0 lose anything | 1 lose on leader crash | all safe (with min ISR>=2)
PRODUCER   linger.ms, batch.size, buffer.memory, compression.type, delivery.timeout.ms, max.in.flight<=5 w/ idempotence
IDEMPOTENT PID + per-partition seq; dedupes retries within one producer session only
TXN        transactional.id unique/stable; initTransactions, begin, sendOffsetsToTransaction, commit; read_committed (LSO)
EOS        Kafka -> Kafka only. DB/HTTP side effects NOT covered -> idempotent consumer / outbox
CONSUMER   poll loop; max.poll.records 500; max.poll.interval.ms 5min (slow processing), session.timeout.ms 45s (dead), heartbeat 3s
COMMIT     after processing = at-least-once; before = at-most-once; committed offset = next to read; __consumer_offsets
RESET      auto.offset.reset only when NO committed offset; CLI reset needs inactive group; --execute to apply
REBALANCE  eager = revoke all; CooperativeStickyAssignor = incremental; group.instance.id = static; KIP-848 GA in 4.0
PARTITIONS N = max(target/producer_per_part, target/consumer_per_part); cannot shrink; consumers <= partitions
LAG        kafka-consumer-groups --describe; records-lag-max; Burrow; alert on trend
SPRING     acks/idempotence yml; ErrorHandlingDeserializer; DefaultErrorHandler(recoverer, FixedBackOff);
           DeadLetterPublishingRecoverer (<topic>.DLT); @RetryableTopic (non-blocking, breaks order); AckMode.MANUAL
SCHEMA     BACKWARD: new reader reads old (consumers first) | FORWARD: old reader reads new (producers first) | FULL both
OUTBOX     order + outbox row in ONE DB tx; poller or Debezium EventRouter; consumers dedupe by event id
KRAFT      metadata in __cluster_metadata Raft log; ZooKeeper removed in 4.0; process.roles, node.id, controller.quorum.voters
OPS        alert: UnderReplicatedPartitions, UnderMinIsr, OfflinePartitions, ActiveControllerCount=1, disk<70%, lag trend
LOSS       acks=1 crash | commit-before-process | unclean election | offset expiry + latest | retention < outage
DUPLICATE  retry w/o idempotence | crash before commit | rebalance mid-batch | outbox re-publish | Debezium restart
```
