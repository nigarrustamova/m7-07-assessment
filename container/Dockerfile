# --- Stage 1: Build Stage ---
FROM python:3.11-slim AS builder

# Set shell and environment variables
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Install system build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Install python dependencies in a virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .

# Install requirements without cache to save space
RUN pip install --no-cache-dir -r requirements.txt

# --- Stage 2: Runtime Stage ---
FROM python:3.11-slim AS runner

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH" \
    PORT=8000

WORKDIR /app

# Install runtime system library needed by LightGBM (libgomp1)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Copy virtual environment from builder stage
COPY --from=builder /opt/venv /opt/venv

# Create a non-privileged user and group for security
RUN groupadd -g 10001 appgroup && \
    useradd -u 10001 -g appgroup -m -s /bin/bash appuser

# Create directory to cache mounted/downloaded model weights
RUN mkdir -p /app/models && chown -R appuser:appgroup /app/models

# Copy source code and config files
COPY --chown=appuser:appgroup app/ /app/app/
COPY --chown=appuser:appgroup config/ /app/config/

# Switch to the non-root user
USER appuser

# Expose port
EXPOSE 8000

# Health check using a python-based check to avoid curl dependency in slim image
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health', timeout=2)" || exit 1

# Start the application using Uvicorn
ENTRYPOINT ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
