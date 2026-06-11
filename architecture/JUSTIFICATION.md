# Architectural Justifications & Trade-Offs

We chose the architecture described in `architecture.md` to meet the specific requirements of Scenario X (B2C Personalized Recommendations). Below are the key justifications and trade-offs.

## 1. Online Inference vs. Batch Pre-computation
* **Choice**: Online inference with real-time feature retrieval.
* **Why**: The mobile app needs recommendations rendered on every home-screen load, and they must incorporate the user's last 30 days of browsing and purchase behavior. Since user behavior changes dynamically (e.g., they bought a shirt 5 minutes ago and we shouldn't recommend it again), pre-computing recommendations in batch is not dynamic enough.
* **Trade-off**: Online serving increases latency risk. We must meet a p95 latency budget of 120 ms end-to-end. To mitigate this, we keep the model small and use an in-memory database (Redis) for feature retrieval.

## 2. Feature Store: Redis-backed Feast
* **Choice**: Redis as the online feature store, orchestrated by Feast.
* **Why**: Getting a user's 30-day history must be extremely fast. Redis is an in-memory key-value store that provides sub-5ms lookups for feature vectors.
* **Trade-off**: Redis holds all active user feature profiles in RAM, which increases infrastructure costs compared to disk-based databases. However, for 800 RPS and a tight latency budget, in-memory lookups are mandatory.

## 3. Model Choice: LightGBM Ranker (150 MB)
* **Choice**: A tree-based ranking model (LightGBM) over deep learning models (like DLRM).
* **Why**: 
  - LightGBM models are compact (150 MB) and run extremely fast on standard CPU instances (typically under 20ms for a batch of candidate items).
  - They do not require expensive GPU hardware for inference, which keeps operational costs low at 800 RPS.
* **Trade-off**: A deep learning model might capture slightly more complex interactions but would require GPUs and introduce significant latency, making it harder to stay under the 120ms budget.

## 4. Cold-Start and Reliability Fallback
* **Choice**: Two-tier fallback logic.
  - If a user is new (cold-start) and has no history in Redis, the service bypasses the LightGBM model and queries a pre-cached Redis list of "popular and trending items" (partitioned by category/region).
  - If the main service or feature store fails entirely, the API Gateway directly redirects traffic to a static fallback cache.
* **Why**: This guarantees high availability. It is better to show popular items quickly than to return an error or timeout.

## 5. Model Weights: Mount vs. Bake
* **Choice**: Mount model weights via an external volume/object store cache instead of baking them into the Docker image.
* **Why**: The product recommendation model will be retrained daily or weekly to capture new shopping trends. If we baked the weights into the container, we would have to rebuild, scan, and deploy a new Docker image daily. By mounting the weights, the CI/CD pipeline only builds the container when serving code changes, and a lightweight sync process updates the active model weights file in-place.
* **Trade-off**: Storing weights externally means the container must fetch them at startup, which slightly increases initial container startup/warm-up time. We use local caching on the host directory to solve this.

## 6. A/B Testing Pattern
* **Choice**: Gateway routing (Envoy/Service Mesh) instead of application-level routing.
* **Why**: Separates the routing logic from the model execution code. The API Gateway inspects a cookie or user ID hash, appends the target version header (`X-Model-Version`), and forwards the request to the matching Kubernetes deployment.
* **Trade-off**: Requires more complex networking configuration at the gateway level, but keeps the FastAPI application clean and easy to test.
