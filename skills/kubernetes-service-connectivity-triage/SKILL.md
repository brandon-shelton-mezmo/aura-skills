---
name: kubernetes-service-connectivity-triage
description: Triage and remediate Kubernetes service connectivity failures — symptoms like "users cannot reach service X", "service returns 5xx but pods are healthy", "endpoints empty", "ingress 404/503", "one service cannot call another". Use when the user-reported symptom is a request failure to a service AND the backing pods appear healthy (Running, Ready). Walks Service → Endpoints → Selector → Ingress → NetworkPolicy in order, with explicit checks for the most common 1-shot diagnosis errors (label/selector drift after a rename, missing target port, Ingress backend mismatch, default-deny NetworkPolicy with no allow). Produces a single primary root cause with confidence and a verified mitigation. Requires a Kubernetes-capable tool. If pods themselves are unhealthy, use kubernetes-pod-triage instead.
---

# Kubernetes Service Connectivity Triage

Use this skill when the **observable symptom is at the request layer** (HTTP 5xx, connection refused, timeouts, "service is down") and the **pods are healthy** (Running and Ready). If the pods are unhealthy, use `kubernetes-pod-triage` first.

This is a high-frequency diagnosis failure mode for AI SREs: the pods look fine, so the agent reports "everything is healthy" and misses that the Service selector no longer matches any pod labels (after a recent rename), or that the Ingress backend points at the wrong service, or that a NetworkPolicy is silently dropping traffic. The fix is usually a one-line edit. Finding it requires walking the request path in order.

You are operating one-shot: one diagnosis, one mitigation, no retries. Be correct first time. Saying "unknown with named hypotheses" is a *good* outcome; guessing is a bad outcome.

## Step 0 — Scope check

Before walking the request path, confirm this is the right skill:

1. **Restate the user's symptom verbatim.** "POST /api/checkout returns 500" is a symptom. "The cluster is broken" is not.
2. **Identify the failing surface:** which HTTP endpoint, which service name, which client.
3. **Confirm the backing pods are healthy:** for the service in question, list pods matching its selector. All `Ready`, no recent restarts, no CrashLoop. If any pod is unhealthy, **stop and use `kubernetes-pod-triage` first** — fix the pod-level problem before chasing connectivity.
4. **Confirm the symptom is request-shaped:** users see request failures, not "pod missing" or "out of memory" symptoms.

If Step 0 routes elsewhere, stop and load the right skill. Do not investigate connectivity for a pod problem.

## Step 1 — Walk the request path in order

The request path from outside-the-cluster to a pod is:

```
client → DNS → Ingress (or LoadBalancer) → Service → Endpoints → kube-proxy → pod
```

Walk it in that order. Stop at the first link where reality does not match the spec.

### 1a. Service exists and has the expected name

Capability: "get a Service by name in a namespace."

- Does the Service exist at all? A missing Service (typo, deleted, namespace mismatch) shows up as "name resolution failed" or "no such host" at the client. This is the `missing_service` failure class.
- Is the Service of the expected type (`ClusterIP`, `NodePort`, `LoadBalancer`)? Type mismatches cause external clients to think there is no service when there is one internally.

### 1b. Service selector matches pod labels (the #1 silent failure)

Capability: "get the Service spec (selector) AND list pods matching that selector."

A Service routes traffic to pods whose labels match its `spec.selector`. If labels and selector drift apart, **the Service exists, the pods exist, but Endpoints will be empty.** Common causes:

- A recent deployment renamed an app label (`app: web` → `app: webapp`) without updating the Service.
- The Service selector was typo'd or partially updated.
- The Service selector includes a label that pods used to have but a new ReplicaSet does not.

How to confirm: list pods with the Service's exact selector. If the returned pod count is zero, you have your root cause — selector mismatch.

### 1c. Endpoints object has addresses

Capability: "get the Endpoints (or EndpointSlice) object for a Service."

The kubelet/endpoint-controller writes pod IPs into the Service's Endpoints object only for pods that:
1. Match the selector (see 1b)
2. Are `Ready` (passed readiness probe)
3. Have an IP

**Empty Endpoints means traffic has nowhere to go.** Causes:
- Selector mismatch (see 1b).
- Pods exist but none are `Ready` — pod-level problem, switch skills.
- Pods have a Service-targeted port name that no container actually exposes (see 1d).

### 1d. Service ports map to pod container ports

Capability: "get Service spec ports + get pod spec container ports."

A Service `port` is what clients connect to. The Service's `targetPort` is the pod port traffic gets forwarded to. Mismatches:

- `targetPort: 8080` but the container only listens on `port: 80` — traffic forwarded to a closed port, connection refused.
- `targetPort: http` (named port) but no container declares a port named `http` — Endpoints object is still populated, but traffic goes nowhere.
- Port name match but protocol mismatch (`protocol: TCP` vs `UDP`) — silently dropped.

This is the `k8s_target_port-misconfig` and `wrong_service_selector` class of fault.

### 1e. Ingress backend correctness (if external traffic)

If the symptom is external (user/browser/curl from outside), check the Ingress:

- Does an Ingress resource exist for the failing hostname/path?
- Is `spec.rules[].http.paths[].backend.service.name` the right Service name? (Common: someone renamed the Service but not the Ingress.)
- Does `backend.service.port.number` match the Service's `port`?
- For TLS, is the secret referenced and valid?

This is the `ingress_misroute` failure class.

### 1f. NetworkPolicy is not silently denying

If 1a-1e all look right but traffic still doesn't flow, check NetworkPolicy:

- Are there NetworkPolicies in the source or destination namespace?
- If a policy exists with `policyTypes: [Ingress]` on the destination pod, it is **default-deny** unless an allow rule matches the source.
- Common: a default-deny policy was added without an allow rule for a specific client (cross-namespace traffic, monitoring, etc.).

This is the `network_policy_block` failure class. Note: NetworkPolicy denies do not log by default — the only signal is "traffic doesn't arrive and there's no obvious reason." Always check NetworkPolicy when 1a-1e look correct.

For DNS-specific failures (`stale_coredns_config`, `wrong_dns_policy`), this is the wrong skill — load `kubernetes-network-triage`.

## Step 2 — Failure mode mapping

If the request-path walk did not isolate the issue, load `references/failure-modes.md` via `read_skill_file` for the detailed decision tree on each pattern.

## Step 3 — Reporting (root-cause discipline)

A diagnosis is one primary root cause. Resist enumerating every anomaly.

```
Failing surface: <hostname/endpoint or in-cluster service:port>
User-reported symptom: <verbatim>
Backing pods: <count> pods, all <Ready/Unready> with <N> restarts

Findings (evidence only):
- Service '<name>' exists / selector '<key=val>' / endpoints addresses: <N>
- Pods matching selector: <N>
- Target port mapping: Service <port→targetPort>, pod listens on <port>
- Ingress (if applicable): backend '<svc>:<port>' → <matches | does not match>
- NetworkPolicy in namespace: <count>, applicable to destination: <yes/no>

Primary root cause: <one sentence — the single link in the request path that is broken>
Confidence: high | medium | low
Why this is the root cause: <evidence>
Why it is not [next-likely alternative]: <evidence>

Other anomalies observed (NOT root causes):
- (List incidental brokenness with reason it is not causal, or "none.")

Recommended mitigation: <see Step 4>
```

### Root-cause discipline

- **One primary fault.** Service exists but selector wrong AND target port wrong = two related anomalies but ONE root cause (the Service spec is incorrect). Don't list two root causes when one fix addresses both.
- **Downstream errors are not root causes.** "Checkout returns 500 because payment is unreachable because payment Service has empty Endpoints because selector mismatch" — the root cause is the selector mismatch. The 500s and unreachability are observable symptoms.
- **Low confidence → diagnosis is "unknown" with hypotheses.** Do not invent a cause.

## Step 4 — Mitigation (pre-flight + post-fix verification)

If diagnosis confidence is `low`, **do not mitigate**. Submit the diagnosis with hypotheses.

Pre-flight before any mutating call:

```
Mitigation pre-flight:
  Diagnosis I am acting on: <restatement>
  Smallest change: <one edit — usually patch a Service selector, fix targetPort, update Ingress backend, add NetworkPolicy allow rule>
  Predicted post-fix state:
    - Endpoints object for Service '<name>' will have <N> addresses
    - test-call from a client pod will return <expected>
    - (specific to the fix)
  Verification calls: <read-only calls to confirm>
  Rollback if predicted ≠ actual: <inverse edit>
```

Apply, verify, compare. If actual ≠ predicted, the diagnosis was wrong. Roll back, report, **do not attempt a second fix**.

## Cross-skill handoffs

- Backing pods are unhealthy → `kubernetes-pod-triage`
- DNS resolution failing (not Service routing) → `kubernetes-network-triage`
- Service routes correctly but downstream is slow/throttled → `kubernetes-capacity-saturation-triage`

## Skill resources

- `references/failure-modes.md` — detailed decision tree for each connectivity failure pattern.
- `references/tool-mapping.md` — capability-to-tool mapping.
