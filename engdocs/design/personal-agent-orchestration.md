# Personal Agent Orchestration on Gas City — Design

Status: design agreed, not yet implemented.
Scope: this fork's deployment design for using Gas City as a
general-purpose orchestrator for non-coding Claude Code agents,
alongside Gas Town for coding work and homelab operations.
Baseline: upstream `gastownhall/gascity` @ `91c83bc3` (2026-06-10).
The fork was ~3,550 commits behind upstream when this design was
written; the working branch was fast-forwarded first. All file
references below are against that baseline.

## 1. Goal

Use Gas City to orchestrate fleets of Claude Code agents that are not
coding agents: productivity/task management, knowledge management and
proactive research, job-search automation (discover → evaluate → draft
applications), plus homelab infrastructure operations. Every agent is a
Claude Code instance with a distinct configuration of instructions
(prompt templates), skills, and MCP servers. Coding work continues via
the Gas Town pack.

## 2. Evaluation verdict

Gas City fits. The SDK honors "zero hardcoded roles" — nothing in the
infrastructure requires git, code, or specific roles:

- A rig is a plain directory; the git check is a warning, not an error
  (`internal/doctor/checks.go` RigGitCheck). Worktree isolation is
  opt-in. Only formula `merge` directives probe git, and only if used.
- Claude Code is a first-class provider: prompt templates, per-agent
  `option_defaults` (model, permission_mode), managed
  `.claude/settings.json` via `install_agent_hooks = ["claude"]` with
  JSON-aware merging, per-agent skills and MCP catalogs, lifecycle
  controls (`one_shot`, `idle_timeout`, `max_session_age`).
- Proactivity is built in: orders with four trigger types — `cron`,
  `cooldown`, `condition` (shell exit code), `event` (event-bus types
  like `bead.closed`, `extmsg.inbound`) — with dedupe gates (an order
  with open work does not re-dispatch). See
  `internal/orders/triggers.go`, `docs/tutorials/07-orders.md`.
- External systems reach in via the HTTP+SSE API: `POST /v0/sling`,
  bead CRUD, manual order runs, `POST /v0/extmsg/inbound/...`,
  SSE event stream.
- Reliability comes from the bead store + patrol loop: work survives
  session crashes; the controller reconciles on a ~10s tick
  (Nondeterministic Idempotence).

Gaps to cover ourselves (section 9): external pollers, budget
enforcement, a knowledge substrate, all prompts (examples are
coding-flavored), upstream version pinning.

## 3. Topology: two cities

```
~/hq/        # personal city: productivity, knowledge, jobsearch rigs + own packs
~/gastown/   # engineering city: gastown pack as shipped; coding repos + homelab as rigs
```

### Decision: separate cities for personal vs engineering work

Coexistence in one city is technically supported — per-rig pack imports
(`[rigs.imports.<name>]`) bind gastown's rig-scoped agents only to
coding rigs; bead `issue_prefix` namespaces isolate work; the
maintenance pack's scripts skip non-git rigs (`prune-branches.sh`
checks for `.git`). Rejected anyway, because:

