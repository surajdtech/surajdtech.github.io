---
layout: single
title: "Reading the Spark UI Like a Pro — Stages → Tasks → DAG (and How to Spot Skew)"
subtitle: "Read in the only order that matters during incidents."
categories: [spark, performance, databricks]
tags: [spark, spark-ui, tuning, skew, shuffle, databricks, pyspark]
permalink: /spark/spark-ui-deep-dive/

header:
  overlay_color: "#9D174D"   # magenta (Spark vibe)
  overlay_filter: 0

# image: /assets/img/spark-ui-hero.png  # optional: keep only for social cards

excerpt: >
  Stop guessing. This post walks the Spark UI in the order you should read it during incidents: Jobs → Stages → Tasks → SQL/DAG → Storage → Executors → Environment. We diagnose a real skew issue and fix it with partitioning, AQE, and salt keys.
---


Performance problems in Spark rarely hide. They leave fingerprints all over the **Spark UI** — you just need to read them **in the right order**.

This is the guide I use with teams during on-calls and postmortems. Grab your screenshots and follow along.

---

## 1) Quick mental model

- A **Job** is triggered by an action (`.count()`, `.collect()`, `save`...).  
- Each job breaks into **Stages** separated by **shuffles**.  
- Each stage runs many **Tasks** (one per partition).  
- The **SQL/DAG** tab tells you *why* a stage exists.  
- **Storage** reveals cached data that might be old or too big.  
- **Executors** tells you if you’re starved on CPU/memory or spending life in GC.  
- **Environment** confirms configs actually applied.

> Golden rule: **Read top→down**: Jobs → Stages → Tasks → SQL/DAG → Storage → Executors → Environment.

![Spark UI Overview](/assets/img/spark-ui-overview.png)

---

## 2) Jobs tab — find the pain fast

Look for jobs with high **Duration** and **Input/Shuffle sizes**. Expand the job to see child stages.

Checklist:
- Unusual **Scheduler Delay**? (cluster busy / undersized)
- **Result Serialization** high? (driver bottleneck / huge collect)
- Is there a **single long stage**? → likely skew or slow I/O.

---

## 3) Stages tab — the shuffle detector

Open the worst stage. Key metrics to scan:

- **Tasks:** if most tasks finish in seconds but a few run for minutes → **skew**.  
- **Shuffle Read/Write:** large numbers → data exchange across nodes; check **Spill**.  
- **Bytes Spilled (memory/disk):** frequent spill = insufficient memory or huge partitions.  
- **Peak Execution Memory:** close to cap → GC pressure likely.

> Screenshot anchor: look for that **“long-tail”** task timeline — a handful of tasks far to the right.

---

## 4) Tasks detail — confirm skew

Click **Summary Metrics for Completed Tasks**.

- **Task duration**: min vs 75th vs max → a big gap = skew.  
- **Shuffle Read (records)**: do a few tasks read millions while others read thousands?  
- **GC Time** & **Input Size** spikes on a small number of tasks? Another skew tell.

> One slow partition can hold the entire stage hostage.

---

## 5) SQL/DAG — understand *why* the shuffle happened

Go to the **SQL/DAG** tab and select the problematic SQL. Expand **Details** to see **Exchange** nodes (shuffles) and their parents.

Red flags:
- `Exchange hashpartitioning(col, N)` on low‑cardinality columns.  
- `SortMergeJoin` with mismatched partitioning.  
- Multiple `Exchange` between `Filter` and `Join` (predicate pushdown missing).

Add a **physical plan** print during dev:
```python
df.explain(True)   # or spark.sql("...").explain(True)
```

---

## 6) Case study — skew in a join

**Symptom:** A stage with 2,000 tasks; 1,980 finish in 5–10s, 20 tasks run for 8–10 minutes.

**Cause:** Joining a huge `events` table with a **hot key** in `users` (few userIds dominate). AQE is off.

**Fixes (apply in this order):**

### 6.1 Enable AQE (Adaptive Query Execution)
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.shuffle.targetPostShuffleInputSize", "134217728")  # 128MB
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```
- AQE coalesces small partitions and **splits skewed** ones automatically.

### 6.2 Broadcast small side
```python
from pyspark.sql.functions import broadcast
df = events.join(broadcast(users_small), "userId", "left")
```
- Use when one side < 10–50 MB (check `spark.sql.autoBroadcastJoinThreshold`).

### 6.3 Salt the hot key (last resort, but precise)
```python
from pyspark.sql import functions as F

# 1) Add salt on the skewed side
hot_keys = ["u123", "u456"]
users_salted = (users
  .withColumn("salt", F.when(F.col("userId").isin(hot_keys), F.rand()*10).otherwise(F.lit(0)).cast("int"))
  .withColumn("userId_salted", F.concat_ws("#", "userId", F.col("salt"))))

# 2) Replicate events rows for the hot keys (fan-out)
events_salted = (events
  .withColumn("salt", F.when(F.col("userId").isin(hot_keys), F.sequence(F.lit(0), F.lit(9))).otherwise(F.array(F.lit(0))))
  .withColumn("salt", F.explode("salt"))
  .withColumn("userId_salted", F.concat_ws("#", "userId", F.col("salt"))))

# 3) Join on salted key
joined = (events_salted
  .join(users_salted.select("userId_salted", "attr1", "attr2"), "userId_salted", "left"))

# 4) Optional: drop salt post-join
result = joined.drop("salt", "userId_salted")
```
- You’re **splitting** the hot key across 10 buckets so tasks distribute evenly.

### 6.4 Partition consciously
```python
# For large writes
df.repartition(200, "date")   .write.partitionBy("date").mode("overwrite").parquet(path)
```
- Tie `repartition(N)` to cluster cores (2–3× cores is a good start).

---

## 7) Storage tab — cache hygiene

- Check **Size in Memory** and **Size on Disk**. Old caches waste RAM.  
- If the cached RDD keeps spilling, **unpersist** or cache post‑filter.
```python
df_cached = df.filter("eventDate >= current_date() - 7").cache()
df_cached.count()
# Later
df_cached.unpersist()
```

---

## 8) Executors tab — CPU, memory, GC

- **GC Time** high → memory pressure; reduce partition size, review UDFs.  
- **Task Time** uneven across executors → data skew or resource imbalance.  
- **Failed Tasks** hotspots → check logs; might be OOM on one executor.

---

## 9) Environment — trust but verify

Look for the exact values you set:
- `spark.sql.shuffle.partitions`
- `spark.sql.adaptive.enabled`
- `spark.sql.autoBroadcastJoinThreshold`
- `spark.executor.memory`, `spark.executor.cores`, `spark.memory.fraction`

If they’re not there, the cluster/job didn’t pick them up.

---

## 10) 10‑minute incident checklist

1. Jobs: find the slow job/stage.  
2. Stages: confirm long tail.  
3. Tasks: quantify skew (max vs p75).  
4. SQL/DAG: identify `Exchange` and hot keys.  
5. Flip **AQE** on; re‑run.  
6. Consider **broadcast join**.  
7. If still skewed, **salt** hot keys.  
8. Right‑size `shuffle.partitions` (start 2–3× cores).  
9. Check **Executors** for GC and failures.  
10. Clean **Storage** caches.

---

## Appendix — handy defaults

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 104857600)  # 100MB
spark.conf.set("spark.sql.shuffle.partitions", "400")  # tune per cluster
```

```sql
-- Spot skewy keys
SELECT userId, COUNT(*) c FROM events
GROUP BY userId ORDER BY c DESC LIMIT 50;
```
