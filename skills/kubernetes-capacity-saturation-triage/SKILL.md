---
name: kubernetes-capacity-saturation-triage
description: Triage and remediate Kubernetes capacity and saturation problems — symptoms like "service is slow", "high latency", "error rate spike under load", "CPU pegged", "GC pauses", "pods Ready but timing out", "load test floods the system". Use when pods are Running and Ready, no recent crashes, no probe failures, no connectivity issues — but the workload cannot keep up with its load. Walks resource utilization → throttling → GC pressure → startup-vs-steady-state → traffic shape. Distinguishes capacity issues from pod-level crashes (kubernetes-pod-triage) and connectivity failures (kubernetes-service-connectivity-triage). Produces a single primary root cause with confidence and a verified mitigation (which may be code-level and out of one-shot remediation scope). Requires a Kubernetes-capable tool and a metrics tool (Prometheus or equivalent).
---

# Kubernetes Capacity / Saturation Triage

Use this skill when the user reports **slowness, high latency, intermittent errors, or load-related failures** AND the pods are demonstrably healthy (Running, Ready, no recent restarts, no probe failures). If pods are unhealthy, use `kubernetes-pod-triage`. If Service routing is broken, use `kubernetes-service-connectivity-triage`. If the symptom is "totally down" rather than "degraded," it's probably not this skill.

You are operating one-shot: one diagnosis, one mitigation, no retries. Be correct first time. Saturation problems often have **mitigations that are application-code-level** (raise a limit, scale out, fix GC config, optimize a slow endpoint) — some of these you can do from Kubernetes, others require handing off to the application team. Recognizing which is in scope is part of correct diagnosis.

## Step 0 — Scope check

Saturation triage is the right skill when:

- Symptom: slow responses, latency spikes, error rate climbing as load climbs, request timeouts (not connection failures), p99 latency degraded.
- Pods: Running, Ready, no restarts in the relevant window, no probe failures.
- The failure correlates with **load** — usage is up, traffic spiked, or a load test is running.

Saturation triage is **not** the right skill when:

- Pods are OOMKilled or CrashLooping → `kubernetes-pod-triage` (OOMKilled is pod-triage, not saturation, because the pod restarted).
- Service Endpoints are empty or wrong → `kubernetes-service-connectivity-triage`.
- DNS or NetworkPolicy is broken → `kubernetes-network-triage`.
- All requests fail uniformly regardless of load → an outage, not saturation.

## Step 1 — Establish the load picture

Before looking at resource usage, establish what the load looks like.

Capability: "query metrics for request rate, error rate, and latency on the failing service."

- **Request rate** (RPS) — is current rate above normal? When did it start climbing?
- **Error rate** — what fraction of requests are failing? Is the error type 5xx (server-side), 4xx (client), or timeouts?
- **Latency** — p50, p95, p99. Has p99 detached from p50? (Long tail under saturation.)
- **Concurrent in-flight requests** (if available) — how many requests are mid-flight at the worst moment?

If load is normal but performance is degraded, the issue is downstream or in the workload itself (slow database, slow dependency, GC, code change). If load is elevated, the question becomes "is the workload provisioned for this load."

## Step 2 — Resource utilization vs limits

Capability: "query container_cpu_usage_seconds_total, container_memory_working_set_bytes, container_cpu_cfs_throttled_periods_total, kube_pod_container_resource_limits for the workload."

For each container in the workload, compute:

- **CPU utilization** — `rate(container_cpu_usage_seconds_total[5m])` vs the `cpu` limit.
- **CPU throttling** — `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])`. Any non-zero throttling under load is a smoking gun: the container is CPU-limited and queueing.
- **Memory utilization** — `container_memory_working_set_bytes` vs the `memory` limit. Approaching the limit produces GC thrash (managed runtimes) or kernel reclaim pressure (native) — both slow the workload before OOMKilling.

Patterns:

| Signal | Diagnosis |
|---|---|
| CPU throttle ratio > 0% under load | CPU limit too low. Raise limit, or remove limit (use request only). |
| Memory near limit, no OOM yet, GC time elevated | Memory limit too low for working set. Raise limit. |
| CPU below limit AND memory below limit AND still slow | Not a resource constraint. Step 3 or 4. |
| One pod hot, others cold | Load not distributed evenly. Service may not be load-balancing (sticky sessions, gRPC connection pooling). |

This is the `ad_service_high_cpu` failure class. CPU throttling is silent — there's no event, no log; only the throttle metric reveals it.

## Step 3 — Runtime-specific saturation (GC, heap, thread pools)

If utilization looks healthy but the app is slow, the issue may be inside the runtime.

**Managed runtimes (JVM, .NET, Go, Node):**
- GC pause time (JVM: `jvm_gc_pause_seconds`; Go: `go_gc_duration_seconds`; Node: `nodejs_gc_duration_seconds`). Sustained GC > 5% of wall-clock is a problem.
- Heap utilization. Heap > 85% of max triggers more frequent GC; > 95% can effectively stall the app.
- This is the `ad_service_manual_gc` failure class (where someone or something is triggering excessive GC).

