---
name: kubernetes-pod-triage
description: Triage and remediate a Kubernetes pod that is failing, crashing, pending, restarting, or otherwise unhealthy. Use when a user reports pod-level trouble by name, when an alert points at a specific pod, or when you have already scoped an incident down to a single pod or workload. Forces a scope check first (the first broken pod you see may not be the one causing the reported symptom — pre-existing failures are common), then walks pod status, events, logs, common failure modes (CrashLoopBackOff, ImagePullBackOff, OOMKilled, scheduling failures, probe failures, evictions), related resources, and recent changes. Produces a single primary root-cause statement with confidence and a verified mitigation. Requires a Kubernetes-capable tool (kubernetes MCP server or shell with kubectl).
---

# Kubernetes Pod Triage

Use this skill when a user names a specific pod (or a workload backing one) and reports it as broken — or when an alert payload identifies a pod by name.

You are operating as a one-shot SRE: **one diagnosis attempt, one mitigation attempt, no retries.** Being correct the first time is the only goal. Saying "I do not have enough evidence" with named hypotheses is a *good* outcome — guessing is a bad outcome, because a wrong fix on a wrong diagnosis fails both stages.

You must separate **what you observed** from **what you concluded**. Do not propose a cause before gathering evidence, and do not propose a mitigation before you can name a single primary root cause with high confidence.

## Step 0 — Scope check (do this before anything else)

The most common cause of a wrong diagnosis is **latching onto the first broken thing you see** instead of the thing causing the user's reported symptom. Pre-existing brokenness (pods that have been failing for hours or days, unrelated to the current incident) is common in real clusters. Don't fall for it.

Before investigating any pod, do this:

1. **Restate the user's symptom verbatim.** If the user said "checkout requests are returning 500," that is the symptom. "A pod is broken" is not the symptom — it is one possible *cause*.
2. **List currently-broken pods in the relevant scope** (namespace or label selector). Capability: "list pods with their status, restart counts, and age."
3. **For each broken pod, ask three questions:**
   - Does its symptom (CrashLoop, OOMKilled, etc.) plausibly cause the user-reported behavior?
   - How long has it been broken? (Compare its restart age / first-failure event timestamp to when the user-reported incident started. A pod failing for 3 days is unlikely to be the cause of a 10-minute-old outage.)
   - Is it on the request path for the failing surface? (A broken pod in an unrelated workload is not the cause.)
4. **Pick a single primary suspect.** If you cannot, list two or three candidate pods with the evidence for/against each — that is your diagnosis output. Do not pick the first one alphabetically or the one with the most restarts.

### Critical: "high restart count" is NOT the same as "this is the broken thing"

When multiple pods are broken, do not pick the one with the largest restart count. Apply this generic SRE heuristic:

- **If a pod has restart count > 0 AND its container logs show normal request processing** (real traffic being served, no panics, no startup errors in the logged-before-restart output), then **the container is not crashing on its own — the kubelet is killing it.** That means the **liveness/readiness probe configuration is the bug**, not the application.
- **If a pod has restart count > 0 AND its container logs show repeated startup errors / OOMKilled / panics / "failed to bind" / missing-dependency errors**, then the **application is crashing on its own** — the workload spec or its dependencies are the bug.

These two patterns produce the same symptom (CrashLoopBackOff with high restart count) but have **completely different root causes**. The decision between them is made by reading the logs, not by reading the restart count.

A pod with 9 restarts that is currently serving traffic successfully is **kubelet-killed**. A pod with 7 restarts whose logs show "out of memory" panics is **app-crashed**. The 9-restart pod's probe is the bug; the 7-restart pod's resource request is the bug. They are different incidents that often appear in the same cluster simultaneously, and the larger-restart-count pod is **not automatically the primary fault** — it may be the **less interesting** one if its application is otherwise healthy.

