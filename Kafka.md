# Kafka (Theory + Spring Implementation + Outbox)

## Simple idea
Kafka is a **distributed, append-only log**. Producers write events to the end; consumers read at their own pace and can **replay**.

**Analogy:** RabbitMQ = post office (letter delivered to one person, then gone). Kafka = **CCTV recording / newspaper archive** – everything stays for N days; any reader can rewind.

## Architecture
```
Producer ──► Topic "orders"
              ├─ Partition 0: [m0][m1][m4][m7]...   ← offsets
              ├─ Partition 1: [m2][m5][m8]...
              └─ Partition 2: [m3][m6][m9]...
                        │
      Consumer Group "billing"        Consumer Group "email"
        C1 ← P0                          C1 ← P0, P1, P2
        C2 ← P1                          (each group gets ALL messages)
        C3 ← P2
```
- **Topic**: category of messages. **Partition**: unit of parallelism + ordering.
- **Offset**: message position inside a partition.
- **Ordering guaranteed only inside a partition.** Same key → same partition (`hash(key) % partitions`) → use `orderId` as key to keep an order's events in sequence.
- **Consumer group**: inside a group, each partition is read by exactly **one** consumer → max useful consumers = number of partitions. Different groups each receive every message.
- **Broker**: a Kafka server. **Replication factor 3**: each partition has 1 leader + 2 followers; if leader dies, follower takes over. **ISR** = in-sync replicas.
- **Retention**: messages deleted by time/size, *not* when consumed.

## Delivery guarantees
| Setting | Effect |
|---|---|
| `acks=0` | fire and forget (may lose) |
| `acks=1` | leader wrote it |
| `acks=all` + `min.insync.replicas=2` | safest |
| `enable.idempotence=true` | producer retries won't create duplicates |
| Transactions | exactly-once across topics |
| Consumer commits offset **after** processing | at-least-once (duplicates possible → make consumer idempotent) |

## Consumer flow & rebalancing
```
poll() → get batch → process → commit offset
                          │
              consumer crashes before commit?
                          ▼
     Group rebalance: partition reassigned → new consumer restarts from
     last committed offset → some messages reprocessed (hence idempotency)
```
Failed message handling: retry topic → **Dead Letter Topic (DLT)**.

## Kafka vs RabbitMQ (your notes have a table; add this reasoning)
| | RabbitMQ | Kafka |
|---|---|---|
| Model | smart broker, dumb consumer (push) | dumb broker, smart consumer (pull) |
| After consume | message deleted | stays until retention |
| Replay | ✘ | ✔ |
| Throughput | 10k–100k/s | millions/s |
| Ordering | per queue | per partition |
| Best for | task queues, complex routing, low latency RPC-like | event streaming, logs, analytics, event sourcing |

## Spring Kafka
```java
@KafkaListener(topics = "orders", groupId = "billing")
public void handle(OrderEvent e) { ... }

kafkaTemplate.send("orders", order.getId(), event);   // key = orderId
```

---

## Spring Implementation in the Order Project

## Flow
```
Order Service ─send(topic "orders", key=orderId, value=event)─► Topic "orders"
        Partition 0 [e1][e4]…   Partition 1 [e2][e5]…   Partition 2 [e3][e6]…   (append-only log, kept N days)
                │                                 │
   Consumer group "analytics"        Consumer group "fraud"      (each group reads ALL events independently)
   C1←P0, C2←P1, C3←P2               C1←P0,P1,P2
   offset committed per group/partition  → after crash resume from last offset
```
Same key → same partition → ordering per order. Details are in the sections above.

## Spring Kafka code
```yaml
spring.kafka:
  bootstrap-servers: kafka:9092
  producer: { acks: all, properties: { enable.idempotence: true }, value-serializer: org.springframework.kafka.support.serializer.JsonSerializer }
  consumer: { group-id: analytics, auto-offset-reset: earliest, enable-auto-commit: false }
```
```java
// Producer
kafkaTemplate.send("orders", order.getId().toString(), new OrderPlaced(order.getId(), order.getTotal()))
    .whenComplete((res, ex) -> { if (ex != null) log.error("send failed", ex); });

// Consumer
@KafkaListener(topics = "orders", groupId = "analytics")
public void consume(OrderPlaced e, Acknowledgment ack) {
    analyticsService.record(e);       // must be idempotent
    ack.acknowledge();                // commit offset AFTER processing → at-least-once
}
// failures: DefaultErrorHandler + retries + DeadLetterPublishingRecoverer → "orders.DLT"
```

## RabbitMQ vs Kafka — which for which job?
```
"Do this job once by one worker" (email, PDF, image resize)   → RabbitMQ queue
"Something happened; many systems may care, maybe replay it"   → Kafka topic
```
| | RabbitMQ | Kafka |
|---|---|---|
| Message after consume | deleted | retained |
| Replay | no | yes |
| Routing | flexible exchanges | topics/partitions |
| Throughput | good | very high |
| Ordering | per queue | per partition |

## The outbox pattern (used with both)
Problem: DB commit succeeded but publish failed (or vice versa) → inconsistent.
```
@Transactional:  save order  +  INSERT into outbox(event_json, sent=false)   ← ONE DB transaction
Background poller / Debezium CDC: read unsent outbox rows → publish → mark sent
```
Consumers must still be idempotent (at-least-once).
