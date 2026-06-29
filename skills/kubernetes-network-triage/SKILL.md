---
name: kubernetes-network-triage
description: Triage and remediate Kubernetes cluster-network failures — DNS resolution failing, CoreDNS misbehaving, NetworkPolicy silently dropping traffic, dnsPolicy/dnsConfig misconfiguration, pod-to-pod connectivity broken across namespaces. Use when the symptom is "name resolution failed", "no such host", "i/o timeout" between pods, or "connectivity works from one namespace but not another". Walks DNS resolution path → CoreDNS health → NetworkPolicy → CNI in order. Distinguishes DNS faults from Service-level faults (load kubernetes-service-connectivity-triage if the issue is Service routing rather than DNS or NetworkPolicy). Produces a single primary root cause with confidence and a verified mitigation. Requires a Kubernetes-capable tool including the ability to exec into a pod (or run a debug pod) for DNS testing.
---

# Kubernetes Network Triage

Use this skill when the symptom is **DNS resolution failure, NetworkPolicy denial, or generalized pod-to-pod connectivity loss** — distinct from Service-level routing problems (selector mismatch, target port, Ingress). If the user can reach the Service IP but not the Service DNS name, that's DNS — you're in the right skill. If the Service DNS resolves but Endpoints is empty, that's connectivity — switch to `kubernetes-service-connectivity-triage`.

You are operating one-shot: one diagnosis, one mitigation, no retries. Be correct first time. "Unknown with hypotheses" beats a wrong fix.

## Step 0 — Scope check

Network triage is the right skill when:

- The symptom is `no such host`, `name resolution failed`, DNS lookup timeouts, or "we changed the DNS config and now nothing resolves."
- Traffic between pods works in one direction or one namespace but not another (NetworkPolicy fingerprint).
- A recent change to CoreDNS ConfigMap, custom resolvers, or NetworkPolicy correlates with the failure.

Network triage is **not** the right skill when:

- The Service's Endpoints is empty → `kubernetes-service-connectivity-triage`.
- The pod itself is unhealthy → `kubernetes-pod-triage`.
- DNS resolves but the application errors → application-level, out of scope.

Restate the user's symptom and verify the fingerprint before continuing.

## Step 1 — Test DNS resolution from inside the cluster

Network triage's first move is always: **prove DNS resolution from a known-good pod**. Without this, you cannot distinguish DNS failure from connectivity failure.

Capability: "exec a command in a pod (or run a debug pod) that performs DNS resolution and basic TCP connectivity tests."

Run, from a pod in the affected namespace:

1. `nslookup kubernetes.default.svc.cluster.local` — the cluster-internal API Service. If this fails, DNS itself is broken at the cluster level.
2. `nslookup <failing-service>.<failing-namespace>.svc.cluster.local` — the specific Service the user can't reach. If `kubernetes.default` resolved but this didn't, the Service doesn't exist (back to `kubernetes-service-connectivity-triage`).
3. `nslookup <failing-service>` (short name) — exercises the pod's search domains. If FQDN works but short name doesn't, the pod's `dnsPolicy` / `dnsConfig` is wrong.
4. `nslookup google.com` (or any external name) — exercises the upstream resolver chain. If internal works but external fails, an upstream `forward` directive in CoreDNS is broken.

Interpret the matrix:

| internal FQDN | internal short | external | Likely fault |
|---|---|---|---|
| ✓ | ✓ | ✓ | DNS is healthy — fault is elsewhere. Re-scope. |
| ✗ | ✗ | ✗ | CoreDNS is down or the pod's nameserver is wrong. Step 2. |
| ✓ | ✗ | ✓ | Pod's search domains / `dnsPolicy` misconfigured. Step 5. |
| ✓ | ✓ | ✗ | CoreDNS upstream forward is broken. Step 3. |
| ✗ | ✗ | ✓ | Specific Service doesn't exist (re-scope) OR CoreDNS is healthy but cluster zone is missing. |

If you cannot exec into a pod or run a debug pod, **stop and report** — you cannot triage DNS without resolution-test capability.

## Step 2 — Check CoreDNS health

Capability: "list pods in kube-system filtered to CoreDNS, get their state and recent logs."

- `kubectl get pods -n kube-system -l k8s-app=kube-dns` — confirm replicas are Running and Ready.
- If any are restarting or CrashLooping, **pod-level issue first** — load `kubernetes-pod-triage` for the CoreDNS pod, then return.
- `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=200` — look for plugin errors, upstream connection failures, or `[ERROR]` entries.

CoreDNS failure modes:

- **All replicas CrashLooping** → recent ConfigMap change with a syntax error. Step 3.
- **Pods Ready but logs show `[ERROR] plugin/errors`** → specific resolution failures, usually upstream forward broken or a domain plugin misconfigured.
- **Pods Ready, no errors, but resolution fails** → kube-proxy or CNI issue downstream of CoreDNS. Step 6.

## Step 3 — Inspect CoreDNS ConfigMap

Capability: "get a ConfigMap by name."

`kubectl get configmap coredns -n kube-system -o yaml`

The Corefile lives under `data.Corefile`. Read it for:

- **Syntax errors** — a recent edit with a missing brace, typo'd plugin name, or wrong block order will CrashLoop CoreDNS. The `kubectl logs` will name the parse error and line number.
- **Wrong upstream `forward`** — `forward . 8.8.8.8` vs `forward . /etc/resolv.conf` vs a custom DNS server that's unreachable. If external resolution fails but internal works, this is the cause.
- **Missing cluster zone** — `kubernetes cluster.local in-addr.arpa ip6.arpa { ... }` block deleted or scoped wrong. Causes internal resolution to fail.
- **Stale custom domains** — `hosts` or `template` plugin entries that override real DNS with wrong addresses. This is the `stale_coredns_config` failure class.

Note: ConfigMap changes do not auto-reload CoreDNS in all setups. After fixing, you may need to delete CoreDNS pods so the new config is picked up. Confirm whether the deployment uses `reload` plugin (auto-pickup) or not.

## Step 4 — Check NetworkPolicy in the destination namespace

NetworkPolicies that look like DNS or connectivity failures:

- A default-deny policy in the destination namespace that doesn't allow DNS (UDP 53 to `kube-system` CoreDNS) — pods cannot resolve anything.
- A policy that allows DNS but doesn't allow the source pod's traffic to the destination.

`kubectl get networkpolicy -n <destination-ns>`

For each policy, check `spec.podSelector` (which destination pods it applies to), `policyTypes`, and the rule lists.

**Test:** if removing the policy (temporarily, in a non-prod environment, with rollback ready) fixes DNS or connectivity, NetworkPolicy is the cause. Re-apply with corrected rules.

This is the `network_policy_block` failure class.

## Step 5 — Pod-level dnsPolicy / dnsConfig

For per-pod DNS issues (Step 1 matrix shows short name resolution broken or pod has unusual resolution behavior):

`kubectl get pod <name> -n <ns> -o yaml` → look at `spec.dnsPolicy` and `spec.dnsConfig`.

- `dnsPolicy: ClusterFirst` (default) — uses CoreDNS, falls back to nodes's resolver. Almost always what you want.
- `dnsPolicy: Default` — uses the node's resolver only, **does not use CoreDNS**. Cluster-internal short names will fail. This is the `wrong_dns_policy` failure class.
- `dnsPolicy: ClusterFirstWithHostNet` — for `hostNetwork: true` pods. If used on a regular pod, behaves like `Default`.
- `dnsPolicy: None` with `dnsConfig` — fully custom. Look at `nameservers` and `searches` for wrong values.

## Step 6 — CNI / kube-proxy (last resort, escalate-level)

If DNS is fully healthy and NetworkPolicy is correct but pod-to-pod connectivity still fails across the cluster, the issue is at the CNI layer:

- `kubectl get pods -n kube-system` — check Calico/Cilium/Flannel/Weave pods are Ready on all nodes.
- `kubectl get pods -n kube-system -l k8s-app=kube-proxy` — check kube-proxy is healthy.
- Node-level: iptables/IPVS rules corrupt, MTU mismatch causing fragmentation, kernel modules missing.

This level usually requires escalation to platform/cluster operators — application-level SRE triage stops here.

## Step 7 — Reporting (root-cause discipline)

```
Failure scope: <which pods/namespaces affected>
User-reported symptom: <verbatim>
DNS resolution matrix (from Step 1):
  internal FQDN: ✓/✗
  internal short: ✓/✗
  external: ✓/✗

Findings (evidence only):
- <call> → <observation>
- ...

Primary root cause: <one sentence>
Confidence: high | medium | low
Why this is the root cause: <evidence>
Why it is not [next-likely alternative]: <evidence>

Other anomalies observed (NOT root causes):
- ...

Recommended mitigation: <see Step 8>
```

Discipline reminders:

- **One primary fault.** "CoreDNS misconfigured AND NetworkPolicy too strict" is suspicious — usually one of them is the cause of the user-reported symptom and the other is pre-existing.
- **Pre-existing CoreDNS warnings** are not the root cause of a 10-minute-old DNS outage. Look at when the symptom started vs when the anomaly first appeared.
- **Low confidence → diagnosis is "unknown" with hypotheses.**

## Step 8 — Mitigation (pre-flight + post-fix verification)

If confidence is `low`, do not mitigate.

```
Mitigation pre-flight:
  Diagnosis I am acting on: <restatement>
  Smallest change: <one edit — patch CoreDNS Corefile, remove/edit NetworkPolicy rule, patch pod dnsPolicy, etc.>
  Predicted post-fix state:
    - nslookup from the same test pod returns: <expected>
    - (specific to the fix)
  Verification calls: <read-only DNS test from the original failing pod>
  Rollback if predicted ≠ actual: <inverse edit>
```

For CoreDNS ConfigMap changes, **rolling-restart CoreDNS pods** is often required for the config to take effect — include that in the change, then verify resolution from a pod after the restart completes.

For NetworkPolicy changes, verify with a test connection from the actual source pod (not just kubectl outside the cluster).

If actual ≠ predicted, the diagnosis was wrong. Roll back, report, do not attempt a second fix.

## Cross-skill handoffs

- DNS healthy but Service Endpoints empty → `kubernetes-service-connectivity-triage`
- CoreDNS pod itself unhealthy → `kubernetes-pod-triage` (on the CoreDNS pod)
- DNS healthy but service is slow under load → `kubernetes-capacity-saturation-triage`

## Skill resources

- `references/failure-modes.md` — detailed decision tree for each network failure pattern.
- `references/tool-mapping.md` — capability-to-tool mapping.