Whenever the cluster shows multiple broken pods and you find yourself reasoning "this pod has more restarts, so it's the root cause" — **stop and read the logs of the pod with FEWER restarts first**. The lower-restart-count pod often has clearer evidence of the actual injected fault.

### Mandatory: investigate ALL broken pods before committing

When the scope check identifies multiple broken pods, **you must read logs and events from ALL of them before concluding which is primary.** Do not investigate the first broken pod in depth, find any plausible cause, and stop. That is the #1 cause of wrong diagnoses.

Concretely: if `kubectl get pods` shows pods A and B both restarting, you must fetch:
- `kubectl logs <A> --previous` and `kubectl logs <B> --previous`
- `kubectl describe pod <A>` and `kubectl describe pod <B>`
- `kubectl get events --field-selector involvedObject.name=<A>` and same for B

Then compare. Only after you have evidence from every broken pod can you pick a primary suspect.

### Coverage check: your root cause must explain ALL observed anomalies

A correct root cause explains **every** anomaly you observed, not just the most prominent one. If you observe:
- Pod A is OOMKilled
- Pod B has restarts but is serving traffic
- Service C's endpoint returns 5xx

…and your proposed root cause only explains pod A — your root cause is **wrong or incomplete.** Pod B's restarts and Service C's errors must also have an explanation. If they don't, keep investigating.

The most common failure mode: picking a flashy-looking issue (e.g., "pod OOMKilled with low memory limit") that explains *some* of the symptoms, declaring victory, and ignoring symptoms it doesn't explain (e.g., pod B's restarts that happened despite no memory pressure). The unexplained symptoms are *exactly* where the actual root cause hides.

If your candidate root cause does not explain every anomaly, **explicitly write out which anomalies it does not explain** — and then either find a root cause that explains all of them, or report `unknown` with the unexplained-anomaly evidence.

If you investigate the wrong pod, the rest of this workflow will produce a confident wrong answer. Step 0 is not optional.

## Step 1 — Confirm the investigation target and gather identifying inputs

Once Step 0 has picked the primary suspect:

1. **Pod name and namespace** — explicit, not implied.
2. **Cluster context** — if multiple are accessible, name which one.
3. **Restated suspect-symptom statement** — "Pod X in namespace N is symptom Y, which I believe causes the user-reported symptom Z because [evidence]."

If any of these are missing and you cannot infer them safely, ask the user or stop and report the gap.

## Step 2 — Snapshot the pod

Fetch the pod's current state. You need:

- **Phase** (`Pending`, `Running`, `Succeeded`, `Failed`, `Unknown`)
- **Pod conditions** (`PodScheduled`, `Initialized`, `ContainersReady`, `Ready`)
- **Per-container status** for every container and init container: `state` (`waiting` / `running` / `terminated`), `reason`, `lastState`, `restartCount`, `ready`
- **Node** the pod is assigned to (if any)
- **Resource requests and limits** on each container
- **Image** for each container
- **Owner reference** (Deployment / ReplicaSet / StatefulSet / DaemonSet)

Capability: "get a pod's full status including container states and conditions." Whatever k8s tool the agent has, this is the first call.

## Step 3 — Read recent events for the pod

Events are where Kubernetes explains scheduling failures, image pull errors, probe failures, OOM kills, and evictions in human-readable form. Highest-signal single source.

Fetch all events whose `involvedObject` is the pod, sorted newest-last. Read the **`reason`** and **`message`** of each.

Common reasons and what they tell you:

| Reason | Meaning |
|---|---|
| `FailedScheduling` | The scheduler cannot place this pod. Message names the constraint. |
| `Failed` (with `ErrImagePull` / `ImagePullBackOff`) | Image cannot be pulled. Message says whether name/tag/auth/network. |
| `BackOff` | Container restarting too quickly. Pair with previous-instance logs. |
| `Unhealthy` | Liveness or readiness probe failing. Message names which probe. |
| `OOMKilling` | Container exceeded memory limit. |
| `Evicted` | Node pressure forced the pod off. Message names the resource. |
| `FailedMount` / `FailedAttachVolume` | Volume problem — missing PVC, wrong access mode, CSI driver issue. |

