# Capacity / Saturation Failure Modes — Decision Tree

For each pattern: how to confirm it, what causes it, and how to fix.

---

## CPU throttling at limit

**Confirm:**
- `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])` > 0 (any non-zero is meaningful; > 25% is severe).
- `rate(container_cpu_usage_seconds_total[5m])` is at or very near the container's `resources.limits.cpu`.
- User-visible symptom: latency spikes, increased timeouts, but no errors from the application itself.

**What it tells you:** the container has hit its CFS quota. The kernel is suspending the container for parts of each 100ms period. Even with idle CPUs on the node, the container cannot use them.

**Common causes:**

- CPU limit set too low for the workload's actual demand under load.
- CPU limit set with no testing — copied from a smaller workload.
- Garbage collector or other background threads competing with request handlers for the same quota.

**Fix path (in-scope):**

- Raise `resources.limits.cpu`.
- Or remove the CPU limit entirely, keeping only `resources.requests.cpu` (controversial but eliminates throttling; do this only if the node has CPU headroom and you're not running multi-tenant).
- Verify by re-checking throttle ratio after the change.

Throttling under load is the **most common silent saturation failure** in Kubernetes — the metric is rarely on default dashboards. Always check it explicitly. This is the `ad_service_high_cpu` failure class.

---

## Memory pressure short of OOM

**Confirm:**
- `container_memory_working_set_bytes` is > 85% of `resources.limits.memory` but not yet hitting it.
- For managed runtimes: GC frequency / pause time is elevated.
- Latency degrades correlates with memory utilization climbing.

**What it tells you:** the workload is approaching its memory limit. Managed runtimes spend more time in GC trying to stay under the cap. Native runtimes face kernel reclaim pressure.

**Common causes:**

- Memory limit too low for the workload's actual working set.
- Cache or in-memory store growing under load.
- Memory leak (rare but possible) — distinguishes by monotonic growth across hours, not load-correlated.

**Fix path (in-scope):**

- Raise `resources.limits.memory`.
- Verify by re-checking working set vs limit and (for managed runtimes) GC pause time after the change.

If working set grows monotonically regardless of load, you have a memory leak. That's an application-level fix, out of one-shot scope.

---

## Excessive GC (managed runtime)

**Confirm:**
- JVM: `jvm_gc_pause_seconds_sum` over the recent window divided by wall time > 0.05 (5%).
- Go: `go_gc_duration_seconds_count` rising rapidly; `go_gc_duration_seconds{quantile="0.99"}` long.
- Node: `nodejs_gc_duration_seconds_sum` similar.
- Symptom: latency spikes correlated with GC events; CPU is busy but throughput is low.

**What it tells you:** the runtime is spending significant time in GC. Either heap is too small, allocation rate is too high, or GC configuration is wrong.

**Common causes:**

- Heap size too small for the working set (`-Xmx` low, Go `GOGC` aggressive).
- Object churn in a hot path (excessive allocation per request).
- Wrong GC algorithm for the workload (e.g., Parallel GC on a low-latency service).
- Someone is calling `System.gc()` or equivalent manually — this is the `ad_service_manual_gc` failure class.

**Fix path:**

- **In-scope:** raise the container memory limit + raise heap (`-Xmx`). Heap should be ~70% of container limit, leaving room for non-heap memory (metaspace, native, threads).
- **Out-of-scope:** GC algorithm choice, code-level allocation reduction, removing manual `System.gc()` calls. Mark as application-team action.

---

## Slow image pull on cold start

**Confirm:**
- Pod events show `Pulling image "..."` followed by `Pulled image ... in 2m30s` (or similar long duration).
- Pod's `Initialized` condition is `False` for the duration of the pull.
- Symptom: scale-out under load is slow because new pods take minutes to become Ready.

**What it tells you:** the container image is large, the node hasn't cached it, and the pull is the bottleneck.

**Common causes:**

- Multi-gigabyte image (often a JDK-based app with bundled dependencies).
- Image registry is slow or rate-limited.
- Node has slow disk I/O.
- Image cache cleared (recent node restart, garbage collection).

**Fix path:**

- **In-scope short term:** prewarm by deploying replicas before the load spike (HPA target lower so scale-out triggers earlier).
- **Out-of-scope:** image size optimization, multi-stage build, distroless base — application/CI team action.

This is the `ad_service_image_slow_load` failure class.

---

## Load test flooding the system

**Confirm:**
- Request rate is 10x+ normal.
- A load generator workload exists in the cluster (e.g., `loadgenerator` pod, `k6-runner`, `locust`).
- The "outage" timing exactly matches the load test schedule.

**What it tells you:** the failure is not an outage — the system is being asked to handle more load than it's provisioned for, and the load is synthetic.

**Fix path:**

- Stop the load test, or scale the target workload to handle it.
- If the load test represents intended production traffic, scale permanently (HPA, replicas, vertical resize).

This is the `loadgenerator_flood_homepage` failure class. Recognize it as a *test* condition, not a true incident — the mitigation may simply be "this is expected, the load gen is doing its job."

---

## Retry storm from upstream client

**Confirm:**
- Request rate spikes far above any organic explanation.
- The spike comes from a single client (one Service, one User-Agent in logs).
- Recent change to that client (or a downstream dependency it's calling) preceded the spike.
- Errors are concentrated on retried requests.

**What it tells you:** an upstream service is retrying failed requests aggressively. A small upstream failure (one slow request, one error) triggers a flood. This is `capacity_decrease_rpc_retry_storm`.

**Fix path:**

- Out-of-scope (application change): add exponential backoff + jitter, cap max retries, implement circuit breaker.
- In-scope short term: rate-limit the offending client at the destination Service (Ingress rate limits, sidecar / mesh policies).

---

## "Pods Ready but timing out" — saturation not connectivity

If pods are Ready, Endpoints populated, target port correct, NetworkPolicy fine — but clients still time out — the issue is **the workload is too slow to respond within the client's timeout**. This is saturation manifesting as timeouts at the client.

Walk back through Steps 2–4 (CPU throttle, memory, GC, slow startup) to find the saturation factor. The connectivity layer is healthy; you're in the right skill.
