# Model Lifecycle: Personalized Recommendations

## End-to-End Diagram

```mermaid
flowchart LR
    subgraph DEV ["Development"]
        D1[Feature Engineering]
        D2[Model Training]
        D3[Offline Evaluation]
    end

    subgraph STAGING ["Staging"]
        S1[Integration Tests]
        S2[Shadow Mode]
        S3[Load Test]
    end

    subgraph PROD ["Production"]
        P1[Canary 5%]
        P2[A/B Test]
        P3[Full Rollout]
    end

    REG[(MLflow Registry)]
    MON[Monitoring]

    D1 --> D2 --> D3
    D3 -->|pass| REG
    REG -->|promote: Staging| S1
    S1 --> S2 --> S3
    S3 -->|pass| REG
    REG -->|promote: Production| P1
    P1 -->|metrics OK| P2
    P2 -->|winner declared| P3
    P3 --> MON
    MON -->|drift detected| D1
```

## Gates

| Gate | Criteria | Approver |
|---|---|---|
| Dev → Staging | Offline NDCG@10 ≥ 0.42, coverage ≥ 85% | ML Engineer (automated) |
| Staging → Production | Shadow p95 < 90ms, zero errors in load test | ML Lead (manual sign-off) |
| Canary → Full | CTR uplift ≥ +2% vs baseline, p95 < 120ms | Product + ML Lead |
| Rollback trigger | Error rate > 1% or p95 > 150ms for 5 min | On-call (automated alert) |

## Retraining Schedule
- **Scheduled:** Weekly full retrain on last 90 days of event data.
- **Triggered:** If feature drift score (PSI) > 0.2 on any top-10 feature.