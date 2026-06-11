# Model Rollback Runbook

This document is a step-by-step guide for an on-call engineer to roll back the `retail-recs-service` to a previous stable model version during an incident.

## 1. Triggers for Rollback

Execute this runbook if any of the following conditions are met:
- [ ] **High Latency Alert Firing**: `RecommendationHighLatency` alert is firing (p95 latency > 120 ms).
- [ ] **High Error Rate Alert Firing**: `RecommendationHighErrorRate` alert is firing (error rate > 0.1%).
- [ ] **Bad Product Recommendation Quality**: Product/marketing teams report that recommendations are broken (e.g., empty lists, unrelated items, or cold-start fallback is running for 100% of users).
- [ ] **A/B Test Metric Drop**: Live BI dashboard shows a catastrophic drop in click-through rates (CTR) or conversions (> 30% drop) after a model release.

---

## 2. Rollback Action Checklist

### Step 1: Identify the Target Stable Version
Check the model registry or GitHub releases to find the version ID of the last known stable deployment (e.g., `v1.1.0` if `v1.2.0` is currently failing).

### Step 2: Roll Back the Kubernetes Deployment
We deploy via environment variables that point to the model registry path. Update the deployment image or the environment variable to target the previous version.

*To roll back to the previous deployment configuration instantly (Kubernetes native rollback):*
```bash
# Roll back the deployment to the previous revision
kubectl rollout undo deployment/retail-recs-serving -n production
```

*Or, if you need to point specifically to a known stable Docker tag:*
```bash
# Set the container image to the target stable tag (e.g., v1.1.0)
kubectl set image deployment/retail-recs-serving serving-container=123456789012.dkr.ecr.us-east-1.amazonaws.com/retail-recs-service:v1.1.0 -n production
```

### Step 3: Monitor the Rollout Status
Track the progress of the container replacement to ensure the rollback completes successfully.
```bash
kubectl rollout status deployment/retail-recs-serving -n production
```

### Step 4: Verify the Rollback
1. **Health Check**: Run a curl request to verify the container has started and loaded the model successfully.
   ```bash
   curl -i http://retail-recs-service.production.local/health
   ```
   *Expected Response:*
   ```json
   {
     "status": "healthy",
     "model_loaded": true,
     "feature_store_connected": true
   }
   ```
2. **Observe Response Headers**: Query the recommendation endpoint and check that the model version header (`X-Model-Version`) matches the target rolled-back version.
   ```bash
   curl -i -X POST http://retail-recs-service.production.local/recommend \
     -H "Content-Type: application/json" \
     -H "X-Correlation-ID: 00000000-0000-0000-0000-000000000000" \
     -d '{"user_id": "usr_test", "limit": 1}'
   ```
   *Verify that the HTTP response includes:*
   `X-Model-Version: v1.1.0` (or your chosen rollback version).

3. **Check Prometheus Dashboard**: Confirm that the latency graphs fall back below 120 ms and error rate returns to 0%.

---

## 3. Post-Incident Follow-Up

- [ ] **Mark Model as Deprecated**: Log into the MLflow Model Registry and change the status of the buggy model version to `Archived` or add the tag `status: failed-prod` to prevent it from being deployed again.
- [ ] **Notify the Team**: Send a message in the `#ml-alerts` Slack/Teams channel detailing what was rolled back, why, and the current system status.
- [ ] **Open Post-Mortem**: Document the incident, root cause, and recovery steps.
