# Architecture Justification

## Chosen Pattern: Online Serving + Streaming Ingestion

### Why Online Serving?
The product requirement is p95 latency ≤ 120ms on every home-screen load.
Batch or async inference cannot satisfy this — the user is waiting on-screen.
Online serving with a Redis cache in front of the model server is the only
pattern that reliably meets the budget.

### Why a Feature Store?
User signals (last 30 days of browsing + purchases) must be available at
inference time in under 10ms. A feature store (Feast) pre-computes and
materializes these vectors so the model server never hits the data warehouse
at request time.

### Why Kafka for ingestion?
Clickstream events arrive at ~800 RPS peak. Synchronous writes to the
warehouse at that rate would create backpressure on the serving path.
Kafka decouples ingestion from storage, absorbs traffic spikes, and feeds
both the feature store (low-latency updates) and the warehouse (batch replay).

### Why A/B Test Router?
The product team requires continuous A/B testing against the model. A thin
router layer in front of the model server routes a configurable % of traffic
to challenger models without touching the serving or gateway code.

### Cold-Start Strategy
Users with no history (<30 days) receive popularity-based fallback
recommendations served by the Recommendation Service directly, bypassing
the model server. This ensures no user sees an empty or broken home screen.

### Trade-offs Accepted
| Decision | What we gain | What we give up |
|---|---|---|
| Redis cache (TTL 60s) | Latency headroom, cost reduction | Personalization staleness up to 60s |
| TorchServe over Triton | Simpler ops, native PyTorch | Lower raw throughput ceiling |
| Feast over custom store | Faster build, maintained OSS | Less flexibility on feature schemas |
| Single-region deployment | Cost, simplicity | No geo-redundancy at launch |