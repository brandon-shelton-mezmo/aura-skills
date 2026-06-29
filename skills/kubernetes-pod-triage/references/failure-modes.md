# Pod Failure Modes — Decision Tree

For each pattern: how to confirm it, what causes it, what to check next, and how to fix.

---

## Pending

**Confirm:** `pod.status.phase == "Pending"` and `PodScheduled` condition is `False`, or `ContainersReady` is `False` with containers in `waiting`.

**Cause it tells you about:** the scheduler cannot place the pod, or the kubelet cannot start its containers.

**Distinguish:**

- `PodScheduled=False` → scheduling problem. Read the `FailedScheduling` event message.
- `PodScheduled=True` but containers in `waiting` with `ContainerCreating` → kubelet is trying to start them. Check for volume mount errors (`FailedMount`), image pulls (`Pulling`), or sandbox creation errors (`FailedCreatePodSandBox` — usually a CNI problem).

**Common scheduling causes from the event message:**

- `0/N nodes are available: insufficient cpu` / `insufficient memory` → resource requests exceed available capacity. Either lower requests, scale the node group, or wait for autoscaler.
- `node(s) had taints that the pod didn't tolerate` → pod is missing a toleration for a tainted node pool (often a dedicated GPU/spot pool).
- `node(s) didn't match Pod's node affinity/selector` → `nodeSelector` or `nodeAffinity` excludes every node. Often a stale `nodeSelector` referencing a node label that no longer exists.
- `pod has unbound immediate PersistentVolumeClaims` → the PVC referenced by the pod has no PV bound. Check the PVC's status and the storage class's provisioner.

**Fix path:** address the constraint named in the message. Don't blindly add tolerations or remove selectors — they're usually there on purpose.

---

## ImagePullBackOff / ErrImagePull

**Confirm:** container `state.waiting.reason` is `ImagePullBackOff` or `ErrImagePull`. Event `Failed` with the same reason.

**Distinguish from the event message:**

- `manifest unknown` / `not found` → image name or tag is wrong (typo, tag was deleted, wrong registry).
- `unauthorized` / `denied` → registry authentication failure. The pod's `imagePullSecrets` (or the node's docker config) doesn't have valid credentials for this registry.
- `dial tcp ... timeout` / `no such host` → network or DNS problem reaching the registry. Less common; check the node's egress and any private-registry config.

**Fix path:**

- Wrong image → fix the image reference in the workload spec, redeploy.
- Auth → create or fix `imagePullSecrets`, attach to the pod's ServiceAccount.
- Network → out of scope of pod triage; escalate to cluster networking.

---

## CrashLoopBackOff

**Confirm:** container `state.waiting.reason == "CrashLoopBackOff"`, `restartCount > 0`, and `lastState.terminated.reason` is typically `Error` (exit code != 0) or `OOMKilled`.

**Logs are authoritative.** Read previous-instance logs (`--previous`). The current instance hasn't started yet — its logs are empty or just the boot banner.

**Common patterns in the previous logs:**

- Application logs a missing env var or unreadable file → the pod references a Secret/ConfigMap key that doesn't exist, or the volume isn't mounted. Check the workload spec against the actual ConfigMap/Secret keys.
- Application logs "address already in use" → two containers in the pod fighting over the same port, or `hostPort` collision on the node.
- Application logs a dependency connection failure (database, queue, internal service) → the dependency is unreachable. If it's the dependency that's broken, that pod becomes the new triage target.
- Application logs nothing → the process is dying before the logger initializes, or the container's entrypoint isn't what you think (e.g., shell script `exec`s wrong binary). Check the image's actual entrypoint vs. the workload's `command`/`args`.
- `lastState.terminated.reason == "OOMKilled"` → see **OOMKilled** below; CrashLoopBackOff is the symptom, OOMKilled is the cause.

**Fix path:** fix the application config / dependency / spec. Restarting the pod will not help — Kubernetes is already restarting it.

---

## OOMKilled

**Confirm:** `lastState.terminated.reason == "OOMKilled"`, exit code 137. Event `OOMKilling` from the kubelet.

**Distinguish two cases:**

1. **Memory limit too low for legitimate working set** → the app's normal memory usage exceeds `resources.limits.memory`. Check the workload's actual usage over time (metrics-server / Prometheus) and raise the limit to comfortably above peak.
2. **Memory leak** → usage grows monotonically until OOM. Raising the limit only delays the next OOM. Capture a heap dump (language-specific) and hand off to the application team.

**JVM gotcha:** Java apps that don't set `-XX:+UseContainerSupport` (default on modern JDKs, but not all images) read host memory instead of the cgroup limit and set heap accordingly. Symptom: OOMKilled with a heap that "looks fine" to the JVM. Fix the JVM flags or move to a `-jdk-slim` image with sane defaults.

