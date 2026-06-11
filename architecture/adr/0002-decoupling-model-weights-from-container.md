# ADR 0002: Decouple Model Weights from the Container Image

## Status
Accepted

## Context
Our personalization model (LightGBM Ranker, ~150 MB) must be updated frequently (daily or weekly) to capture shifting retail trends and user purchasing behaviors. 

If we bake the model weights directly into the Docker container image:
1. Every daily model update would require running a CI/CD build, container security scan, and pushing a new ~400 MB image to our Container Registry.
2. Pushing new images daily consumes bandwidth, registry storage, and slows down deployments.
3. Rollbacks to previous model versions would require redeploying the entire container stack rather than just pointing to a different weights file.

## Decision
We will separate the model serving code from the model weights.
* The Docker container image will contain only the FastAPI code, dependencies, and shell scripts. It will be built and pushed only when code changes.
* Model weights will be saved to an external Cloud Object Storage bucket (e.g., AWS S3 or Google Cloud Storage) under `/models/<model_version>/model.bst`.
* At startup, the container reads the target model version from an environment variable (`MODEL_VERSION`), pulls the model file from the object store, and caches it locally in memory.
* If a model update is triggered, we can perform a rolling update of the Kubernetes pods with an updated `MODEL_VERSION` environment variable, or use an API endpoint to tell the running container to reload the weights.

## Consequences
* **Positive**:
  - Image size remains small (~150 MB instead of 300 MB+).
  - Pushing a new model update is instant (simply uploading a 150 MB file to S3 and updating a deployment variable) without rebuilding Docker images.
  - Faster CI/CD pipelines since code-only changes are decoupled from model training runs.
* **Negative**:
  - The container has a runtime dependency on the Object Storage bucket at startup. If the bucket is down, the container cannot start (we mitigate this by using local caching on the host node).
  - The pod startup takes an extra 2-3 seconds to download the model file.