If events conclusively explain the symptom, jump to **Step 7 (Reporting)**.

## Step 4 — Read container logs

For each container in `waiting` or with a non-zero restart count:

- Fetch current logs (last 200–500 lines).
- **Always fetch previous-instance logs** if the container has restarted (`--previous`). The crash you care about is in the previous instance.

Read with these questions:

- Did the application log a startup error (missing env var, missing secret, can't reach a dependency, port already in use)?
- Did it log a panic, fatal, or unrecoverable error?
- Did it log nothing? (Process died before logger initialized, or the entrypoint isn't what you think.)

## Step 5 — Map symptoms to failure modes

If Steps 2–4 haven't given you a clear cause, work the failure-mode playbook. Load `references/failure-modes.md` via `read_skill_file` for the detailed decision tree on each pattern:

- **Pending** → scheduling, PVC, or image-pull issue
- **ImagePullBackOff / ErrImagePull** → image name, tag, registry, or auth
- **CrashLoopBackOff** → application-level crash on startup (logs are authoritative)
- **OOMKilled** → memory limit too low or memory leak
- **Liveness/Readiness failing** (pod `Running` but `Ready=False`) → probe misconfiguration or slow app
- **Init container failed** → dependency setup failed (logs from the init container)
- **Evicted** → node resource pressure
- **Stuck Terminating** → finalizer holding the pod
- **Running but the user says it's broken** → application-level issue, network policy, Service/Endpoints mismatch

Load `references/failure-modes.md` *only* if Steps 2–4 didn't already isolate the cause.

## Step 6 — Check related resources and recent changes (only if implicated)

Pull only the resources the failure mode implicates. Do not enumerate the namespace.

- **ReplicaSet / Deployment / StatefulSet** owning the pod — check `desired` vs `ready` vs `available`, and `conditions`.
- **ConfigMap / Secret** referenced by env vars or volumes — confirm keys exist.
- **PVC / PV** — bound, correct storage class, access mode matches.
- **Service / Endpoints** — only if "running but unreachable" is the symptom. (If yes, you may be in the wrong skill — consider `kubernetes-service-connectivity-triage`.)

A pod healthy yesterday and broken today usually correlates with a change:

- A recent **Deployment rollout** (new ReplicaSet, image tag change, env change).
- A recent **ConfigMap or Secret** update — these don't trigger rollouts by default, so a running pod may be fine while new pods crash.
- A recent **node event** (drain, cordon, autoscaler scale-down).

## Step 7 — Reporting (with root-cause discipline)

### Diagnostic closure: when you have the answer, COMMIT to it

The single most common reason diagnoses fail evaluation is that the agent **keeps investigating after it already has the answer**, runs out of turns, and submits an in-progress note instead of a verdict.

**You have "the answer" when:**
- You can name a specific resource (Deployment / Pod / Service / ConfigMap) AND a specific configuration value or state (e.g., "Deployment `frontend` has liveness probe `failureThreshold: 1` with HTTP path `/healthz` that returns 404"), AND
- That configuration fully explains every observed anomaly (coverage check above passed), AND
- You have at least one piece of direct evidence (`kubectl describe` output, log line, event message) supporting it.

**The moment you have the answer, stop investigating and write the report.** Do not do "one more check" — that's how you exhaust your budget and submit a partial answer. Do not say "Let me also verify…" if you already have the verdict. **Investigation ends. Report begins.**

If you submit an in-progress note ("I found X, let me check Y") instead of the structured report below, your diagnosis is **incomplete** and will fail evaluation regardless of what you found.

### Report format — write this VERBATIM as your submission

A diagnosis is one primary root cause statement, not a list. Resist the urge to enumerate every anomaly you observed — that is exactly what gets penalized.

Conclude with a report in this exact shape:

```
Pod: <name> in <namespace> on <cluster/context>
User-reported symptom: <verbatim or one-sentence restatement>
Observed pod symptom: <one sentence>

Findings (evidence only — no interpretation):
- <what you saw, with the call that produced it>
- <...>

Primary root cause: <one sentence — the single fault you will mitigate>
Confidence: high | medium | low
Why this is the root cause: <evidence ruling it in>
Why it is not [next-most-likely alternative]: <evidence ruling that out>

Other anomalies observed (NOT root causes):
- <pre-existing failure that is incidental> — reason it is not causal: <evidence>
- <downstream impact you can see in metrics/logs> — reason it is not a separate root cause: <evidence>
- (If none, write "none.")

Recommended mitigation: <see Step 8>
```

### Root-cause discipline rules

- **Single primary fault.** Report more than one root cause only if you can demonstrate they share a cause or are linked by evidence. If a CrashLoopBackOff pod and an OOMKilled pod are both broken, that is **two anomalies**, not two root causes. One of them is the cause of the reported symptom; the other is incidental, downstream, or pre-existing.
- **Pre-existing brokenness is not a root cause.** If a pod has been broken for days and the incident is minutes old, it is not the cause.
- **Downstream impact is not a root cause.** "Service B is erroring because Service A is down" — the root cause is A's outage. B's errors go under "Other anomalies."
- **If confidence is low, the primary root cause field is `unknown`.** Then list two or three competing hypotheses, with the evidence that would distinguish them, in a numbered list. This is a legitimate diagnosis output — do not invent certainty.

## Step 8 — Mitigation (pre-flight + post-fix verification)

**Under 1-shot conditions, a wrong fix on a wrong diagnosis fails both stages.** If your diagnosis confidence is `low`, do not mitigate. Submit the diagnosis with hypotheses and stop.

If confidence is `high` (or `medium` with explicit risk acceptance), before applying any change, write a pre-flight block:

```
Mitigation pre-flight:
  Diagnosis I am acting on: <restatement of the primary root cause>
  Smallest change that resolves it: <one specific edit — patch a probe, fix a configmap key, change an image tag, raise a limit. Reject scope creep.>
  Predicted post-fix state: <what you expect to observe — specific resource fields, event reasons, log lines, metric values>
  Verification calls: <the specific read-only calls you will make after the fix>
  Rollback if predicted state ≠ actual: <the inverse edit, exactly>
```

Then apply the change. Use the read-only verification calls. Compare actual to predicted.

```
Post-fix verification:
  Actual post-fix state: <what you observed>
  Match to predicted state: yes | no | partial
  If no/partial: STOP. The diagnosis was wrong. Apply the rollback, report the new evidence, do not attempt a second fix.
  If yes: mitigation succeeded. Final report.
```

### Mitigation guardrails

- **Do not chain attempts.** If the first fix did not produce the predicted state, the diagnosis was wrong. A second fix on a wrong diagnosis compounds the error.
- **Smallest change wins.** Fix the one thing. Do not bundle "while I'm in here" cleanups.
- **Read-only investigation does not need pre-flight.** Only mutating calls do.
- **If the operator has configured HITL approval gates**, mutating calls go through them. Do not work around them.

## When to escalate vs continue

Escalate to a human and stop without mitigating when:

- Confidence is `low` after the full workflow.
- The failure is a symptom of a cluster-wide or node-level problem (multiple unrelated workloads failing the same way, node `NotReady`, control-plane errors).
- The fix would risk persistent data loss.
- Step 0's scope check could not isolate a single primary suspect.

Otherwise, continue and resolve.

## Skill resources

Load on demand with `read_skill_file`:

- `references/failure-modes.md` — detailed decision tree for each pod failure pattern.
- `references/tool-mapping.md` — capability-to-tool mapping for common kubernetes MCP servers and shell kubectl.
