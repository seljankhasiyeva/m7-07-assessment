# ADR 0001: Synchronous Online Inference

## Status
Accepted

## Context
Home-screen recommendations must render before the user sees the app.
The product team set a p95 latency budget of 120ms end-to-end.
Two options were considered: synchronous online inference (request → model → response)
and async pre-computation (recommendations generated ahead of time, stored, fetched).

## Decision
We adopt synchronous online inference with a Redis cache layer.

Fresh personalization signals (last 30-day activity) are incorporated at request
time via the feature store. Cache TTL is set to 60 seconds to limit redundant
model calls for the same user within a session.

## Consequences

**Positive**
- Recommendations reflect the user's most recent activity.
- A/B test assignment is clean — each request is routed deterministically.
- No pre-computation job to maintain or schedule.

**Negative**
- Model server becomes a hard dependency in the critical path.
- Cache miss latency must be tightly controlled (target < 90ms model + feature
  fetch, leaving 30ms for gateway + network).
- Requires robust circuit-breaking if the model server degrades.

## Alternatives Rejected
**Async pre-computation:** Lower operational complexity but recommendations are
stale by minutes to hours. Unacceptable for users with high activity velocity
(e.g. a user who browses 20 items then immediately returns to the home screen).