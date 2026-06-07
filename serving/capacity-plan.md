# Capacity Plan

## Target Load
| Metric | Value |
|---|---|
| Peak RPS | 800 |
| Average RPS | 300 |
| p95 latency budget | 120ms end-to-end |
| p95 latency target (model server) | ≤ 90ms |
| p99 latency target (model server) | ≤ 150ms |

## Latency Budget Breakdown
| Component | Budget |
|---|---|
| API Gateway + network | 10ms |
| Redis cache lookup | 2ms |
| Feature store fetch | 15ms |
| Model inference | 80ms |
| Response serialization | 3ms |
| **Total** | **110ms** (10ms headroom) |

## Cache Hit Rate Assumption
At TTL=60s with ~300 avg RPS, a returning user hits cache ~70% of the time.
Effective model RPS = 800 × 0.30 = **240 RPS** at peak.

## Model Server Sizing
Single TorchServe instance (4 vCPU, 16GB RAM) benchmarks at ~80 RPS at p95=80ms.
Required replicas at peak: ceil(240 / 80) = **3 replicas minimum**.
With 50% headroom for traffic spikes: **5 replicas**.

## Full Stack Sizing
| Component | Instance Type | Count | Reason |
|---|---|---|---|
| API Gateway | AWS managed | — | Scales automatically |
| Recommendation Service | t3.medium (2vCPU, 4GB) | 4 | Thin orchestration layer |
| Redis Cache | r7g.large (2vCPU, 16GB) | 2 (primary + replica) | Session cache |
| Feature Store (Feast) | r7g.xlarge (4vCPU, 32GB) | 2 | Low-latency feature reads |
| Model Server (TorchServe) | c7g.2xlarge (8vCPU, 16GB) | 5 | Inference, autoscale 3–8 |
| Kafka | m7g.large × 3 | 3 | Event ingestion |

## Autoscaling Policy
- Model Server: scale-out at CPU > 65% for 2 minutes; scale-in at CPU < 30% for 10 minutes.
- Min replicas: 3 (to handle cold-start on scale-out)
- Max replicas: 8

## Monthly Cost Estimate (AWS us-east-1)
| Component | Monthly |
|---|---|
| Model Server (5× c7g.2xlarge) | ~$1,400 |
| Feature Store (2× r7g.xlarge) | ~$480 |
| Redis (2× r7g.large) | ~$280 |
| Recommendation Service (4× t3.medium) | ~$120 |
| Kafka (3× m7g.large) | ~$360 |
| S3 + data transfer | ~$150 |
| **Total** | **~$2,790/month** |