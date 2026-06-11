# Architecture Diagram

This document describes the high-level architecture of the `retail-recs-service` for Scenario X (Personalized B2C Retail Recommendations).

## System Architecture Diagram

Below is the Mermaid diagram showing how data flows from the user, through the API gateway, to the serving service and the database, as well as the offline training loop.

```mermaid
graph TD
    %% User and Ingress
    User([Mobile App Client]) -->|1. GET /recommend| Gateway["API Gateway / Envoy"]
    Gateway -->|2. Route Request| Router["A/B Router / Service Mesh"]
    
    %% Real-Time Serving Path
    Router -->|3a. Call| Serving["FastAPI Recommendation Service"]
    Router -->|3b. Direct fallback if system error| FallbackDB[("Redis Cache: Trending Fallback")]
    
    %% Feature Retrieval & Serving Logic
    Serving -->|4. Get last 30d features| FeatureStore[("Redis: Feast Feature Store")]
    Serving -->|5. Predict & Rank| ModelEngine["LightGBM Ranker Model"]
    Serving -->|6. Query popular products if cold-start| FallbackDB
    
    %% Data Ingestion (Feedback Loop)
    User -->|7. Log click/purchase events| Kafka["Kafka Event Broker"]
    Kafka -->|8. Stream ingestion| ObjectStorage[("Cloud Object Storage: S3/GCS")]
    
    %% Offline Training and Feature Store Refresh
    ObjectStorage -->|9. Compute features| Spark["Spark Feature Pipeline"]
    Spark -->|10. Write online features| FeatureStore
    Spark -->|11. Write offline features| Lakehouse[("Delta Lake / Iceberg")]
    
    %% ML Lifecycle
    Lakehouse -->|12. Train model| Training["Training Job: Vertex AI / SageMaker"]
    Training -->|13. Log metadata| Registry[("MLflow Model Registry")]
    Registry -->|14. Publish approved weights| ModelStorage[("Model Bucket: S3/GCS")]
    
    %% Mounting weights to Container
    ModelStorage -.->|15. Read/cache weights| Serving
    
    %% Styling
    classDef primary fill:#2b5c8f,stroke:#1d3f63,stroke-width:2px,color:#fff;
    classDef database fill:#e67e22,stroke:#d35400,stroke-width:2px,color:#fff;
    classDef client fill:#2ecc71,stroke:#27ae60,stroke-width:2px,color:#fff;
    classDef offline fill:#7f8c8d,stroke:#34495e,stroke-width:2px,color:#fff;
    
    class User client;
    class Gateway,Router,Serving,ModelEngine primary;
    class FeatureStore,FallbackDB,ObjectStorage,Lakehouse,Registry,ModelStorage database;
    class Kafka,Spark,Training offline;
```

### Flow Description

1. **Client Request**: The mobile app sends a request to get recommendations for a user.
2. **API Gateway**: Handles authentication, rate limiting, and forwards the request.
3. **A/B Router**: Routes traffic based on the active A/B testing splits (e.g., 50% to `control`, 50% to `treatment-a`).
4. **Recommendation Service**:
   - Queries the **Feature Store** (Redis) using the `user_id` to get the user's last 30 days of browsing and purchase data.
   - If the user has history, the service passes features to the **LightGBM Ranker** (loaded in-memory) to get scores.
   - If the user is a cold-start (no history in Feature Store) or if the database lookup fails, the service falls back to fetching **Trending Fallback** products.
5. **Feedback Loop**: User interactions (clicks, purchases) are sent to Kafka, processed offline, and used to refresh both the feature store and the training data.
