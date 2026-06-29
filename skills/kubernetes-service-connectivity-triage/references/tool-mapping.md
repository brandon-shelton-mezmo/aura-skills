# Capability → Tool Mapping

The skill describes investigation steps in **capability** terms. This reference maps capabilities to concrete tools across common Kubernetes MCP servers and the shell fallback.

## Capability matrix

| Capability | manusa/kubernetes-mcp-server | Flux159/mcp-server-kubernetes | Shell (kubectl) |
|---|---|---|---|
| Get a Service by name | `resources_get` | `kubectl_get` | `kubectl get svc <name> -n <ns> -o yaml` |
| List Endpoints for a Service | `resources_get` | `kubectl_get` | `kubectl get endpoints <name> -n <ns> -o yaml` |
| List pods matching a label selector | `pods_list` with labelSelector | `kubectl_get` with selector | `kubectl get pods -n <ns> -l <key=val>` |
| Get pod spec (for container ports) | `pods_get` | `kubectl_get` | `kubectl get pod <name> -n <ns> -o yaml` |
| Get Ingress | `resources_get` | `kubectl_get` | `kubectl get ingress <name> -n <ns> -o yaml` |
| List NetworkPolicy in namespace | `resources_list` with kind=NetworkPolicy | `kubectl_get` with `resourceType: networkpolicy` | `kubectl get networkpolicy -n <ns>` |
| Test connectivity from inside cluster | (varies — usually shell exec) | `kubectl_exec` | `kubectl run debug --rm -it --image=curlimages/curl -- curl http://<svc>:<port>` |
| Edit Service (mitigation) | `resources_update` / `resources_patch` | `kubectl_patch` | `kubectl patch svc <name> -n <ns> --type=merge -p '<json>'` |
| Edit Ingress (mitigation) | `resources_update` / `resources_patch` | `kubectl_patch` | `kubectl patch ingress <name> -n <ns> --type=merge -p '<json>'` |
| Edit NetworkPolicy (mitigation) | `resources_update` / `resources_patch` | `kubectl_patch` | `kubectl patch networkpolicy <name> -n <ns> --type=merge -p '<json>'` |

## SREGym mapping

If running under SREGym, the kubectl MCP server is namespaced as `sregym_kubectl`:

- Read-only: `sregym_kubectl.exec_read_only_kubectl_cmd(command)` — pass the full kubectl command string.
- Mutating: `sregym_kubectl.exec_kubectl_cmd_safely(command)` — same shape, used only during mitigation.

For Service connectivity work, the patterns reduce to:

- `exec_read_only_kubectl_cmd("get svc <name> -n <ns> -o yaml")`
- `exec_read_only_kubectl_cmd("get endpoints <name> -n <ns> -o yaml")`
- `exec_read_only_kubectl_cmd("get pods -n <ns> -l <selector> --show-labels")`
- `exec_read_only_kubectl_cmd("get networkpolicy -n <ns>")`
- `exec_kubectl_cmd_safely("patch svc <name> -n <ns> --type=merge -p '{...}'")`

## Notes

- Always pass `-n <ns>` explicitly. Defaulting to the current context's namespace is a common confusion source.
- Use `--show-labels` on `get pods` to see actual labels vs. expected selector at a glance.
- For NetworkPolicy debugging, a curl from inside a pod in the source namespace is the only definitive test — kubectl alone cannot prove a NetworkPolicy is the cause.