1. **The mayor is city-scoped by design** ("global coordinator … above
   all rigs"). In a mixed city it would see and could act on personal
   beads. Constraining it means patching the pack — permanent merge
   friction against a fast-moving upstream.
2. **Blast radius.** Coding swarms are high-churn/high-spend; a spawn
   storm or controller incident must not stall personal pipelines.
3. **Coordinator collision.** The personal city will grow its own
   city-scoped triage/overseer agent; two city-scope coordinators
   reacting to the same events is asking for crossed wires.
4. **Cost of separation is low.** A city is a directory; the second one
   is one more controller + Dolt instance. Cross-city handoff, when
   ever needed, goes through the other city's HTTP API
   (`POST /v0/sling`).

### Personal city layout

- One rig per life-domain (`productivity/`, `knowledge/`,
  `jobsearch/`), one pack per domain carrying its agents, prompts,
  skills, and MCP definitions.
- The `knowledge/` rig is a git repo of markdown — the knowledge base
  itself. Beads track work *about* knowledge; files hold the knowledge.
- Agents are **ephemeral by default**: `one_shot` / on-demand, spawned
  by orders, dying when their bead closes. Fresh context per task, no
  idle burn, trivial crash recovery. Long-lived sessions only for (at
  most) a triage agent per domain.
- Runtime: default tmux provider on an always-on box. `tmux attach` for
  live observation is invaluable while tuning prompts. K8s exists if
  ever needed; don't start there.
- Behaviors are order + formula pairs, e.g.: 7am cron "plan the day";
  6h-cooldown research sweep; `bead.created` event order dispatching a
  job-posting evaluator per candidate.

### Rejected alternatives

- **Hand-rolled cron + `claude -p`**: fine for 1–2 periodic jobs; loses
  persistent task state w/ dependencies, dedupe, crash recovery,
  messaging. Crossover at >2 agents or any multi-step pipeline.
- **One mega-agent**: loses per-agent MCP/permission scoping (job-search
  agent must not hold calendar credentials, etc.); giant prompts
  degrade.
- **City per domain**: triples ops footprint, loses cross-domain mail;
  rig-level prefix isolation suffices.
- **All long-lived agents**: context drift + idle cost; fights the
  bead-centric reliability model.

## 4. Homelab: lives in the engineering city

The homelab is IaC (terraform + ansible, deployed from GitHub). The
remediation path *is* the deployment path, so ops and coding belong in
the same city. Adding a self-authored SRE pack to the gastown city is
composition, not modification — the gastown pack stays stock.

- **Rig**: clone of the homelab repo, `gc rig add`. Gastown's
  rig-scoped agents stamp it → coding improvements work immediately.
- **SRE pack agents** (both ephemeral):
  - `medic` (reactive): one-shot, spawned per alert. Prompt = runbooks.
    Rule: fix symptoms directly when safe; if structural, file an
    `hl-` bead describing the IaC change (picked up by the normal
    polecat → PR flow) and close the wisp with notes.
  - `steward` (proactive): cron. Checks updates (apt, images, provider
    versions). Codify, don't mutate: routine safe updates applied via
    playbook; anything version-pinned in the repo becomes a bump PR.
- **Alert ingestion**: start with a `condition`-trigger order polling
  the Alertmanager API (~1 min; dedupe gate = storm protection).
  Upgrade later to Alertmanager webhook →
  `POST /v0/extmsg/inbound/alertmanager/...` → `event` order.
- **Tools**: ssh/ansible/terraform are CLIs — most SRE agents need no
  MCP at all. Credentials scoped per agent `env`, never in polecat
  worktrees.
- **Tiered autonomy**:
  - (a) auto-execute: cleanups, service restarts, routine updates;
  - (b) auto-PR, human merge: any IaC change;
  - (c) mail-gated (section 7): reboots, stateful `terraform apply`,
    destructive ops.
- **Recursion caveat**: the box running the controllers is itself
  homelab infra. Exclude it from auto-remediation; tier (c) only.
- **Audit**: medic prompt requires "write what you did and why to the
  bead before closing" — beads + event log + attachable tmux = incident
  log.

## 5. Skills

Skills are standard Claude Code skills (directory + `SKILL.md`); Gas
City catalogs and distributes them (`internal/materialize/skills.go`).

- **Scopes & precedence** (highest wins): agent-local
  `agents/<name>/skills/` > city pack `skills/` > imported pack
  `skills/` (namespaced `<binding>.<name>`) > bootstrap core pack
  (ships `gc-work`, `gc-dispatch`, `gc-mail`, … — how agents learn the
  `gc` CLI).
- **Materialization**: supervisor symlinks the effective set into the
  provider sink (`.claude/skills/<name>`) every patrol tick;
  per-session worktrees get a PreStart pass. Idempotent and
  ownership-aware: only gc-managed symlinks are pruned/replaced;
  user-placed files coexist. Editing a pack skill updates every agent
  immediately (symlinks).
- The `skills = [...]` field in `agent.toml` is a tombstone (ignored
  since v0.15.1). Directories only.
- Pattern: shared runbook skills at pack level; sensitive/specialized
  skills agent-local (e.g. `agents/medic/skills/alert-triage/`).

## 6. MCP servers

Neutral TOML catalog → provider-native projection
(`internal/materialize/mcp*.go`).

- **Authoring**: one TOML file per server in an `mcp/` directory.
  Filename = identity (`<name>.toml`, lowercase `[a-z0-9-]`, `name`
  field must match). `command`/`args`/`env` (stdio) XOR
  `url`/`headers` (http) — mutually exclusive, cross-field rules
  enforced. Relative commands resolve against the file's directory, so
  a pack can ship its own server binary.
- **Templates**: `<name>.template.toml` runs through Go text/template
  (`missingkey=error`) with `{{.CityRoot}}`, `{{.AgentName}}`,
  `{{.RigName}}`, `{{.RigRoot}}`, `{{.WorkDir}}`, `{{.IssuePrefix}}`,
  `{{.Branch}}`, plus every key of the agent's `env` map — the
  sanctioned way to inject per-agent tokens into shared definitions.
- **Precedence** (later wins; collisions logged as shadows, override is
  intentional): bootstrap → implicit imports → explicit imports → city
  pack `mcp/` → rig imports → agent-local `agents/<name>/mcp/`. Unlike
  skills, names are NOT namespaced across packs.
- **Projection**: written (not symlinked) to the provider-native file
  in the agent workdir — Claude: `.mcp.json`. The file is
  gascity-managed: pre-existing content is snapshotted to
  `.gc/mcp-adopted/` then owned; drift is reconciled away. **Never
  hand-edit `.mcp.json` in an agent workdir** — author in `mcp/` dirs.
- The `mcp = [...]` field in `agent.toml` is a tombstone.

## 7. MCP security posture: fail-closed

Problem: Claude Code authenticated via the claude.ai subscription
auto-imports the account's connectors as MCP servers — every agent
would get every connector tool, and the set drifts as connectors are
added.

**Decision: block all connectors everywhere; grant MCP per-agent via
Gas City `mcp/` definitions only.** A new connector on claude.ai then
changes nothing; every capability grant is a reviewable TOML file in
version control.

- **Baseline (both cities)**:

  ```toml
  # city.toml
  [workspace.env]
  ENABLE_CLAUDEAI_MCP_SERVERS = "false"
  ```

  `[workspace] env` applies to every managed session (lowest
  precedence, per-agent overridable — docs/reference/config.md).
- **Default agents**: no `mcp/` definitions → empty catalog → no
  `.mcp.json` → genuinely zero MCP.
- **Granted agents**: server definition in `agents/<name>/mcp/` +
  settings overlay with `"allow": ["mcp__<server>"]` (or per-tool) and
  `"enableAllProjectMcpServers": true` (avoids headless trust prompts).
- **Critical pitfall**: deny beats allow in Claude Code permissions. A
  blanket `"deny": ["mcp__*"]` cannot be punched through by allow
  rules — do NOT put it in a universal baseline; reserve it for agents
  that must never have MCP (hard local guarantee).
- **Host hygiene**: user-scope servers in `~/.claude.json` leak into
  every agent. Keep orchestrator hosts' user scope empty; add a pack
  doctor check (fail if `claude mcp list` at user scope is non-empty).
- **Auth tradeoff accepted**: no claude.ai-managed OAuth. API-key /
  static-token services wire cleanly (token in agent `env` +
  `.template.toml`). Prefer CLIs where they exist (`gh`, mail/calendar
  CLIs). Interactive-OAuth-only services need a one-time token dance —
  defer until actually needed.
- **Verify on deploy**: default agent `/mcp` shows empty; granted agent
  shows exactly its wired servers. Also verify
  `ENABLE_CLAUDEAI_MCP_SERVERS=false` behaves on the installed Claude
  Code version (docs-confirmed at design time; it is load-bearing).
- Optional outer fence (machine-wide, not per-agent): managed
  `managed-mcp.json` suppresses connectors by default and supports
  `allowedMcpServers`/`deniedMcpServers`.

## 8. Human-in-the-loop

Available mechanisms:

1. **Bead dependencies** — blocked beads are excluded from `bd ready`
   and never dispatched. The core primitive.
2. **Formula `needs` DAGs** — approval steps between investigate and
   apply steps.
3. **Mail with humans as participants** — agents mail you; read via
   `gc mail inbox`, dashboard, or a messaging bridge.
4. **`cascade-nudge-on-blocker-close`** (maintenance pack) — event
   order on `bead.closed` nudging dependents' assignees. Approval IS
   the trigger; no polling.
5. **Gates** (`bd gate check`, 30s sweep) — timer and GitHub-status
   gates work; **bead-type gates are currently disabled upstream**
   (beads v1.0.2 hard-fails them — see gate-sweep.sh comments). Use
   plain dependencies for human approval.

**Propose / approve / apply pattern** (read-only proposer → human →
write-capable applier):

```toml
[[steps]]
id = "propose"            # read-only pool; writes EXACT proposed action to the bead
[[steps]]
id = "approve"            # routed to no pool; human closes it (gc bd close <id>)
needs = ["propose"]
[[steps]]
id = "apply"              # write-capable pool; dispatched on approval close
needs = ["approve"]
```

Proposer's last act: mail the human with the proposal and the literal
close command. Enforcement boundaries, precisely:

- **Capability boundary (hard)**: proposer has no write tools or
  credentials; applier can't be reached pre-approval because dispatch
  never sees an un-ready bead.
- **Workflow boundary (softer)**: any agent with `bd` access could
  close the approval bead; prompts prevent it, not sandboxing. Defense
  in depth: applier prompt re-verifies the approve step's close
  metadata (actor) before acting. Strongest variant for tier-(c)
  actions: approval also injects a short-lived write credential, making
  the workflow boundary a capability boundary.

## 9. External messaging (extmsg)

`internal/extmsg/` is a complete fabric — **with zero concrete service
adapters shipped**. Slack/Discord/etc. appear only as example provider
strings; the bridge is ours to write.

- Provider-neutral `ConversationRef` (provider, account_id,
  conversation_id); persistent session↔conversation **bindings**;
  per-conversation **transcripts** with ack tracking; all bead-backed.
  Events: `extmsg.inbound/outbound/bound/unbound`.
- **Adapter registry** is in-memory per controller lifetime, keyed
  (provider, account_id). Out-of-process adapters register via
  `POST /v0/extmsg/adapters` with a callback URL; gascity forwards
  outbound publishes there. **Registrations don't survive controller
  restarts — adapters must re-register on reconnect.** Run the bridge
  under systemd with retry.
- Inbound: bridge POSTs `/v0/extmsg/inbound/{provider}/...` →
  transcript + `extmsg.inbound` event (order trigger) + delivery to the
  bound session.
- **Slack bridge** (~150–300 lines, any language): register on
  start/reconnect; publish callback → `chat.postMessage`; Slack Socket
  Mode (no public ingress) → inbound POST. One bridge serves alerts,
  approval requests, job-application drafts.
- **Don't use extmsg for one-way pings** — an agent can `curl` a Slack
  incoming-webhook from its prompt. The fabric earns its keep for
  two-way stateful conversations (bindings, transcripts,
  approval-by-reply: inbound → event order → close the approval bead).

## 10. Known gaps and risks

| Gap | Mitigation |
|---|---|
| No budget/spend enforcement (`internal/pricing/` is advisory) | Cooldowns, conservative cron, `one_shot` lifecycle, `max_session_age`; review usage early |
| No built-in pollers for external systems | `condition`-trigger orders wrapping API checks; webhooks via extmsg |
| Beads = task store, not knowledge base | Markdown-in-git rig for knowledge; beads track work about it |
| All example prompts are coding-flavored | Author every non-coding prompt from scratch (prompts ARE the roles) |
| Upstream velocity (~3,550 commits / 3.5 weeks observed) | Pin a version; upgrade deliberately, not by tracking main |
| Bead-type gates disabled upstream | Plain dependencies for approvals |
| OAuth-interactive MCP servers fail headless | Static tokens / headersHelper / CLIs; defer OAuth-only services |
| Adapter registrations ephemeral | Bridge re-registers on reconnect, systemd-managed |
| Controller host is itself homelab infra | Exclude from auto-remediation; tier-(c) only |

## 11. Build order

1. **Personal city skeleton**: city.toml with `[workspace.env]`
   connector block; one rig (`knowledge/`); one pack with one
   cron-triggered one-shot research agent. Proves order → formula →
   bead → ephemeral agent end to end. Lowest stakes (no external
   writes).
2. Add `productivity/` and `jobsearch/` rigs + packs; first
   `condition`-trigger order; mail notifications via webhook-curl.
3. **Engineering city**: gastown pack stock; coding repos as rigs;
   homelab rig + SRE pack with medic (polling alert order) and steward
   (cron updates); tiered autonomy in prompts.
4. Propose/approve/apply formula for tier-(c) actions; CLI/dashboard
   approval first.
5. Slack bridge; approval-by-reply; bind medic to the alerts channel.
6. Doctor checks: empty user-scope MCP; bridge liveness.

## 12. Key references

- Agent config fields: `docs/reference/config.md`,
  `docs/specs/pack-spec.md`
- Orders/triggers: `docs/tutorials/07-orders.md`,
  `internal/orders/triggers.go`
- Skills materialization: `internal/materialize/skills.go`
- MCP catalog/projection: `internal/materialize/mcp.go`,
  `mcp_resolve.go`, `mcp_project.go`, `mcp_runtime.go`
- Extmsg fabric: `internal/extmsg/`,
  `internal/api/huma_handlers_extmsg.go`
- HITL machinery: `examples/gastown/packs/maintenance/orders/`
  (`cascade-nudge-on-blocker-close`, `gate-sweep`),
  `docs/tutorials/04-communication.md`, `05-formulas.md`, `06-beads.md`
- Gastown pack wiring (per-rig imports example):
  `examples/gastown/city.toml`, `examples/gastown/pack.toml`
- Claude Code MCP/permissions (external):
  code.claude.com/docs — mcp, permissions, managed-mcp, settings,
  cli-reference, headless
