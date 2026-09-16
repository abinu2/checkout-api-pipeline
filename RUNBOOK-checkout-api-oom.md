# Runbook: checkout-api OOMKilled Incident

## Scope

Use this runbook when `checkout-api` pods are restarting and Kubernetes reports
`OOMKilled`. The service runs in the `checkout-api` namespace.

Known-good baseline from the deployment manifest:

- Memory request: `64Mi`
- Memory limit: `128Mi`
- Deployment: `checkout-api`
- Container: `checkout-api`
- Expected replicas: `1`

## Diagnosis

### 1. Check pod health and restart activity

```bash
kubectl get pods -n checkout-api -l app=checkout-api -o wide
kubectl get deployment checkout-api -n checkout-api
kubectl get events -n checkout-api --sort-by=.lastTimestamp | tail -30
```

Look for a pod in `CrashLoopBackOff`, a growing `RESTARTS` count, or recent
warning events such as `OOMKilled` or `Back-off restarting failed container`.

### 2. Identify the affected pod

```bash
POD=$(kubectl get pods -n checkout-api -l app=checkout-api \
  -o jsonpath='{.items[0].metadata.name}')
echo "$POD"
```

If more than one pod is present, select the affected pod explicitly instead of
using the command above.

### 3. Confirm the termination reason

```bash
kubectl describe pod "$POD" -n checkout-api
kubectl get pod "$POD" -n checkout-api \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{" last="}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{"\n"}{end}'
```

Diagnosis is confirmed when the `checkout-api` container's last terminated
state has `reason=OOMKilled` (normally exit code `137`). This means the
container exceeded its Kubernetes memory limit; it does not by itself prove a
memory leak.

### 4. Check the configured limit and recent usage

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="checkout-api")].resources}{"\n"}'
kubectl top pod "$POD" -n checkout-api --containers
kubectl logs "$POD" -n checkout-api --previous --tail=200
```

If `kubectl top` is unavailable, continue with the pod status and deployment
configuration. The postmortem failure mode is a memory limit reduced from
`128Mi` to `8Mi`, which is below the service's normal working requirement.

## Resolution

### Immediate recovery: restore the known-good limit

First inspect the current value:

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="checkout-api")].resources.limits.memory}{"\n"}'
```

Restore the limit and request without editing unrelated settings:

```bash
kubectl set resources deployment/checkout-api -n checkout-api \
  --requests=cpu=100m,memory=64Mi \
  --limits=cpu=250m,memory=128Mi
```

This changes the pod template and starts a rollout. Watch it complete:

```bash
kubectl rollout status deployment/checkout-api -n checkout-api --timeout=120s
kubectl get pods -n checkout-api -l app=checkout-api -o wide
```

### Declarative recovery through the manifest

If the deployment is managed from this repository, ensure
`checkout-api-deployment.yaml` contains the known-good values, then apply it:

```bash
kubectl apply -f checkout-api-deployment.yaml
kubectl rollout status deployment/checkout-api -n checkout-api --timeout=120s
```

Do not apply a manifest that still contains the unsafe `8Mi` limit.

### Verify the service is stable

```bash
kubectl get pods -n checkout-api -l app=checkout-api
kubectl describe deployment checkout-api -n checkout-api
kubectl get events -n checkout-api --sort-by=.lastTimestamp | tail -30
kubectl logs deployment/checkout-api -n checkout-api --tail=100
```

Recovery is complete when the new pod is `Running` and `READY`, the restart
count stops increasing, readiness and liveness probes pass, and no new
`OOMKilled` events appear during the observation period.

## Rollback and escalation

If the rollout is unhealthy for reasons unrelated to memory, inspect the
rollout history and return to the previous revision:

```bash
kubectl rollout history deployment/checkout-api -n checkout-api
kubectl rollout undo deployment/checkout-api -n checkout-api
kubectl rollout status deployment/checkout-api -n checkout-api --timeout=120s
```

Escalate when the pod is still OOMKilled at `128Mi`, usage is steadily growing,
or the deployment cannot become ready. Capture these outputs for the incident:

```bash
kubectl describe pod "$POD" -n checkout-api
kubectl logs "$POD" -n checkout-api --previous --tail=500
kubectl get deployment checkout-api -n checkout-api -o yaml
kubectl get events -n checkout-api --sort-by=.lastTimestamp
```

Do not increase limits blindly. If normal usage approaches `128Mi`, measure
peak usage under representative load and agree on updated requests and limits
before changing production capacity.
