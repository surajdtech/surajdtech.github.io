---
layout: single
title: "Designing a Dual-Path Event Capture Service on AWS (Kinesis + MSK) — Latency Benchmarks & Production Checklist"
subtitle: "Small messages to Kinesis; large to MSK Serverless. FastAPI gateway on EKS."
categories: [aws, streaming, kinesis, msk, eks, fastapi]
tags: [kinesis, kafka, msk, firehose, eks, lambda, s3, neo4j, flink, fastapi]
permalink: /aws/event-data-capture-service/

# REMOVE this block to stop overlaying the big image under the title
header:
  overlay_image: /assets/img/event-capture-hero.png
  overlay_filter: 0.25
  overlay_color: "#0F2A4D"   # pick any hex
  overlay_filter: 0          # keep text crisp
  # teaser: /assets/img/event-capture-hero.png

# Keep for social cards / previews only
# image: /assets/img/event-capture-hero.png

excerpt: >
  A practical blueprint for an ingestion service that sends small payloads to Kinesis and larger ones to MSK Serverless — fronted by FastAPI on EKS. Includes routing logic, latency methodology, results, and a production checklist.
---


Teams often want **one ingestion endpoint** that works for a variety of event sizes without forcing client changes. In this post I share a pattern that has worked repeatedly:
- Route **small messages** to **Kinesis** for simple scaling and cost efficiency.
- Route **large messages** to **MSK Serverless** (Kafka with IAM auth) for high throughput and relaxed size limits.
- Put a **FastAPI service on EKS** in front to apply config‑driven routing and emit latency headers for observability.

The goal is **clarity and repeatability**: one service, two output paths, measured performance.

---

## Architecture overview

Clients call a **FastAPI** service exposed via an **NLB**. The service reads a small YAML config and routes each request:

- If `large_message=false` (or below a size threshold) → **Kinesis** using `PutRecords` in batches.
- Otherwise → **Kafka** → **MSK Serverless** with SASL/OAUTHBEARER (IAM auth).

Downstream:
- Kinesis → **Firehose** → **S3** (Parquet/CSV), optional Athena table for ad‑hoc queries.
- Kafka → your preferred sink (connector, consumer app) — optionally **Neo4j** for graph use‑cases.

```yaml
# routing-config.yml
streams:
  - team: rpp
    app: promo-clicks
    kinesis_stream: rpp-promo-clicks-dev
    kafka_topic: rpp-promo-clicks-dev

routing:
  large_message_threshold_kb: 100
```

**Routing snippet (FastAPI):**
```python
# pseudo-code sketch
if payload_kb <= cfg.threshold_kb:
    kinesis.put_records(Records=batch, StreamName=cfg.kinesis)
else:
    kafka_producer.send(cfg.kafka_topic, value=payload)
```

> Tip: keep routing **purely config‑driven** so environments (dev/qa/prod) are just config swaps, not code changes.

![High-level architecture](/assets/img/event-capture-arch.png)

---

## Latency methodology

To understand the bottlenecks we measured three things per request:

1. **Network latency** — time from the client sending to reaching the service (reported back as header).
2. **Processing latency** — time spent by the service doing validation, batching, and the first enqueue call (also a header).
3. **End‑to‑end** — measured by the load generator from send → HTTP response.

**Headers exposed by the API** (example):
```
X-Network-Latency: 9ms
X-Processing-Latency: 14ms
```

**Load profile**
- Injectors: **2 × c6a.2xlarge EC2**
- Ramp: **2500 → 4500 RPS**
- Payload: **5 KB JSON**

---

## Results and observations

- The system stayed stable up to **~4.5K RPS** in our test bed.
- **Kinesis** behaved best with **batched** `PutRecords` (200–500 per call).
- **MSK Serverless** with IAM auth was reliable; re‑use tokens to avoid frequent refresh.
- With EKS, ensure an **HPA** is in place and watch CPU throttling under the NLB.
- Shard math matters — we kept **18 shards** for headroom during spikes.

---

## Production checklist

- **Networking**: Private subnets, **VPC endpoints** for Kinesis/STS.
- **Reliability**: Retries with backoff; DLQ strategy for both paths.
- **Observability**: Structured logs, request IDs, latency headers, dashboards.
- **Storage layout**: S3 partitioning by `date/hour/team/app`.
- **Cost hygiene**: Monitor shards, Firehose buffering, and EKS node sizing.

---

## Where to take it next

- Put **API Gateway + Lambda authorizer** ahead of NLB for auth & throttling.
- Add **Flink** (Kinesis/MSK) for streaming enrichments.
- Stream certain topics to **Neo4j** to add graph‑based features (journey, influence).

If you want the full repo with configs and scripts, ping me — I’ll publish it as a companion.

