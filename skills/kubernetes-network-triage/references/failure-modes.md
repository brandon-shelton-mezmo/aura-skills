# Network Failure Modes — Decision Tree

For each pattern: how to confirm it, what causes it, and how to fix.

---

## CoreDNS CrashLooping after ConfigMap edit

**Confirm:**
- `kubectl get pods -n kube-system -l k8s-app=kube-dns` shows `CrashLoopBackOff` or rapidly restarting.
- `kubectl logs -n kube-system -l k8s-app=kube-dns --previous` shows a parse error or plugin initialization failure, often naming a Corefile line number.
- All resolution attempts from any pod fail with timeout.

**What it tells you:** the Corefile is syntactically invalid or references an undefined plugin.

**Common causes:**

- Missing closing brace `}` on a plugin block.
- Plugin name typo (`forwward` vs `forward`).
- Plugin used outside the `kubernetes` plugin scope where it requires it.
- A custom plugin reference that's not compiled into the CoreDNS image being used.

**Fix path:** edit the ConfigMap to restore valid syntax, then delete the CoreDNS pods (or wait for the `reload` plugin if enabled) so they pick up the new config. Verify CoreDNS Ready, then re-test DNS from a debug pod.

---

## CoreDNS healthy but specific zone fails

**Confirm:**
- CoreDNS pods Ready.
- `nslookup` from a pod resolves `kubernetes.default.svc.cluster.local` but fails for a specific external or custom domain.
- CoreDNS logs show `[ERROR] plugin/errors: ... HINFO: read udp ...` or `i/o timeout` for the failing domain.

**What it tells you:** an upstream `forward` target is unreachable, slow, or wrong.

**Common causes:**

- `forward . 8.8.8.8` configured but the node has no egress to 8.8.8.8 (firewall, VPC route).
- `forward . /etc/resolv.conf` and the node's `/etc/resolv.conf` is empty or points to an unreachable resolver.
- Custom forward block (`forward example.internal 10.0.0.1`) for an internal domain, but the internal resolver is down.

**Fix path:** correct the forward target in the Corefile, or fix the upstream resolver. Verify by `nslookup` to the failing domain from a pod after CoreDNS reload.

---

## Stale CoreDNS hosts / template overrides

**Confirm:**
- A specific domain resolves to a wrong/old IP address.
- The Corefile contains a `hosts { ... }` block or `template` block with stale entries.
- Real DNS for the domain has changed but the override has not.

**What it tells you:** someone added an override (intentional for testing, or a migration leftover) that's still in effect.

**Fix path:** remove or update the entry in the Corefile. This is the `stale_coredns_config` failure class.

---

## Pod dnsPolicy: Default (no cluster DNS)

**Confirm:**
- `kubectl get pod <name> -n <ns> -o yaml | grep dnsPolicy` → `dnsPolicy: Default`.
- `nslookup` from the pod for `kubernetes.default.svc.cluster.local` fails ("no such host"), while a node-local domain works.
- The pod's `/etc/resolv.conf` does **not** contain `nameserver <cluster-dns-ip>`.

**What it tells you:** the pod is using the node's DNS resolver, not CoreDNS, so internal cluster names cannot be resolved.

**Common causes:**

- Pod template was copy-pasted from a `hostNetwork: true` pod that legitimately uses `Default`.
- Someone changed `dnsPolicy` while debugging and didn't change it back.
- Helm chart values default to `Default` when they should default to `ClusterFirst`.

**Fix path:** change `dnsPolicy: Default` to `dnsPolicy: ClusterFirst` in the pod spec. This will trigger a pod restart on Deployment-owned pods. Verify with `nslookup` from a new pod. This is the `wrong_dns_policy` failure class.

---

## NetworkPolicy blocking DNS

**Confirm:**
- Pods in a namespace cannot resolve any names (internal or external).
- `kubectl get networkpolicy -n <namespace>` shows a policy with `policyTypes: [Egress]` and a `podSelector` matching the affected pods.
- The policy does NOT contain an egress rule allowing UDP/53 (and TCP/53 for large responses) to `kube-system` CoreDNS pods.

**What it tells you:** the namespace has a default-deny egress NetworkPolicy that didn't include DNS as an allowed egress.

**Fix path:** add an egress rule to the policy:

```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
  ports:
  - protocol: UDP
    port: 53
  - protocol: TCP
    port: 53
```

DNS is the most common "thing forgotten in a default-deny egress policy." Always check it.

---

## NetworkPolicy blocking pod-to-pod (non-DNS)

**Confirm:**
- DNS resolution succeeds (so it's not the DNS NetworkPolicy case above).
- A TCP connection from a source pod to a destination pod's IP fails with timeout (NOT connection refused).
- The destination namespace has a NetworkPolicy with `policyTypes: [Ingress]` and a `podSelector` matching the destination pods.
- The policy's `ingress` rules do not include the source pod's selector or namespace.

**What it tells you:** the destination pods have a default-deny ingress policy with no allow rule for this source.

**Common causes:**

- Default-deny policy added for security but forgot to allow legitimate cross-namespace traffic (monitoring, ingress controller, sidecar mesh).
- Policy `namespaceSelector` requires a specific label that the source namespace doesn't have.
- Policy `podSelector` is more restrictive than intended.

**Fix path:** add an ingress rule allowing the source. Verify from the actual source pod. This is the `network_policy_block` failure class.

> NetworkPolicy denies are silent — no events, no logs in default setups. The only signal is "TCP connection times out and nothing else explains it." Some CNIs (Calico with logging enabled, Cilium with Hubble) expose deny logs — consult cluster docs.

---

## Cross-namespace DNS works but service unreachable

**Confirm:**
- `nslookup <svc>.<other-ns>.svc.cluster.local` from your pod returns the Service's ClusterIP.
- TCP connection to that ClusterIP and port fails.
- Removing/relaxing the NetworkPolicy in the destination namespace makes it work.

**What it tells you:** DNS is fine; NetworkPolicy is blocking the actual traffic. Follow the "NetworkPolicy blocking pod-to-pod" path.

---

## DNS appears intermittent

**Confirm:**
- Some `nslookup` calls succeed, others fail.
- CoreDNS has multiple replicas; one or more is unhealthy or under-provisioned.

**What it tells you:** kube-proxy is round-robin'ing to CoreDNS replicas, and a fraction of replicas are returning errors or timing out.

**Common causes:**

- One CoreDNS replica's upstream is broken (e.g., NodeLocal DNS is being used and one node's NodeLocal cache is wedged).
- CoreDNS pods are under CPU pressure and timing out under load — `kubernetes-capacity-saturation-triage` for the CoreDNS workload.
- A NodeLocal DNS Cache deployment has a stale or misconfigured replica.

**Fix path:** identify the unhealthy CoreDNS replica(s) via logs and metrics, then triage them individually (pod-triage or capacity-triage as appropriate).

---

## Generalized pod-to-pod connectivity broken (CNI-level)

**Confirm:**
- DNS is healthy.
- NetworkPolicy is not in play (no policies in destination namespace).
- TCP from pod A to pod B's IP fails cluster-wide, across multiple workloads.
- CNI pods in `kube-system` (Calico, Cilium, Flannel) show errors or restarts.

**What it tells you:** the CNI plugin or kube-proxy is broken at the cluster level.

**Fix path:** escalate to platform/cluster operators. SRE-level triage stops here — CNI repair usually requires node-level access and cluster-wide change management.
