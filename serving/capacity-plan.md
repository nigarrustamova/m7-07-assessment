# Capacity Plan

This document outlines the resource requirements, replica sizing, and cost estimations to serve our personalization model at a peak of 800 RPS with a p95 latency budget of 120 ms.

## 1. Resource Footprint (Per Replica)

* **Model size**: 150 MB (LightGBM ranker).
* **Memory footprint**:
  - Model weights: 150 MB
  - Python runtime + FastAPI + libraries: 250 MB
  - Operating system overhead: 100 MB
  - Working memory / buffers: 100 MB
  - **Total RAM required per replica**: ~600 MB
* **CPU requirements**:
  - Model inference takes ~20 ms of single-threaded CPU time.
  - Fetching user features from Redis takes ~5 ms.
  - Parsing and framework overhead takes ~5 ms.
  - **Total processing time**: ~30 ms of CPU time per request.

## 2. Throughput & Replica Math

* **Peak Load**: 800 RPS.
* **Target Instance**: AWS `c6i.xlarge` (Compute Optimized, 4 vCPUs, 8 GB RAM).
* **Throughput per Core**: 
  - With a 30 ms execution time, a single CPU core can handle:
    `1 second / 0.030 seconds = 33.3 requests per second` (at 100% utilization).
* **Throughput per Instance**:
  - A `c6i.xlarge` has 4 vCPUs. Running 4 Uvicorn workers, it can handle:
    `4 cores * 33.3 RPS = 133.3 RPS` (at 100% utilization).
* **Target Utilization**:
  - To prevent queueing and handle traffic bursts, we target a maximum of **60% CPU utilization** at peak.
  - Safe throughput per instance: `133.3 RPS * 0.60 = 80 RPS`.
* **Number of Replicas Required**:
  - To serve 800 RPS: `800 RPS / 80 RPS per instance = 10 replicas`.
* **Redundancy (High Availability)**:
  - We apply `N+2` redundancy so that if 2 availability zones go down or 2 nodes fail during peak, we still have enough capacity.
  - **Final Replica Count**: `10 + 2 = 12 replicas`.

## 3. Online Feature Store Sizing (Redis)

* **Active User Base**: 10 million active users.
* **Data Size per User**:
  - Features stored: `user_id` (string), `last_30d_clicks` (int), `last_30d_purchases` (int), `category_preference` (string).
  - Estimated size per user record (including Redis metadata overhead): ~300 bytes.
* **Total Memory Required**:
  - `10,000,000 users * 300 bytes = 3,000,000,000 bytes` (~3 GB).
* **Instance Choice**:
  - AWS ElastiCache Redis `cache.m6g.large` (2 vCPUs, 6.38 GB RAM).
  - We configure a **Multi-AZ Replication Group** with 1 Primary node and 1 Replica node for high availability.

## 4. Monthly Cost Estimate

| Component | AWS Resource | Unit Cost | Quantity | Monthly Cost |
|---|---|---|---|---|
| Model Serving (compute) | `c6i.xlarge` (4 vCPUs, 8 GB RAM) | $0.17 / hr | 12 instances | $1,468.80 |
| Feature Store (Redis) | `cache.m6g.large` (2 vCPUs, 6.38 GB RAM) | $0.136 / hr | 2 instances | $195.84 |
| Load Balancer | AWS ALB | ~$0.0225 / hr + LCU | 1 unit | $40.00 |
| **Total Estimated Cost** | | | | **~$1,704.64 / month** |
