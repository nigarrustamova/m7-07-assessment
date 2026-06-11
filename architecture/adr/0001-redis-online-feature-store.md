# ADR 0001: Use Redis-backed Feast as the Online Feature Store

## Status
Accepted

## Context
Our B2C retail recommendation service must serve recommendations on every home-screen load, with a peak load of 800 RPS and a strict p95 latency budget of 120 ms end-to-end. To generate personalized recommendations, the ranking model requires real-time user behavior features (such as user click counts, browse history categories, and purchase counts over the last 30 days). 

Querying these raw events directly from the transactional database during a request is too slow (taking 150ms+), which violates the latency budget. We need a way to pre-compute and store these features offline and fetch them in real-time with sub-10ms latency.

## Decision
We will implement an online feature store using **Feast** with **Redis** as the online database provider.
* **Feast** acts as the registry and API layer to define and fetch features consistently between training and serving.
* **Redis** is used as the low-latency key-value store to serve features at inference time.
* An offline Spark job runs daily to compute features from raw logs and load ("materialize") them into the Redis cluster.

## Consequences
* **Positive**:
  - Feature lookups take less than 5 ms, giving the LightGBM model plenty of time to run inference within the 120 ms budget.
  - Consistent feature definitions prevent training-serving skew.
* **Negative**:
  - Redis holds feature data in-memory, which increases cloud infrastructure costs.
  - We must maintain a feature ingestion pipeline (Spark/Feast materialization job) to update Redis daily.
  - If a user is not found in Redis, the application must handle the cold-start case gracefully.
