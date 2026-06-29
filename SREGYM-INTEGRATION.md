# Wiring aura-skills into the SREGym benchmark

This document describes the one-line config change that mounts this skill
library into AURA-on-SREGym, plus the prerequisites and expected behavior.

## What changes

A single TOML stanza, added to
`reproduction/configs/aura-sregym.toml.template` in your
`aura-sregym-benchmark-results` (or wherever the benchmark runner renders
it from). Place it after the existing `[agent]` block and before `[agent.llm]`.

```toml
# Skill library — on-demand SRE instructions. The catalog of skill
# names + descriptions is auto-appended to the agent's system prompt;
# the agent calls load_skill(name) to fetch a skill's full SKILL.md
# body on demand, and read_skill_file(skill, path) to pull in
# reference files when the workflow tells it to.
#
# This intervention is purely additive: SREGym's user-message prompt
# (Stratus's verbatim text) is unchanged, the MCP tool catalog is
# unchanged, the system_prompt above is unchanged. Only the appended
# skill catalog and the load_skill / read_skill_file tools are added.
[[agent.skills.local]]
source = "{{ env.AURA_SKILLS_DIR }}"
```

Then export the env var when launching the SREGym runner so it gets
injected at config-render time:

```bash
export AURA_SKILLS_DIR="/Users/brandon.shelton/Documents/GitHub/aura-skills/skills"
```

(Or whatever path the benchmark runner has access to — if the SREGym
worker runs in a separate VM, the skills directory must be mounted/copied
into that VM and `AURA_SKILLS_DIR` set to its in-VM path.)

## Prerequisites

1. **Build AURA from the skills branch.** The skills feature is on
   `feature/LOG-0000-skills-and-hitl` and not yet on `main`. The
   benchmark runner's `--aura-bin` flag must point at a binary built
   from this branch:
   ```bash
   cd ~/Documents/GitHub/aura
   git checkout feature/LOG-0000-skills-and-hitl
   cargo build --release -p aura-web-server --bin aura-web-server
   # binary: target/release/aura-web-server
   ```
   Then pass `--aura-bin target/release/aura-web-server` to
   `mezmo-bench sregym-run` (or whatever the runner CLI looks like).

2. **Confirm AURA's skill feature is reachable in the runner's spawn
   args.** The runner already passes the rendered TOML; the
   `[[agent.skills.local]]` block is loaded by the same TOML parser
   that loads the rest, so no Python-side changes should be needed.

## What this changes about agent behavior

Before:
- Stratus prompt (verbatim) tells the agent how to diagnose / mitigate.
- AURA passes that prompt + the MCP tool catalog to the model.
- Model investigates and submits.

After (with skills mounted):
- All of the above, **plus** the agent's system prompt has an additional
  block appended:
  ```
  Available skills (use the `load_skill` tool to load before answering):
  - kubernetes-pod-triage: <description>
  - kubernetes-service-connectivity-triage: <description>
  - kubernetes-network-triage: <description>
  - kubernetes-capacity-saturation-triage: <description>
  ```
- The agent has two extra tools available: `load_skill` and `read_skill_file`.
- The model decides on its own whether to call them. If the user-message
  prompt (Stratus) describes a Kubernetes pod problem, the catalog
  description for `kubernetes-pod-triage` should match strongly enough that
  the model chooses to load it.

## Recommended initial experiment

Three runs on the same 5-problem subset (suggested: a mix from the
0.0-composite list — one each from probe, service-routing, network,
saturation, and pod failure-mode classes):

| Run | retry config | skills mounted | What it measures |
|---|---|---|---|
| A | current 10-shot mitigation | no | baseline reproduction |
| B | 1-shot mitigation | no | 1-shot floor without skills |
| C | 1-shot mitigation | yes | skills' contribution under leaderboard-strict rules |

The Run C − Run B delta is the direct measurement of skill value.
Run B − Run A measures the cost of dropping retries.
Run C − Run A is the "did we end up better than where we started, while
playing by the strict rules" story.

Approximate cost based on prior runs (~$3 per problem in Bedrock):
$45 for the experiment. Cheap enough to run multiple times if results
are surprising.

## What success looks like

For Run C, the target shape:

- **Diagnosis pass rate**: meaningfully above Run B's. The skills'
  scope-check and root-cause-discipline sections are specifically
  designed to address the "over-report" failure mode that caused 17
  of 67 problems to score 0.0 composite in the original run9.
- **Mitigation pass rate**: at minimum 70%+ to be competitive with
  the leaderboard's strict-rules entries. (Stratus's 78.5% is the
  number to beat for legitimacy.)
- **E2E pass rate**: this is the headline. Today's E2E is 41.8% on
  10-shot mitigation. The goal under 1-shot + skills is to beat
  Claude Code's 60.7%.

If diagnosis improves significantly but mitigation drops sharply, that
means the diagnoses are now correct but the agent can't translate them
into correct kubectl actions — the skills' Mitigation pre-flight
sections need to be tightened. We'd iterate on that with a small
follow-up batch rather than going wide.

If neither improves, the skills aren't earning their token cost and
we rethink the design rather than write more skills.

## Tool-agnostic invariant

The skills do not name any specific MCP tool. They describe
**capabilities** ("get a pod's full status", "list NetworkPolicy in a
namespace") and let the agent map those to whatever tools SREGym has
exposed — `sregym_kubectl.exec_read_only_kubectl_cmd`,
`sregym_prometheus.get_metrics`, etc. This is by design: the same skill
library works against any AURA deployment with any kubernetes/metrics
MCP server, not just SREGym.

The `references/tool-mapping.md` files in each skill include an
explicit SREGym section showing the namespaced tool calls — these are
informational, not normative. The agent can choose to use them or not.

## Files this PR touches in the benchmark repo

If you adopt this:

1. `reproduction/configs/aura-sregym.toml.template` — add the
   `[[agent.skills.local]]` block.
2. `reproduction/scripts/launch-parallel-sregym.sh` (or wherever the
   env vars are set) — set `AURA_SKILLS_DIR` to the path of the
   skills directory on the worker host.
3. (Optional) `reproduction/scripts/bootstrap-sregym-kind.sh` — if the
   skills directory needs to be copied into the worker VM as part of
   bootstrap.

No changes needed in `mezmo-benchmark` Python code — the skills feature
is a config-driven AURA capability, transparent to the adapter.
