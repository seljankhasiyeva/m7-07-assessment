# Architecture: Personalized Recommendations (Scenario X)

## System Diagram

```mermaid
flowchart TD
    subgraph CLIENT ["Client Layer"]
        APP[Mobile App]
    end

    subgraph SERVING ["Serving Layer"]
        GW[API Gateway]
        REC[Recommendation Service]
        CACHE[Redis Cache]
    end

    subgraph ML ["ML Layer"]
        FS[Feature Store]
        MOD[Model Server - TorchServe]
        AB[A/B Test Router]
    end

    subgraph DATA ["Data Layer"]
        ES[Event Stream - Kafka]
        DW[Data Warehouse - Snowflake]
        REG[Model Registry - MLflow]
    end

    APP -->|GET /recommendations| GW
    GW --> CACHE
    CACHE -->|miss| REC
    REC --> AB
    AB --> MOD
    FS --> MOD
    DW --> FS
    APP -->|click/purchase events| ES
    ES --> DW
    ES --> FS
    REG -->|model weights| MOD
```

## Component Overview

| Component | Technology | Role |
|---|---|---|
| API Gateway | AWS API Gateway | Auth, rate limiting, routing |
| Recommendation Service | Python / FastAPI | Orchestration, cold-start fallback |
| Redis Cache | Redis 7 | Response cache, TTL 60s |
| Feature Store | Feast | Real-time + historical features |
| Model Server | TorchServe | Online inference |
| A/B Test Router | Custom | Traffic split between model versions |
| Event Stream | Kafka | Clickstream ingestion |
| Model Registry | MLflow | Versioning, promotion gates |
| Data Warehouse | Snowflake | Historical training data |