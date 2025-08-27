---
layout: single
title: "Real-time Customer-Journey Graph with Flink + Neo4j (Kinesis/MSK → Events → Graph)"
categories: [graphs, streaming, neo4j, flink, aws]
tags: [neo4j, flink, kinesis, msk, databricks, real-time, graph]
permalink: /graphs/realtime-journey-neo4j-flink/

# ✅ Color-only hero (no image)
header:
  overlay_color: "#0E7490"   # teal (Neo4j vibe)
  overlay_filter: 0

# (optional) social preview only:
# image: /assets/img/neo4j-flink-hero.png

excerpt: >
  Build a streaming journey graph: ingest clickstream/events from Kinesis/MSK, process with Flink, and upsert to Neo4j for real-time recommendations and anomaly detection. Includes Cypher and Flink code sketches.
---


## Why a journey graph?

Tabular aggregates hide **paths**. A graph makes user **sequences and relationships** first‑class: `User → Session → Action → Product`. With a streaming writer, the graph is **always fresh** for features and decisions.

---

## Architecture

```
Clients → Kinesis/MSK → Flink job(s) → Neo4j (Bolt/HTTP)
                         ↘ S3 (raw)  ↘ Databricks (batch features)
```

- **Ingress:** Kinesis/MSK; partitions by `tenantId, date` for parallelism.
- **Processing:** Flink keyed streams, out‑of‑order handling, small state (RocksDB), optional dedupe.
- **Sink 1:** Neo4j (transactions batched per key), idempotent **MERGE** UPSERTs.
- **Sink 2:** S3/Delta for offline features and reprocessing.

---

## Event schema (example)

```json
{
  "tenantId": "acme",
  "userId": "u-123",
  "sessionId": "s-456",
  "ts": "2025-08-27T12:00:01Z",
  "action": "view",
  "itemId": "sku-777",
  "attrs": {"ref": "email", "campaign": "diwali"}
}
```

---

## Neo4j modeling

```cypher
// Nodes
MERGE (u:User {id:$userId, tenant:$tenantId})
MERGE (s:Session {id:$sessionId, tenant:$tenantId})
MERGE (a:Action {id:$eventId, type:$action, ts:$ts})

// Relationships
MERGE (u)-[:HAS_SESSION]->(s)
MERGE (s)-[:HAS_ACTION]->(a)
FOREACH (_ IN CASE WHEN $itemId IS NULL THEN [] ELSE [1] END |
  MERGE (p:Product {id:$itemId})
  MERGE (a)-[:ON_PRODUCT]->(p)
)
```

Best practices:
- Keep **tenant** as a label/property for multi‑tenant isolation.
- Use **idempotent MERGE** with **composite keys** (`tenantId+id`).
- Batch by `tenantId` to reduce contention.

---

## Flink writer sketch (Scala‑like pseudocode)

```scala
case class E(tenantId:String, userId:String, sessionId:String, ts:Long, action:String, itemId:Option[String], attrs:Map[String,String])

val stream = env
  .addSource(kinesisOrKafka(...))
  .map(parseJsonToE)
  .keyBy(e => (e.tenantId, e.userId, e.sessionId))
  .process(new RichKeyedProcessFunction[...] {
     override def processElement(e: E, ctx: Context, out: Collector[CypherOp]): Unit = {
       // emit a small list of Cypher statements + params
     }
  })

stream.addSink(neo4jSink(batchSize=200, maxInFlight=4))
```

Tuning notes:
- Scale by partitions; keep batches **\< 1s** for low write latency.
- Use connection pools; retry on transient Bolt exceptions.
- For replays, read from S3/Delta and re‑emit to the Neo4j sink.

---

## Observability

- Emit per‑tenant **lag** and **write success rate** metrics.
- Log sample Cypher per error class; add DLQ by event hash to S3.
- Keep a “**reconciliation**” job that reads Neo4j and S3 to spot drift.

---

## What this unlocks

- **On‑the‑fly recommendations** (similar products, people also viewed).  
- **Anomaly detection** (sudden path changes, high drop‑off steps).  
- **Support insights** (what users did just before raising a ticket).

---

If you want the production Terraform + configs as a repo, I’ll post it next.
