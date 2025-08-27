---
layout: single
title: "Agentic AI for Data Pipelines — Build a Self-Serve SRE with MCP, Prometheus & Databricks"
subtitle: "An AI that watches lag, skew, and failures; proposes fixes with reasoning; and can safely execute approved changes."
categories: [ai, ops, databricks, aws]
tags: [mcp, agent, fastapi, prometheus, databricks, spark, kinesis, msk]
permalink: /ai/agentic-pipeline-sre/
header:
  overlay_color: "#7C3AED"
  overlay_filter: 0
excerpt: >
  Turn your pipeline runbooks into actions an agent can execute. This guide shows how to wire an MCP server that reads Prometheus,
  reasons about incidents (Spark skew, MSK lag, OOMs), proposes fixes with diffs and commands, and—behind approvals—executes changes.
---

## TL;DR

- Use **MCP (Model Context Protocol)** to expose safe tools the agent can call.
- Teach the agent **your runbooks** (Spark skew, MSK lag, Databricks OOM, etc.).
- Wire to **Prometheus** for signals, **Databricks Jobs** for actions, and **GitHub** for change control.
- Start **read-only**, then move to **approval-gated** writes.

---

## 1) Architecture (ASCII)

```
+------------------+         scrape          +-----------------+
| Pipelines (ETL)  |  ---->  metrics  ---->  |  Prometheus     |
| Spark, DBR Jobs  |                          +--------+--------+
+---------+--------+                                   |
          | webhooks                                   | API (read)
          v                                            v
+---------+------------------+                +-----------------------+
| Events (MSK/Kinesis)       |                |  MCP Agent (FastAPI)  |
| Lag, errors, skew hints    |                |  Tools:               |
+----------------------------+                |   - get_metric()      |
                                              |   - get_job_status()  |
                                              |   - suggest_conf()    |
                                              |   - rollout_config()  |
                                              +----+--------------+---+
                                                   |              |
                                       approval -> |              | <- read-only mode
                                                   v              v
                                      +-----------------+   +-------------------+
                                      | Databricks Jobs |   | GitHub (IaC/Conf) |
                                      +-----------------+   +-------------------+
```

**Flow:** metrics & events → agent reasons with your playbooks → proposes a fix → waits for approval → executes (job rerun, config change PR, or scaling action) → posts result.

---

## 2) MCP tool surface (what the agent is *allowed* to do)

Create a small FastAPI server that the model calls through MCP. Tools:

- `get_metric(query, window)`: read PromQL.
- `get_job_status(job_id)`: Databricks Jobs API.
- `suggest_conf(problem, plan_context)`: return a *plan* (Spark conf deltas, commands).
- `rollout_config(change, target)`: open a PR or call a job with parameters (approval-gated).

### Tool schema (YAML)

```yaml
tools:
  - name: get_metric
    description: Query Prometheus for a scalar or series.
    args:
      query: string
      window: string
  - name: get_job_status
    description: Get last run result and logs pointer for a Databricks job.
    args:
      job_id: int
  - name: suggest_conf
    description: Given a symptom, propose Spark/Kafka tuning with reasoning.
    args:
      problem: string
      plan_context: string
  - name: rollout_config
    description: Create a GitHub PR or trigger a Databricks job with new params.
    args:
      change: string
      target: string
    requires_approval: true
```

### Minimal FastAPI skeleton

