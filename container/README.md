# Container Plan

## Base Image
`python:3.11-slim` — Debian-based slim variant.
Chosen over Alpine for better compatibility with PyTorch C extensions.

## Bake vs Mount Decision
**Model weights are mounted at runtime, not baked into the image.**
See ADR 0002 for full rationale.

At container startup, `src/server.py` downloads the model artifact from:
`s3://ml-artifacts/recommendations/${MODEL_VERSION}/model.pt`
Download time: ~15s on a warmed instance (S3 Transfer Acceleration enabled).

## Multi-Stage Build

| Stage | Purpose | What it produces |
|---|---|---|
| `builder` | Install Python dependencies | `/install` directory |
| `runtime` | Copy code + deps, drop privileges | Final deployable image |

The builder stage is discarded; only the runtime layer ships.

## Image Size Estimate

| Layer | Size |
|---|---|
| python:3.11-slim base | ~130MB |
| PyTorch (CPU) + deps | ~420MB |
| Application code | ~5MB |
| **Total** | **~555MB** |

Model weights (~800MB) are not included. Registry stores image and model separately.

## Security

Runs as non-root user `appuser`. No SSH or unnecessary shell utilities. Trivy scan runs in CI before push; critical CVEs block promotion. No secrets baked into the image — all credentials are injected via environment variables at runtime.

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `MODEL_VERSION` | Yes | e.g. `v1.3.0` — used to fetch weights from S3 on startup |
| `MODEL_STORE_URI` | Yes | S3 base URI for model artifacts |
| `PORT` | No | Default `8080` |
| `REDIS_URL` | Yes | Redis connection string |
| `FEATURE_STORE_URL` | Yes | Feast online store endpoint |