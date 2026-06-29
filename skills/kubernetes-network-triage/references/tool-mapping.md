# Capability → Tool Mapping

## Capability matrix

| Capability | manusa/kubernetes-mcp-server | Flux159/mcp-server-kubernetes | Shell (kubectl) |
|---|---|---|---|
| Exec command in a pod | (varies — usually shell exec) | `kubectl_exec` | `kubectl exec -n <ns> <pod> -- <cmd>` |
| Run a debug pod for DNS testing | (varies) | `kubectl_run` | `kubectl run dnsutils --rm -it --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 -- bash` |
| Get CoreDNS pods | `pods_list` with selector | `kubectl_get` | `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| Get CoreDNS logs | `pods_log` | `kubectl_logs` | `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=200` |
| Get CoreDNS ConfigMap | `resources_get` | `kubectl_get` | `kubectl get configmap coredns -n kube-system -o yaml` |
| List NetworkPolicy in namespace | `resources_list` with kind=NetworkPolicy | `kubectl_get` | `kubectl get networkpolicy -n <ns>` |
| Get pod's dnsPolicy/dnsConfig | `pods_get` | `kubectl_get` | `kubectl get pod <name> -n <ns> -o jsonpath='{.spec.dnsPolicy}'` |
| Restart CoreDNS pods (mitigation) | `pods_delete` with selector | `kubectl_delete` | `kubectl delete pod -n kube-system -l k8s-app=kube-dns` |
| Edit CoreDNS ConfigMap (mitigation) | `resources_update` | `kubectl_patch` | `kubectl edit configmap coredns -n kube-system` (manual) or `kubectl patch ...` |

## SREGym mapping

Under SREGym, the kubectl MCP server is `sregym_kubectl`:

- DNS testing via exec: `sregym_kubectl.exec_read_only_kubectl_cmd("exec -n <ns> <pod> -- nslookup <name>")` — note that exec is generally read-only-classifiable when the command is a query like nslookup.
- CoreDNS ConfigMap inspect: `sregym_kubectl.exec_read_only_kubectl_cmd("get configmap coredns -n kube-system -o yaml")`
- NetworkPolicy list: `sregym_kubectl.exec_read_only_kubectl_cmd("get networkpolicy -n <ns> -o yaml")`
- ConfigMap patch (mitigation): `sregym_kubectl.exec_kubectl_cmd_safely("patch configmap coredns -n kube-system --type=merge -p '{...}'")`
- CoreDNS pod restart (mitigation): `sregym_kubectl.exec_kubectl_cmd_safely("delete pod -n kube-system -l k8s-app=kube-dns")`

## Notes

- Always test resolution from a **pod in the affected namespace**, not from your local shell. Cluster DNS is pod-context-sensitive (search domains, NetworkPolicy).
- After CoreDNS ConfigMap changes, deletion of CoreDNS pods is often needed unless the `reload` plugin is enabled in the Corefile. Check whether `reload` appears in the Corefile before deciding.
- For NetworkPolicy debugging, `kubectl exec` into a pod and run `nc -zv <target> <port>` or `curl -v` — there is no kubectl-side equivalent.
