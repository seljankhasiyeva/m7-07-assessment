# Load Test Plan

## Tool
k6 (https://k6.io) — scriptable, CI-friendly, outputs p95/p99 metrics natively.

## Environment
Staging environment only. Never run against production.
Model version: latest staging candidate (set via `X-Model-Version` header).

## Test Scenarios

### Scenario 1: Baseline (steady state)
- **Duration:** 10 minutes
- **RPS:** 300 (average load)
- **Pass criteria:**
  - p95 latency < 100ms
  - Error rate < 0.1%
  - Zero 5xx responses

### Scenario 2: Peak load
- **Duration:** 5 minutes ramp-up → 10 minutes sustained → 5 minutes ramp-down
- **RPS:** ramp 0 → 800 over 5 min, hold 800 for 10 min
- **Pass criteria:**
  - p95 latency < 120ms during sustained phase
  - Error rate < 0.5%
  - Model server CPU < 80%

### Scenario 3: Cold-start flood
- **Duration:** 5 minutes
- **RPS:** 200
- **User profile:** 100% new users (no feature store data)
- **Pass criteria:**
  - All requests return 200 (fallback fires, no errors)
  - p95 latency < 120ms (fallback path must be fast)

### Scenario 4: Cache bypass stress
- **Duration:** 5 minutes
- **RPS:** 200
- **Setup:** Redis disabled in test env
- **Pass criteria:**
  - System degrades gracefully; p95 < 200ms
  - No 5xx errors

## k6 Script Sketch
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    peak: {
      executor: 'ramping-arrival-rate',
      startRate: 0,
      timeUnit: '1s',
      stages: [
        { target: 800, duration: '5m' },
        { target: 800, duration: '10m' },
        { target: 0,   duration: '5m' },
      ],
      preAllocatedVUs: 100,
      maxVUs: 200,
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<120'],
    http_req_failed:   ['rate<0.005'],
  },
};

export default function () {
  const userId = `usr_${Math.random().toString(36).slice(2, 10)}`;
  const res = http.get(`https://staging-api.example.com/v1/recommendations/${userId}`, {
    headers: {
      'X-API-Key': __ENV.API_KEY,
      'X-Model-Version': __ENV.MODEL_VERSION,
    },
  });
  check(res, {
    'status 200': (r) => r.status === 200,
    'p95 < 120ms': (r) => r.timings.duration < 120,
  });
  sleep(0.1);
}
```

## CI Integration
Load tests run automatically when a model is promoted to staging.
A failed load test blocks promotion to production (see cicd/deploy-model.yml).

## Note
The k6 script above is saved as `serving/load-test-script.js` for CI execution.
This document describes the test plan; the script is the executable version.