---

## Liveness / Readiness Probe Failing

**Confirm:** pod is `Running` but `Ready=False`. Event `Unhealthy` from the kubelet, with message naming `Liveness probe failed` or `Readiness probe failed`.

**Liveness failures cause restarts.** If you see `Liveness probe failed` repeating and `restartCount` climbing, the probe is killing a healthy-looking app.

**Common causes:**

- Probe `initialDelaySeconds` too short → app is still booting when the first probe runs, gets killed, never makes it past startup. Either raise `initialDelaySeconds`, or move slow startup checks to a `startupProbe` so liveness only starts after startup succeeds.
- Probe path or port wrong → 404 from the app or connection refused. Confirm the probe targets the actual health endpoint and port the app serves.
- Probe times out (`timeoutSeconds` too low) → app responds correctly but takes longer than the timeout. Either raise `timeoutSeconds` or fix the slow handler.
- Probe is succeeding from your shell but failing for the kubelet → probe runs from the node, not from inside the cluster network. If the probe targets an external dependency (it shouldn't), that's the bug.

**Fix path:** correct the probe spec. Do not delete the pod — the next one will fail the same way.

---

## Init Container Failed

**Confirm:** pod `Initialized=False`, init container in `waiting` with `reason: PodInitializing` or `terminated` with non-zero exit.

The pod cannot start its main containers until every init container exits 0.

**Fix path:** read the failed init container's logs (`kubectl logs <pod> -c <init-container-name>`). Init containers typically wait for a dependency, populate a volume, or run a migration. The failure is usually "dependency not ready yet" (transient) or "dependency misconfigured" (needs a real fix).

---

## Evicted

**Confirm:** pod phase `Failed`, `reason: Evicted`. Event from the kubelet naming the resource (`The node was low on resource: ephemeral-storage`, `memory`, etc.).

**What it means:** the node ran out of a resource and the kubelet's eviction policy targeted this pod (usually because it had no resource requests or was over its requests).

**Fix path:**

- Set or raise resource requests so the pod gets a more favorable QoS class (`Guaranteed` > `Burstable` > `BestEffort`).
- If it's ephemeral storage, check what the pod is writing (logs, temp files) and either bound it or mount a real volume.
- If it's a node-level capacity problem (many pods evicted, not just this one), escalate to capacity planning.

Evicted pods stay around in `Failed` state. Delete them after the underlying fix; they don't auto-clean.

---

## Stuck Terminating

**Confirm:** pod has a `deletionTimestamp` set but is still present after well past `terminationGracePeriodSeconds`.

**Causes:**

- A **finalizer** on the pod (or a controller's finalizer) is blocking deletion. Check `metadata.finalizers`. Common culprits: a CSI driver removing a volume, a service mesh sidecar cleanup, a custom operator. Never remove finalizers manually unless you understand what they're cleaning up — you can orphan real resources.
- The node hosting the pod is **NotReady** — the kubelet isn't around to confirm the pod stopped. Pod is force-deletable with `--grace-period=0 --force` *after* you've confirmed the node really is gone and the workload is safe to detach (StatefulSets in particular).

**Fix path:** identify the blocking finalizer, fix the underlying cleanup it's waiting on. Force-delete only as a last resort and with the user's explicit go-ahead.

---

## Running but the User Says It's Broken

The pod is `Ready=True`, restart count is stable, no recent events — but the user reports it's not working. The symptom is downstream of the pod.

**Investigation order:**

1. **Application-level errors in current logs.** The app may be returning 5xx, throwing exceptions, or logging warnings. Read recent logs with the symptom in mind.
2. **Service / Endpoints mismatch.** Get the Service the pod is supposed to back, get its Endpoints. If the Endpoints list is empty or doesn't include this pod's IP, the Service selector doesn't match the pod's labels. (Recent label change is a classic cause.)
3. **NetworkPolicy.** If a NetworkPolicy exists in the namespace, confirm it allows traffic to/from this pod. A default-deny policy with no matching allow rule looks like "pod is up but unreachable."
4. **Resource saturation under load.** CPU throttling (`container_cpu_cfs_throttled_periods_total` in Prometheus) shows up as slow responses without errors. Memory pressure short of OOM causes GC thrash.
5. **Upstream dependency.** The pod is healthy but a database/queue/API it depends on is degraded. Pod triage hands off to whatever owns that dependency.

---

## Multiple Pods of the Same Workload Failing the Same Way

This is no longer pod triage — it's a workload or platform issue. Tell the user, and re-scope to:

- **Same node?** Node problem. Cordon and investigate the node.
- **Same image / config?** Bad rollout. Roll back.
- **Across nodes and unrelated workloads?** Cluster-level (CNI, DNS, control plane). Escalate.
