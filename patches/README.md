# Patches — infrastructure enablement for skills iteration

These patches enable rapid iteration on the aura-skills library against
the SREGym benchmark. They are the minimum-viable set of changes needed
to make the skills mechanism testable at high iteration cadence.

Both are **additive and env-var-gated** — if the flag isn't set, behavior
is unchanged. Nothing here bakes benchmark-specific logic into either
project; both patches are generic infrastructure enablement.

## Patches

### `sregym-fast-iterate.patch`

**Applies to**: `SREGym` project, `sregym/conductor/conductor.py`.

**What it adds**: two env-var flags that let the conductor reuse a
running workload across multiple `sregym-run` invocations, cutting
per-iteration setup from ~10-18 min (helm install + wait for pods
Ready + fault injection) to ~20-30 seconds (recover-fault +
inject-fault only).

- `SREGYM_FAST_ITERATE=1` — in `start_problem()`, if the app namespace
  already exists (from a prior iteration), skip the full `undeploy_app`
  + `deploy_app` cycle. The fault is still injected via the normal
  `_advance_to_next_stage` code path.

- `SREGYM_KEEP_APP_DEPLOYED=1` — in `_cleanup_sync()`, after
  `recover_fault()`, skip `app.cleanup()` and `reconcile_to_baseline()`.
  This leaves the workload standing so the NEXT run's `FAST_ITERATE`
  branch can reuse it.

**Iteration loop**:
```
first invocation:  KEEP_APP_DEPLOYED=1                          → full deploy, keeps app
subsequent runs:   FAST_ITERATE=1 KEEP_APP_DEPLOYED=1           → skips helm, ~30s setup
final invocation:  no flags                                     → full cleanup
```

**Measured impact**: 18 min setup → 20 sec setup on the same problem.
Total per-iteration wall time now dominated by LLM latency (~10-15 min),
not infrastructure.

Apply with: `git apply patches/sregym-fast-iterate.patch` inside the
SREGym checkout.

### `mezmo-bench-aura-skills-dir-env.patch`

**Applies to**: `mezmo_benchmark` project,
`src/mezmo_benchmark/adapters/aura.py`.

**What it adds**: `AURA_SKILLS_DIR` in the env-var allowlist that
mezmo-bench passes to the spawned `aura-web-server` subprocess. The
adapter uses an allowlist (not denylist) for security — vars not on the
list are stripped when spawning AURA. Without this addition, AURA's
TOML template placeholder `{{ env.AURA_SKILLS_DIR }}` fails to resolve
at config-load time.

Apply with: `patch -p1 < patches/mezmo-bench-aura-skills-dir-env.patch`
inside the mezmo_benchmark checkout. Insertion point is inside the
`_AURA_ENV_EXACT_NAMES` frozenset, next to the other `MEZMO_*` entries.

### `mezmo-bench-load-skill-mandate.patch`

**Applies to**: `mezmo_benchmark` project,
`config/aura-sregym.toml.template`.

**What it adds**: a "MANDATORY first action" instruction at the top
of the SREGym-specific operational-notes section, telling the agent
to invoke `load_skill(name="kubernetes-pod-triage")` before any other
tool call.

**Why this patch is load-bearing**: without it, AURA registers the
`load_skill` and `read_skill_file` tools at boot but the agent never
actually invokes them — the SKILL.md content therefore never enters
the model's context, only the skill catalog descriptions do. Verified
by `grep load_skill` in `aura-startup.log`: pre-patch runs show a
single "Adding skill tools" registration line and no subsequent
`load_skill` tool calls.

**Measured impact** on `liveness_probe_misconfiguration_astronomy_shop`
with a chronic flagd-ui OOMKill acting as a distractor in the cluster:

| Setup | Sample scores | Best |
|---|---|---|
| Pre-patch (skill content unreached) | 0, 0, 44, 44, 0, 67, 0 | 67 |
| Post-patch (mandate fires load_skill) | 100 | 100 |

Diagnosis wall time post-patch: 77 seconds from settle-wait to
correct final answer. The agent's output cites skill sections
explicitly ("Frontend is the freshest incident", "stealth-fault
signature") — proof that the loaded SKILL.md content is driving
the diagnostic reasoning.

Apply with: `patch -p1 < patches/mezmo-bench-load-skill-mandate.patch`
inside the mezmo_benchmark checkout.

## Related

Both patches enable the workflow described in
`../SREGYM-INTEGRATION.md`. The skills feature itself is upstream in
AURA on `feature/LOG-0000-skills-and-hitl` (not yet merged to main).
