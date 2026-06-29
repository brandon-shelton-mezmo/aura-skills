# Capability → Tool Mapping

Capacity triage relies more heavily on metrics tools than the pure-kubectl skills. Most clusters have Prometheus; SREGym specifically provides it.

## Metrics capabilities

| Capability | PromQL example | Notes |
|---|---|---|
| Request rate | `sum(rate(http_server_requests_total{service="<svc>"}[5m]))` | Adjust label name to your metrics |
| Error rate | `sum(rate(http_server_requests_total{service="<svc>",status=~"5.."}[5m]))` | |
| Latency p99 | `histogram_quantile(0.99, sum(rate(http_server_requests_duration_seconds_bucket{service="<svc>"}[5m])) by (le))` | |
| CPU usage | `rate(container_cpu_usage_seconds_total{namespace="<ns>",pod=~"<pod-prefix>.*"}[5m])` | |
| CPU limit | `kube_pod_container_resource_limits{namespace="<ns>",pod=~"<pod-prefix>.*",resource="cpu"}` | |
| CPU throttle ratio | `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])` | The critical metric — almost never on default dashboards |
| Memory working set | `container_memory_working_set_bytes{namespace="<ns>",pod=~"<pod-prefix>.*"}` | |
| Memory limit | `kube_pod_container_resource_limits{namespace="<ns>",resource="memory"}` | |
| JVM GC pause | `rate(jvm_gc_pause_seconds_sum[5m])` / total | Requires JVM micrometer/jmx_exporter |
| Go GC pause | `rate(go_gc_duration_seconds_sum[5m])` | Standard in Go apps with Prometheus client |

## Tool mapping

| Capability | manusa/kubernetes-mcp-server | Flux159/mcp-server-kubernetes | Shell |
|---|---|---|---|
| Query Prometheus | (varies — usually separate prometheus MCP) | (same) | `curl 'http://prometheus:9090/api/v1/query?query=...'` |
| Get HPA status | `resources_get` | `kubectl_get` | `kubectl get hpa -n <ns>` |
| Scale a deployment (mitigation) | `resources_patch` on replicas | `kubectl_scale` | `kubectl scale deployment/<name> -n <ns> --replicas=N` |
| Patch resource limits (mitigation) | `resources_patch` | `kubectl_patch` | `kubectl patch deployment/<name> -n <ns> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","resources":{"limits":{"cpu":"2000m"}}}]}}}}'` |

## SREGym mapping

SREGym provides:

- `sregym_prometheus.get_metrics(query)` — PromQL queries against the in-cluster Prometheus.
- `sregym_kubectl.exec_read_only_kubectl_cmd(command)` — for kubectl read-only.
- `sregym_kubectl.exec_kubectl_cmd_safely(command)` — for kubectl mutating (mitigation).

Typical saturation triage flow under SREGym:

1. `sregym_prometheus.get_metrics("rate(http_server_requests_total{service='<svc>'}[5m])")` — establish load.
2. `sregym_prometheus.get_metrics("rate(container_cpu_cfs_throttled_periods_total{namespace='<ns>',pod=~'<svc>.*'}[5m]) / rate(container_cpu_cfs_periods_total{namespace='<ns>',pod=~'<svc>.*'}[5m])")` — check throttle ratio.
3. `sregym_prometheus.get_metrics("container_memory_working_set_bytes{namespace='<ns>',pod=~'<svc>.*'}")` and `kube_pod_container_resource_limits{...,resource='memory'}` — memory vs limit.
4. Application-specific metrics for GC if the workload exports them.
5. `sregym_kubectl.exec_kubectl_cmd_safely("patch deployment/<name> -n <ns> -p '...'")` for the mitigation.

## Notes

- **Aggregate across pods** when checking throttling — one hot pod can be hidden in the average. Use `max by (pod)` or `topk(5, ...)` to surface outliers.
- **Choose your window carefully** — 5m is good for steady-state; 1m for spiky load tests; 15m+ for slow drift.
- **Saturation metrics lag** — give them 1–5 minutes to settle after a mitigation change before declaring success or failure.
