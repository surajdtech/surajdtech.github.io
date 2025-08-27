---
layout: single
title: "Projects & Pipelines"
permalink: /projects/
---

Below are representative **pipelines and architectures** I’ve built or led. Most have companion posts with code/configs.

## 1) Dual‑Path Event Capture Service (AWS)
**Stack:** FastAPI (EKS) · NLB · Kinesis Data Streams + Firehose · MSK Serverless · S3 · Databricks  
**Idea:** One ingestion endpoint → **small** messages to **Kinesis**, **large** to **Kafka/MSK**.  
**Why:** Simple client integration, elastic scaling, cheap fan‑out to S3/DBR.  
**Post:** [/aws/event-data-capture-service/](/aws/event-data-capture-service/)

## 2) Real‑time Customer‑Journey Graph (Flink → Neo4j)
**Stack:** Kinesis/MSK → **Apache Flink** (PyFlink/Scala) → **Neo4j** · Databricks for offline features.  
**Idea:** Stream events into a **journey graph** (users, sessions, actions) for anomaly detection, recommendations, and support insights.  
**Post:** [/graphs/realtime-journey-neo4j-flink/](/graphs/realtime-journey-neo4j-flink/)

## 3) Databricks Medallion Migration
**Stack:** RDBMS → Hadoop → **AWS + Databricks** (Bronze/Silver/Gold), Auto Loader, UC, Delta Live Tables.  
**Idea:** Consolidate legacy ETL into medallion layers; **reduce cost** and **improve SLAs** via tuning, autoscaling, and notebook orchestration.  
**Post:** (coming soon)

## 4) AdTech Ingestion & Analytics
**Stack:** Lambda · Step Functions · SQS/SNS · EMR/Glue · S3 · Elastic/Kibana · MEMSQL.  
**Idea:** Multi‑source ad impressions & revenue events with near‑real‑time dashboards and robust IaC.

If you want a specific deep‑dive first, ping me. I’ll publish it next.