```python
from fastapi import FastAPI
import requests

app = FastAPI()

PROM = "http://prometheus:9090"

@app.post("/get_metric")
def get_metric(query: str, window: str = "5m"):
    r = requests.get(f"{PROM}/api/v1/query", params={"query": f"{query}[{window}]"})
    return r.json()

@app.post("/get_job_status")
def get_job_status(job_id: int):
    # call Databricks jobs API (token from env)
    # return: life_cycle_state, result_state, error, run_page_url
    ...

@app.post("/suggest_conf")
def suggest_conf(problem: str, plan_context: str = ""):
    # Very light rule layer; the LLM fills in details.
    library = {
      "spark_skew": {
        "why": "Long-tail tasks and large shuffle read on a few partitions.",
        "fix": [
          'spark.sql.adaptive.enabled=true',
          'spark.sql.adaptive.skewJoin.enabled=true',
          'spark.sql.autoBroadcastJoinThreshold=100MB',
          'spark.sql.shuffle.partitions=<2-3x cores>'
        ],
        "actions": ["repartition()", "broadcast()", "salt_hot_keys()"]
      },
      "msk_lag": {
        "why": "Consumers behind; partitions uneven.",
        "fix": ["increase consumer concurrency", "raise max.poll.interval.ms", "rebalance group", "check partition skew"]
      }
    }
    return library.get(problem, {"why": "unknown", "fix": [], "actions": []})

@app.post("/rollout_config")
def rollout_config(change: str, target: str):
    # Normally: open a PR with change or call DBR job with params.
    # This endpoint should be approval-gated outside the agent.
    return {"submitted": True, "target": target, "diff": change}
```

---

## 3) The agent’s system prompt (drop into your orchestrator)

```
You are a Pipeline SRE Assistant.
- Read metrics with get_metric() and job states with get_job_status().
- Never change anything without first proposing a PLAN with reasoning and diffs.
- Prefer the smallest reversible fix first (AQE on, broadcast, repartition).
- When confident, return an APPROVAL REQUEST with:
  * impact, risk, rollback, commands/PR diff, and verify steps.
- After approval, call rollout_config() and report result with links.
```

---

## 4) Incidents the agent can handle on day 1

### 4.1 Spark stage skew (from the Spark UI post)

**Signal:** p95 task duration ≫ p50, few tasks hold the stage; shuffle read spike.

**Plan:**
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 104857600)  # 100MB
spark.conf.set("spark.sql.shuffle.partitions", "2-3x cores")
# Optionally:
from pyspark.sql.functions import broadcast
df = big.join(broadcast(dim_small), "k", "left")
```

**Verification:** reduced tail (>50% drop in max task duration), lower spilled bytes, stable GC.

---

### 4.2 Databricks job OOM

**Signal:** run failure; executor OOM; GC time > 40% CPU.

**Plan:** lower partition size, enable AQE, add `--conf spark.memory.fraction=0.6`, scale workers by +1, retry once.

**Rollout:** trigger the same job with new params; or open a PR to the job JSON.

---

### 4.3 MSK/Kinesis consumer lag

**Signal:** `sum(kafka_consumer_lag{group="etl"})` rising > threshold.

**Plan:** increase parallelism, check hot partitions, ensure `fetch.max.bytes` and `max.poll.interval.ms` fit batch time; rebalance group.

**Verification:** lag returning to baseline; no commit failures.

---

## 5) Observability & guardrails

- **Metrics:** incident count by type, MTTR, fix success rate, “plan only” vs “executed”.
- **Approvals:** Slack/GitHub comment approval required for `rollout_config`.
- **Change windows:** deny writes outside allowed windows.
- **Dry-run mode:** default on; include exact commands/PR diff.

---

## 6) Implement in a day (starter checklist)

1. Expose Prometheus and Databricks tools via FastAPI (as above).  
2. Add MCP adapter so your LLM can call tools.  
3. Seed runbooks (this page + your own) in the prompt context.  
4. Start read-only: agent posts PLAN + DIFF to Slack/GitHub.  
5. After a week of shadowing, enable approval-gated `rollout_config`.  

---

## Appendix — sample PromQL

```promql
# Kafka consumer lag (per group)
sum(kafka_consumergroup_lag{consumergroup="etl"}) by (topic)

# Spark executor GC ratio
sum(rate(spark_executor_gc_time_seconds_total[5m])) / sum(rate(spark_executor_cpu_time_seconds_total[5m]))
```

If this saves you one on-call night, it paid for itself. Next up, we’ll package this as a template repo with Terraform & GitHub Actions so you can clone-and-go.
