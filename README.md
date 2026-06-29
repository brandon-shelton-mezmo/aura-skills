# aura-skills

A community library of [Agent Skills](https://agentskills.io/specification)
for [Aura](https://github.com/mezmo/aura), focused on **SRE workflows**.

Skills package task-specific instructions that an Aura agent loads on demand —
keeping the base system prompt small while giving the agent deep, reviewed
playbooks for the situations it actually encounters.

## Using these skills

Point any Aura agent at the `skills/` directory in your config:

```toml
[[agent.skills.local]]
source = "/path/to/aura-skills/skills"
```

Aura will discover every skill in that directory at agent build time and
append a `name + description` catalog to the system prompt. The agent calls
`load_skill(name)` to pull the full instructions only when it needs them,
and `read_skill_file(skill, path)` to fetch supporting references.

In orchestration mode, the same line under `[orchestration.worker.<name>.skills]`
scopes a skill set to a single worker; an explicit empty list
(`skills.local = []`) disables skills for that worker.

> **Status:** Aura's skills feature is on the
> [`feature/LOG-0000-skills-and-hitl`](https://github.com/mezmo/aura/tree/feature/LOG-0000-skills-and-hitl)
> branch and not yet merged to `main`. These skills work today against that
> branch; they will work against `main` once the feature ships.

## Design principles

These skills are written to be **tool-agnostic**:

- The skill body talks in **capability** terms ("fetch the pod's recent events"),
  not specific tool calls. The agent maps abstract steps to whatever MCP servers
  or shell tools are in its toolbox at runtime.
- Each skill's `description` declares the **capability it requires** (e.g.,
  "Requires a Kubernetes-capable tool") so operators know what to wire up and
  the agent knows when the skill applies.
- A `references/tool-mapping.md` file lists known-good MCP servers and shell
  equivalents for the skill's domain — a crib sheet, not a hard dependency.

A skill in this repo should work for **any** Aura user who has the listed
capability available, regardless of which MCP server or shell access they're
using to provide it.

## Repository layout

```
aura-skills/
├── README.md                              ← this file
└── skills/                                ← point [[agent.skills.local]].source here
    └── <skill-name>/
        ├── SKILL.md                       ← required: frontmatter (name, description) + workflow
        └── references/                    ← optional: deep-dive playbooks, tool mappings
            ├── ...
            └── tool-mapping.md
```

## Available skills

All four current skills are designed for **one-shot SRE work**: one diagnosis, one mitigation, no retries. Each bakes in the same discipline:
- Step 0 scope check (don't latch onto the first broken thing).
- Evidence-only findings, then a single primary root cause.
- Mitigation pre-flight + post-fix verification (no chained fixes on a refuted diagnosis).

| Skill | What it does | Capability required |
|---|---|---|
| [`kubernetes-pod-triage`](skills/kubernetes-pod-triage/SKILL.md) | Triage and remediate an unhealthy Kubernetes pod (CrashLoopBackOff, OOMKilled, Pending, probe failures, ImagePullBackOff, evictions, stuck terminating). | Kubernetes (MCP server or shell with kubectl) |
| [`kubernetes-service-connectivity-triage`](skills/kubernetes-service-connectivity-triage/SKILL.md) | Triage Service-layer connectivity failures — pods healthy but requests fail. Covers selector/label drift, target port mismatch, Ingress backend errors, NetworkPolicy denial. | Kubernetes |
| [`kubernetes-network-triage`](skills/kubernetes-network-triage/SKILL.md) | Triage cluster-network failures — DNS resolution, CoreDNS health, NetworkPolicy egress/ingress denials, pod `dnsPolicy` misconfiguration. | Kubernetes (with pod exec for DNS testing) |
| [`kubernetes-capacity-saturation-triage`](skills/kubernetes-capacity-saturation-triage/SKILL.md) | Triage capacity/saturation issues — CPU throttling, memory pressure, GC pause, slow startup, load-shape problems. Pods are healthy but the workload can't keep up. | Kubernetes + metrics (Prometheus or equivalent) |

The four skills are designed to **route between each other** — each skill's Step 0 scope check explicitly lists when to load a sibling skill instead. Together they cover the dominant SRE diagnosis classes that appear in real incidents and in benchmarks like SREGym.

## Contributing a skill

1. Create `skills/<your-skill>/SKILL.md` with the [Agent Skills frontmatter](https://agentskills.io/specification):
   ```markdown
   ---
   name: <your-skill>        # must match the directory name
   description: <1-1024 chars; what it does, when to use it, and what capability it needs>
   ---
   ```
   `name` must be 1–64 chars, lowercase alphanumeric and hyphens only, no
   leading/trailing/consecutive hyphens.
2. Write the workflow as **capability-talk**, not tool-talk. If you need to
   reference a specific tool, do it as an example (`e.g., the kubernetes MCP
   server's pods_get tool, or kubectl get pod`).
3. Optionally add a `references/` subdirectory with deep-dive playbooks and a
   `tool-mapping.md` for known capability-to-tool mappings.
4. Keep total skill content well under 512 KB — Aura warns above that
   threshold because skills load fully into the LLM's context when invoked.
