# Personalized Recommendations — MLOps Design Dossier

## Executive Summary

This repository is the complete MLOps design dossier for Scenario X — a
personalized product recommendation system for a mobile retail app. The system
serves up to 800 RPS at peak with a p95 latency budget of 120ms end-to-end.
A TorchServe model server reads real-time user features from a Feast feature
store; a Redis cache reduces effective model load to ~240 RPS at peak. A/B
testing is supported via a thin routing layer in front of the model server.
Users with no purchase history receive popularity-based fallback
recommendations. The system runs on AWS EKS with canary rollouts, automated
load-test promotion gates, and multi-window burn-rate alerts tied to explicit
rollback triggers.

## Architecture Diagram

See [architecture/architecture.md](architecture/architecture.md).

## Key Numbers

| Metric | Value |
|---|---|
| Target RPS (peak) | 800 |
| p95 latency budget | 120ms end-to-end |
| Availability SLO | 99.9% / 30-day window |
| Latency SLO | 99.5% of requests < 120ms |
| Model size | ~800MB (mounted from S3 at startup) |
| Hardware (inference) | AWS c7g.2xlarge × 5 (autoscale 3–8) |
| Monthly cost estimate | ~$2,790 |

## Navigation

| Area | Artifact |
|---|---|
| Architecture + justification | [architecture/architecture.md](architecture/architecture.md) |
| Pattern trade-offs | [architecture/JUSTIFICATION.md](architecture/JUSTIFICATION.md) |
| ADR: sync vs async inference | [architecture/adr/0001-sync-vs-async-inference.md](architecture/adr/0001-sync-vs-async-inference.md) |
| ADR: model bake vs mount | [architecture/adr/0002-model-bake-vs-mount.md](architecture/adr/0002-model-bake-vs-mount.md) |
| Model lifecycle + promotion gates | [lifecycle/lifecycle.md](lifecycle/lifecycle.md) |
| Model registry spec | [lifecycle/model-registry.yaml](lifecycle/model-registry.yaml) |
| Dockerfile | [container/Dockerfile](container/Dockerfile) |
| Container plan | [container/README.md](container/README.md) |
| API contract (OpenAPI 3.1) | [api/openapi.yaml](api/openapi.yaml) |
| Example payloads | [api/examples/](api/examples/) |
| Capacity plan | [serving/capacity-plan.md](serving/capacity-plan.md) |
| SLO definitions | [serving/slos.yaml](serving/slos.yaml) |
| Load test plan | [serving/load-test-plan.md](serving/load-test-plan.md) |
| CI/CD pipeline | [cicd/.github/workflows/deploy-model.yml](cicd/.github/workflows/deploy-model.yml) |
| Monitoring alerts | [monitoring/alerts.yaml](monitoring/alerts.yaml) |
| Rollback runbook | [runbooks/rollback.md](runbooks/rollback.md) |

## Open Questions

1. **Feature freshness SLA** — the design assumes the feature store reflects
   user events within 60 seconds. If the data engineering team cannot guarantee
   this, the Redis TTL and cold-start fallback threshold need to be revisited.

2. **A/B assignment persistence** — requests are currently routed by user ID
   hash at request time. If the product team requires sticky experiment
   assignment across devices or sessions, a persistent assignment store must
   be added before the first A/B test goes live.

3. **Model size growth** — the ~800MB artifact estimate is based on the current
   two-tower candidate. If the team adopts a larger transformer-based model,
   the S3 cold-start time (~15s) and instance memory budget must be re-evaluated
   before promotion to production.