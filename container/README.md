# Containerization Plan

This document details the containerization strategy for the `retail-recs-service`.

## 1. Bake-vs-Mount Decision
* **Decision**: **Mount / Download Weights at Runtime** (Decoupled Strategy).
* **Justification**:
  - Our LightGBM ranking model is retrained daily to adapt to the latest retail shopping trends.
  - If we **baked** the model weights (`model.bst`, ~150 MB) directly into the image:
    - We would have to run a full container build and registry push every 24 hours.
    - Our Container Registry would accumulate GBs of redundant layers.
  - If we **mount / download** the weights:
    - The container image contains only python code and library dependencies (which change rarely).
    - When a pod starts, it checks the `MODEL_VERSION` environment variable and pulls the 150 MB binary file from the S3 bucket to `/app/models/model.bst` in memory or a fast local SSD volume.
    - Updating a model version is a simple Kubernetes configuration change or rolling restart.

## 2. Base Image Selection
* **Base Image**: `python:3.11-slim`
* **Why not Alpine?**
  - Scientific Python libraries like NumPy and machine learning libraries like LightGBM compile C/C++ code. Alpine uses `musl libc`, which causes compatibility and stability issues with these libraries unless you install heavy compatibility layers (`gcompat`), which defeats Alpine's size advantage.
* **Why not full Python/Ubuntu?**
  - The standard `python:3.11` image is around 1 GB because it contains compilers and headers that are unnecessary for running the application. Using the `slim` version keeps the footprint small.
* **Multi-Stage Build**:
  - We use a **builder stage** to install compilers (`build-essential`) and build the Python wheels.
  - We then copy only the compiled virtual environment `/opt/venv` to the **runner stage**, which does not have any compilers installed. This significantly reduces image size and improves security.

## 3. Image Size Estimate

Below is the size estimation breakdown of our final runtime image:

| Component / Layer | Estimated Size | Justification / Notes |
|---|---|---|
| `python:3.11-slim` Base | ~120 MB | Base Debian OS + Python runtime |
| `libgomp1` System Library | ~10 MB | Required by LightGBM to run multi-threaded CPU calculations |
| Python Virtual Env (`/opt/venv`) | ~90 MB | FastAPI, Uvicorn, LightGBM, NumPy, Feast SDK, Pydantic |
| Application Code (`/app`) | ~2 MB | Lightweight python service code |
| **Total Runner Image Size** | **~222 MB** | Small, fast to pull, and secure |

*(Compare this to a baked-in image which would be ~372 MB, or a non-multi-stage build containing compilers which would exceed 800 MB).*

## 4. Container Security Practices
* **Non-Root Execution**: We create a user `appuser` (UID 10001) and group `appgroup` (GID 10001) and switch to it using `USER appuser`. The container runs without root privileges, protecting the host system from potential container escapes.
* **No Shell Access in Production**: The system packages are minimal. Build tools are stripped out in Stage 1, leaving no compilers available to an attacker.
* **Health Check**: We implement a custom python-based health check that hits the `/health` endpoint to ensure the FastAPI app and model loading are active.
