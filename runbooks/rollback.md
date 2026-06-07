# Rollback Runbook: Recommendations Service

## Trigger Thresholds
Roll back immediately if ANY of the following are true for ≥ 5 minutes:
- [ ] Error rate > 1% (alert: `AvailabilityBurnRateCritical`)
- [ ] p95 latency > 150ms (alert: `LatencyP95Exceeded`)
- [ ] Multiple `X-Model-Version` values across replicas (alert: `ModelVersionMismatch`)

## Rollback Steps

### 1. Confirm the alert (2 min)
- [ ] Open Grafana → Recommendations dashboard → verify error rate / latency spike
- [ ] Check `kubectl get pods -n production -l app=recommendations` — any CrashLoopBackOff?
- [ ] Identify current model version: `kubectl get deployment recommendations -n production -o jsonpath='{.spec.template.spec.containers[0].env}'`

### 2. Roll back model version (3 min)
- [ ] Find the last known-good version in MLflow:
```bash
  python scripts/get_last_good_version.py --stage production
  # outputs e.g. v1.2.1
```
- [ ] Patch the deployment:
```bash
  kubectl set env deployment/recommendations \
    MODEL_VERSION="<last_good_version>" \
    -n production
  kubectl rollout status deployment/recommendations -n production --timeout=5m
```

### 3. Verify recovery (5 min)
- [ ] p95 latency returns below 120ms within 2 minutes of rollout complete
- [ ] Error rate returns below 0.1%
- [ ] All replicas report the same `X-Model-Version` in Grafana

### 4. Abort canary if active (1 min)
- [ ] Scale canary deployment to 0:
```bash
  kubectl scale deployment recommendations-canary --replicas=0 -n production
```
  > Note: canary deployment only exists during active rollouts. Skip this step if no canary is running.
```

### 5. Communicate and document (5 min)
- [ ] Post in `#ml-incidents` Slack: model version rolled back from `<bad>` to `<good>`, reason, time
- [ ] Open incident ticket; link to alert and this runbook
- [ ] Mark bad model version as `archived` in MLflow:
```bash
  python scripts/archive_model.py --model-version "<bad_version>" --reason "production rollback"
```

## Total estimated time: < 20 minutes

## Escalation
If rollback does not resolve within 20 minutes, page the ML Lead on-call.