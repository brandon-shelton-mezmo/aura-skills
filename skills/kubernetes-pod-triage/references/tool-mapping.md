# Capability → Tool Mapping

The `kubernetes-pod-triage` skill describes investigation steps in **capability** terms ("fetch the pod's events"). This reference maps those capabilities to concrete tools across the most common Kubernetes MCP servers and the shell fallback.

The agent should use whatever tool in its toolbox satisfies the capability. This file is a crib sheet, not a contract — tool names and parameters change between MCP server versions, so prefer reading the agent's actual tool descriptions when there is ambiguity.

## Capability matrix

| Capability | [`manusa/kubernetes-mcp-server`](https://github.com/manusa/kubernetes-mcp-server) | [`Flux159/mcp-server-kubernetes`](https://github.com/Flux159/mcp-server-kubernetes) | Shell (kubectl) |
|---|---|---|---|
| Get a pod's full status | `pods_get` | `kubectl_get` with `resourceType: pod` | `kubectl get pod <name> -n <ns> -o yaml` |
| Describe a pod (human-readable) | `pods_get` (includes events inline) | `kubectl_describe` | `kubectl describe pod <name> -n <ns>` |
| List events for a pod | `events_list` with field selector | `kubectl_get` with `resourceType: events` and field selector | `kubectl get events -n <ns> --field-selector involvedObject.name=<name> --sort-by=.lastTimestamp` |
| Read current container logs | `pods_log` | `kubectl_logs` | `kubectl logs <pod> -n <ns> -c <container>` |
| Read previous-instance logs | `pods_log` with `previous: true` | `kubectl_logs` with `previous: true` | `kubectl logs <pod> -n <ns> -c <container> --previous` |
| Get owning Deployment/ReplicaSet | `resources_get` | `kubectl_get` | `kubectl get deploy,rs -n <ns> -l <selector>` |
| Get a ConfigMap or Secret | `resources_get` | `kubectl_get` | `kubectl get configmap/secret <name> -n <ns> -o yaml` |
| Get PVC and bound PV | `resources_get` | `kubectl_get` | `kubectl get pvc <name> -n <ns>` then `kubectl get pv <bound>` |
| Get Service and Endpoints | `resources_get` | `kubectl_get` | `kubectl get svc <name> -n <ns> -o yaml` and `kubectl get endpoints <name> -n <ns>` |
| Rollout history | `resources_get` on Deployment with revisions | `kubectl_get` on deployments | `kubectl rollout history deployment/<name> -n <ns>` |
| Node status (for evictions / NotReady) | `nodes_list` / `resources_get` | `kubectl_get` with `resourceType: nodes` | `kubectl get node <name> -o yaml` |

## When a capability is missing

If the agent's toolbox does not provide a needed capability:

- **Read-only capabilities** (status, events, logs) are essential. Stop and tell the user the skill cannot run without them.
- **Mutating capabilities** (rollback, scale, delete) are optional for this skill — it is an investigation skill, not a remediation skill. The agent should produce a recommendation and let the user (or a separate remediation skill / human operator) act on it.

## Notes on common MCP servers

- **manusa/kubernetes-mcp-server**: Go-based, supports OpenShift extensions. Tool names use `pods_*`, `resources_*`, `nodes_*`, `events_*`. Events are also embedded in `pods_get` output.
- **Flux159/mcp-server-kubernetes**: Node.js, models tools after kubectl verbs (`kubectl_get`, `kubectl_describe`, `kubectl_logs`). Most operations route through a generic `kubectl_*` tool with `resourceType` and name parameters.
- **kubectl-ai's MCP**: exposes a single shell-like tool that runs kubectl commands. Functionally equivalent to shell fallback.

## Shell fallback notes

If the agent has shell access but no kubernetes MCP server:

- Always pass `-n <namespace>` explicitly. Defaulting to the current context's namespace is a common source of confused tool calls.
- Prefer `-o yaml` over `-o json` for human-readable output the agent can scan; use `-o jsonpath=...` when extracting a specific field.
- For events, always `--sort-by=.lastTimestamp` — the default order is creation time, which is rarely what you want.
- Be careful with `kubectl exec` — it is a mutating capability (can change container state) and should go through the HITL approval gate if one is configured.
