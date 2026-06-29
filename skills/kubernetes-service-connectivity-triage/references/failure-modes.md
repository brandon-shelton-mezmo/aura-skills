# Service Connectivity Failure Modes — Decision Tree

For each pattern: how to confirm it, what causes it, what to check next, and how to fix.

---

## Service exists but Endpoints is empty (selector mismatch — the #1 silent failure)

**Confirm:**
- `kubectl get service <name>` returns a Service.
- `kubectl get endpoints <name>` (or `kubectl get endpointslices -l kubernetes.io/service-name=<name>`) returns an object with no addresses (or `subsets: <none>`).
- `kubectl get pods -l <service-selector>` returns zero pods.
- `kubectl get pods` (no selector) shows pods exist that *should* back this service.

**What it tells you:** the Service's `spec.selector` does not match any pod's `metadata.labels`. Traffic to the Service has nowhere to go.

**Common causes:**

- **Label rename without Service update.** Someone changed `app: web` to `app: web-api` on the Deployment, but the Service still selects `app: web`. The Service is stale.
- **New ReplicaSet with different labels.** A rollout changed pod labels (rare but happens with template patches). Old pods deleted, new pods don't match the selector.
- **Typo in selector.** `app: webb` instead of `app: web`. The Service was misconfigured from the start.
- **Selector includes a label pods no longer have.** Service selects `{app: web, version: v1}`, pods are now `{app: web, version: v2}` — Service selector is too narrow.

**Fix path:** edit the Service's `spec.selector` to match actual pod labels, **or** edit the pods' labels to match the Service's selector. Choose based on which is the intended state. Verify by re-checking Endpoints — addresses should appear within seconds.

---

## Service exists, pods match, but targetPort is wrong

**Confirm:**
- `kubectl get service <name> -o yaml` shows ports like `port: 80, targetPort: 8080`.
- `kubectl get pods -l <selector> -o yaml` shows containers exposing `containerPort: 80` (not 8080).
- Endpoints object IS populated (pods match selector and are Ready), but `curl` to the Service:port gets connection refused or times out.

**What it tells you:** kube-proxy forwards traffic to `<pod-ip>:<targetPort>` — that port has nothing listening on it.

**Common causes:**

- Service spec was copy-pasted from a different app with a different port.
- The pod's container port was changed (security hardening: moved from 80 to 8080) but the Service was not updated.
- Named target port (`targetPort: http`) but no container declares a port with that name.

**Fix path:** edit the Service's `targetPort` to match the container's actual `containerPort`, **or** edit the container to listen on the expected port. Verify with a curl from a debug pod inside the cluster.

---

## Service missing entirely (typo, deleted, wrong namespace)

**Confirm:**
- `kubectl get service <name>` returns "Not Found."
- Clients log "no such host" or DNS resolution failure on `<name>.<namespace>.svc.cluster.local`.
- An Ingress or another Service is configured to route to this missing service.

**What it tells you:** the Service resource does not exist where consumers expect it.

**Common causes:**

- Service was deleted accidentally (kubectl delete typo, GitOps removal, namespace cleanup).
- Service was created in the wrong namespace (`default` instead of `production`).
- Service name typo in the manifest (`payment-service` vs `payments-service`).
- Service was renamed but consumers still reference the old name.

**Fix path:** recreate the Service from spec, or fix consumer references. If the Service is owned by a Helm chart / GitOps, re-sync rather than ad-hoc create.

---

## Ingress backend wrong

**Confirm:**
- External client (browser, curl from outside cluster) gets a 404 from the ingress controller, or a 503 "no upstream available."
- `kubectl get ingress <name> -o yaml` shows `backend.service.name` is wrong, or the `port.number` does not match the Service's `port`.
- The target Service exists and has populated Endpoints when accessed directly from inside the cluster.

**What it tells you:** the Ingress is routing the hostname/path to a Service that does not match the requested service.

**Common causes:**

- Service was renamed; Ingress backend not updated.
- Multiple Ingress rules and the wrong one matches first.
- Path-based routing where the path regex is wrong.
- TLS certificate Secret missing or expired (different symptom: TLS handshake failure, not 404).
- Annotations for the specific Ingress controller (nginx, traefik) are misconfigured.

**Fix path:** edit the Ingress backend service name/port. For TLS, fix or recreate the cert Secret.

---

## NetworkPolicy default-deny without allow rule

**Confirm:**
- All other checks (Service, Endpoints, target port, Ingress) are correct.
- Curl from inside the cluster, from a pod in the same namespace as the destination, may work.
- Curl from a different namespace or from monitoring/probe pods fails (connection timeout, not refused).
- `kubectl get networkpolicy -n <namespace>` shows one or more policies with `policyTypes: [Ingress]` and a `podSelector` that matches the destination pods.

**What it tells you:** the destination pod is subject to a NetworkPolicy that defaults to deny, and no ingress rule allows the source.

**Common causes:**

- A team added a default-deny policy to harden a namespace but forgot to allow monitoring, ingress controller, or cross-namespace clients.
- A policy was scoped wider than intended (`podSelector: {}` matches all pods).
- The allow rule has an incorrect `namespaceSelector` or `podSelector` — close but not quite right.

**Fix path:** add an ingress rule allowing the specific source (label selector, namespace label, or CIDR), or remove the default-deny if it was added prematurely. Test from the actual source after applying.

> NetworkPolicy denies are **silent** — there are no events, no logs, and no Kubernetes-level error. The only way to detect them is by elimination after confirming the rest of the request path is correct.

---

## Service routes correctly, but the application returns errors

**Confirm:**
- Service Endpoints populated, target port correct, traffic arrives at pods.
- Pods are `Ready`, no restarts, but the application logs show 5xx or business-logic errors.

**What it tells you:** this is **not a service-connectivity problem**. The connectivity layer is healthy. The fault is application-level or downstream.

**Fix path:**
- If the app's errors mention a downstream service, that downstream is the new triage target — recursively scope.
- If the app's errors are internal logic, hand to the application team — out of SRE scope.
- If the symptom is high latency rather than errors, load `kubernetes-capacity-saturation-triage`.

Do not invent a service-connectivity root cause when the request path is healthy.

---

## Multiple services failing the same way

If the connectivity layer is broken for multiple unrelated Services, this is a **platform-level issue** — not service-by-service triage. Re-scope to:

- **kube-proxy** unhealthy on a node — `kubectl get pods -n kube-system -l k8s-app=kube-proxy`
- **CNI plugin** failing (Calico, Cilium, etc.) — pod-to-pod traffic broken across the cluster
- **CoreDNS** down — load `kubernetes-network-triage`
- **Cluster-wide NetworkPolicy** newly applied — check namespace-wide and cluster-wide policies

Escalate and stop service-by-service triage.