**Native runtimes:**
- Thread pool exhaustion (HTTP server threads, DB connection pool).
- Lock contention — visible as low CPU but high latency.

If the workload exposes a `/metrics` endpoint, those tell the story. If not, you may be limited to inferring from external symptoms (request latency vs CPU).

## Step 4 — Slow startup vs steady-state slow

If the symptom is "pods become Ready but slow for the first minute," check:

- Application startup work (JIT warm-up, cache fills, connection pool warm-up).
- Slow image pull at the node level — `kubelet` events on the pod for `Pulling` / `Pulled` with long duration. This is `ad_service_image_slow_load`. If the image is multi-gigabyte and the node has slow disk or limited bandwidth, every pod restart suffers.

Slow startup looks like saturation when a HPA scales out under load — new pods take time to warm up, latency spikes during scale-out.

## Step 5 — Traffic shape (load test or real spike)

If load is genuinely above what the system was provisioned for:

- Is this a **load test** (synthetic, controlled, can be turned off)? — `loadgenerator_flood_homepage` is a test workload that floods the application as part of the demo.
- Is this a **real traffic spike** (organic, must be served)?
- Is this a **retry storm** from an upstream client misbehaving? — capacity-decrease + RPC retry storm patterns. If retries are uncapped or have no backoff, a small failure becomes a flood.

Mitigation depends on the answer:
- Load test: stop the test, scale appropriately for real load.
- Real spike: scale up (HPA, manual replica increase, vertical scale).
- Retry storm: fix the client's retry policy, or add circuit-breaking at the failing service.

## Step 6 — Reporting (root-cause discipline)

```
Failing workload: <deployment/statefulset>
User-reported symptom: <verbatim>
Load picture: <RPS, error rate, latency — current vs normal>
Resource picture: CPU <util/throttle>, Memory <util>, GC <% if relevant>

Findings (evidence only):
- <metric query> → <observation>
- ...

Primary root cause: <one sentence — the single saturation factor>
Confidence: high | medium | low
Why this is the root cause: <evidence>
Why it is not [next-likely alternative]: <evidence>

Other anomalies observed (NOT root causes):
- ...

Recommended mitigation: <see Step 7 — explicitly call out if mitigation is out of one-shot scope>
```

Discipline reminders:

- **One primary fault.** "CPU throttled AND GC elevated" — pick the one that explains the symptom. Often elevated GC is downstream of CPU throttling (the GC threads are also throttled).
- **Saturation symptoms often present after the cause.** A burst of traffic 5 minutes ago can cause a queue that's still draining now. Look at when load *started*, not just current values.
- **Some mitigations are out of one-shot scope.** "Refactor the slow endpoint" or "fix the GC config in the application" is not something you can do from kubectl. Diagnose correctly; recommend the fix; mark mitigation as "code-level, requires application owner."

## Step 7 — Mitigation (pre-flight + post-fix verification)

If confidence is `low`, do not mitigate.

If the mitigation is in-scope (raise a limit, scale replicas, stop a load test, fix a retry config), pre-flight:

```
Mitigation pre-flight:
  Diagnosis: <restatement>
  Smallest in-scope change: <e.g., raise cpu limit from 500m to 1000m, scale deployment from 2 to 4 replicas>
  Predicted post-fix state:
    - CPU throttle ratio drops below 1% within <N> minutes
    - p99 latency returns to <baseline value>
    - error rate drops to <baseline>
  Verification calls: <metric queries to confirm>
  Rollback if predicted ≠ actual: <revert the limit/replica change>
```

If the mitigation is out-of-scope (application code, GC tuning), the mitigation field should say so explicitly:

```
Mitigation: out-of-scope for kubectl-level remediation. Application owner action required: <specific recommendation, e.g., "raise JVM heap to -Xmx2g and add -XX:+UseG1GC", or "implement exponential backoff on retries to <upstream service>"]. Filing as diagnosis-only; no mitigation will be attempted from this run.
```

For in-scope mitigations, apply, wait for stabilization, then verify. Saturation metrics take 1–5 minutes to settle — don't declare success after 10 seconds.

If actual ≠ predicted, the diagnosis was wrong. Roll back, report, do not attempt a second fix.

## Cross-skill handoffs

- Pod crashes appearing during load → `kubernetes-pod-triage` (the crash, not the saturation, is the leading edge)
- Service Endpoints flapping under load → `kubernetes-service-connectivity-triage` (probe failures are pod-triage's domain)
- Database / external dependency slow → out of skill scope; investigate the dependency

## Skill resources

- `references/failure-modes.md` — detailed decision tree for saturation patterns.
- `references/tool-mapping.md` — capability-to-tool mapping for k8s metrics and operations.
