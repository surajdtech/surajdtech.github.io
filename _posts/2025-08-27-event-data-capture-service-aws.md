---
layout: single
title: "Event Data Capture Service on AWS — From PoC to 4.5K RPS"
subtitle: "Kinesis for small payloads, MSK Serverless for large ones. Latency-tested, cost-aware, production-ready."
categories: [aws, streaming, kinesis, msk, eks, fastapi]
tags: [kinesis, kafka, msk, firehose, eks, lambda, s3, neo4j, flink, fastapi]
permalink: /aws/event-data-capture-service/
header:
  overlay_image: /assets/img/event-capture-hero.png
  overlay_filter: 0.25
  teaser: /assets/img/event-capture-hero.png
excerpt: |
  Architecture, code snippets, and real load-test results of a dual-path ingestion service: small messages → Kinesis; large messages → MSK Serverless. Includes EKS FastAPI gateway, IAM-auth Kafka producer, and Firehose to S3.
---

> **Why this exists**: Teams needed a single endpoint that smartly routes events without changing client code. We built a dual-path router with strong observability and ruthless focus on latency.

---

## TL;DR
- **Small payloads** → **Kinesis** (cheap, simple, fan-out to Firehose)
- **Large payloads** → **MSK Serverless** (Kafka + IAM auth, high throughput)
- **Gateway**: FastAPI on EKS, with config-driven routing and latency headers
- **Storage**: S3 via Firehose (Kinesis path) and Kafka Connect/Sink (optional)
- **Throughput** tested up to **4.5K RPS** with 5 KB messages

![High-level architecture](/assets/img/event-capture-arch.png)

---

## 1) Architecture (minimum viable)
- **Clients** → **FastAPI (EKS, NLB)** → Router:
  - if `large_message=false` → Kinesis PutRecords (batched)
  - else → Kafka producer (MSK Serverless, SASL/OAUTHBEARER IAM)
- **Downstream**:  
  - Kinesis → Firehose → S3 (parquet/csv), optional Athena table  
  - Kafka → Sink connector or Neo4j ingestion

**Config example (YAML):**
```yaml
streams:
  - team: rpp
    app: promo-clicks
    kinesis_stream: rpp-promo-clicks-dev
    kafka_topic: rpp-promo-clicks-dev

routing:
  large_message_threshold_kb: 100
```

**Routing snippet (Python/FastAPI):**
```python
if payload_kb <= cfg.threshold_kb:
    kinesis.put_records(Records=batch, StreamName=cfg.kinesis)
else:
    kafka_producer.send(cfg.kafka_topic, value=payload)
```

---

## 2) Latency testing setup
- **Load injectors**: 2× c6a.2xlarge EC2, Python async client  
- **RPS ramp**: 2500 → 4500  
- **Payload**: 5 KB JSON  
- **Metrics**: network_latency, processing_latency (returned in headers), end-to-end  

**Headers from API:**
```
X-Network-Latency: 9ms
X-Processing-Latency: 14ms
```

---

## 3) Results & learnings
- Stable up to **~4.5K RPS** on our test bed
- Keep **18 shards** for headroom on Kinesis; shard math matters
- Kafka with **IAM auth** works well; cache tokens
- Enable **HPA** on EKS; watch CPU throttling
- Batch Kinesis puts (200–500 records) for best $$

---

## 4) Production checklist
- ✅ VPC endpoints for Kinesis/STS  
- ✅ Retry & DLQ strategy  
- ✅ Structured logs + request IDs  
- ✅ S3 partitioning (date/hour/team/app)  
- ✅ Cost dashboards (shards, Firehose, EKS nodes)  

---

## 5) What’s next
- Add **API Gateway + Lambda authorizer** in front of NLB  
- Plug **Flink on Kinesis/MSK** for streaming enrichments  
- Stream to **Neo4j** to unlock graph features (journey, influence)  

---

*Repo link and code snippets are coming in a follow-up post.*
