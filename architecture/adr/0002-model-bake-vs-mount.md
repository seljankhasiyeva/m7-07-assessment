# ADR 0002: Model Weights Mounted at Runtime, Not Baked into Image

## Status
Accepted

## Context
The model artifact (~800MB) can either be baked into the Docker image at build
time or mounted from object storage (S3) at container startup. Both approaches
are viable; the choice affects CI/CD speed, image size, and rollback procedure.

## Decision
Model weights are mounted from S3 at container startup. The Docker image
contains only the serving code and dependencies. The model version to load
is controlled by an environment variable `MODEL_VERSION` injected at deploy time.

## Consequences

**Positive**
- Docker image remains ~600MB instead of ~1.4GB; registry push/pull is fast.
- Rolling back the model requires only changing `MODEL_VERSION` and redeploying
  — no new image build required.
- The same image can serve any registered model version, enabling blue/green
  and A/B deployments without rebuilding.
- CI/CD pipeline does not need access to model weights during the build stage.

**Negative**
- Container cold-start is slower (~15s to download weights from S3 on first start).
- S3 availability becomes a startup dependency; mitigated by using S3 Transfer
  Acceleration and a pre-warmed instance pool.
- Image and model provenance are tracked separately; the registry must link them.

## Alternatives Rejected
**Baked image:** Simpler provenance (one artifact = one deployable unit) but
rebuilding the image on every model promotion adds ~8 minutes to the promotion
pipeline and bloats the image registry.