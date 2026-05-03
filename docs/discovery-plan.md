# Bragi Agent Control Plane — Discovery Plan

**Date:** 2026-05-03
**Branch:** `agent/bootstrap-discovery`
**Author:** Control Plane Bootstrap Agent

---

## Purpose

This document defines the initial discovery phase for the Bragi Agent Control Plane (ACP). The ACP's goal is to be a local, GitHub-connected orchestration layer that manages multiple Bragi projects with AI manager/executor loops.

Before any code is written, the ACP must understand the existing ecosystem. This plan defines exactly what to inspect, what to leave alone, and what to report.

### Bridge Is Operational Before Discovery

The Phase 1 ChatGPT-to-CLI bridge was set up before this discovery plan executes. This is intentional.

**Every discovery pass MUST report through the Phase 1 manual structured bridge.** No discovery task may silently inspect or modify systems without producing a structured status report.

**Requirements:**
- Every discovery pass must end with a BACP STATUS REPORT written to `outbox/cli-to-manager/`.
- Every BACP STATUS REPORT must follow the schema defined in `docs/chatgpt-cli-bridge.md`.
- Every status file must follow the filename convention: `TIMESTAMP~PROJECT~ROLE~PID~NONCE.TYPE.json`.
- The bridge log at `~/.bragi/agent-control-plane/logs/bridge.log` must be updated after every bridge operation.
- Discovery remains read-only for Router, Portal, MSDK, AppKit, and the vault. Bridge reporting is the only write activity.

---

## Scope of Discovery

### What WE WILL inspect
- All discovery activity reports through the Phase 1 bridge at `~/.bragi/agent-control-plane/`
- Directory structures, file layouts, and configuration files in each project
- All CLAUDE.md and project instructions files
- GitHub Actions workflows and CI configuration
- Message/flag passing conventions and examples
- Obsidian vault structure and knowledge graph topology
- Cross-repo collaboration patterns (handover packages, orchestration loops)
- Claude Code CLI configurations, skills, and scheduled tasks
- Watcher agent designs and knowledge bases
- bragi-vault symlink topology
- Platform architecture documents at the `Antigravity/` level

### What WE WILL NOT touch
- No source code files (`.py`, `.ts`, `.tsx`, `.swift`, `.java`, etc.)
- No running services, databases, or environment variables
- No production or staging infrastructure
- No PRs, issues, or branches on existing repos
- No Confluence page modifications
- No changes to any Router, Portal, MSDK, or AppKit repository

### What we will report
- Complete ecosystem topology map
- Integration surface catalog (what connects to what, how)
- Gap analysis: what the ACP needs vs. what exists
- Risk register for each integration point
- Discovery artifacts in `docs/discovery/` subdirectory

---

## Discovery Methodology

Each discovery pass is read-only. The ACP agent will:

1. **List** — enumerate the structure of each target area
2. **Read** — inspect key files (head 100 lines for unfamiliar files)
3. **Map** — document relationships between targets
4. **Report** — record findings in structured discovery artifacts

Discovery executes in ordered passes. Each pass must complete before the next begins. Findings from each pass inform the next.

---

## Pass 1 — Vault Structure

### Target
`bragi-vault/` at `/Users/nikolajhviid/Desktop/Antigravity/bragi-vault/`

### Actions
- Enumerate all top-level directories (not hidden, not symlinked repos)
- Map all symlink targets: `appkit-docs/`, `portal-docs/`, `msdk-docs/`, `appkit-gsd/`, `portal-gsd/`, `msdk-gsd/`
- Read `CLAUDE.md` — understand vault purpose and rules
- Read `README.md` — understand knowledge method
- Count files in each vault-original folder (`cross-cutting/`, `journal/`, `decisions/`, `notes/`, `business/`, `inbox/`)
- Identify `.obsidian/` plugin list (if accessible)
- Read topic-map files: `AppKit.md`, `Portal.md`, `mSDK.md`, `2026-04-15.md`

### Artifacts
- `docs/discovery/vault-topology.md` — symlink map, folder purposes, file counts
- `docs/discovery/vault-topic-maps.md` — key MOC pages and their linked content

### Questions to answer
- What does the vault's Obsidian graph look like at the structural level?
- Which folders are vault-original vs. symlinked?
- How many atomic notes exist in `notes/`?
- What is the journal capture cadence?
- What decisions have been recorded in `decisions/`?

---

## Pass 2 — Message and Flag System

### Target
`bragi-vault/messages/` and flag files

### Actions
- Enumerate all `.notify-*` flag files in `messages/`
- Enumerate all pending messages
- Enumerate all completed messages (count and categorize by sender/receiver)
- Read 3 representative messages (one pending, one completed appkit→all, one portal→appkit)
- Read the message template from `templates/message.md`
- Read `ORCHESTRATION.md` in `coordination/`
- Read `MILESTONE-STATUS.md` in `coordination/`

### Artifacts
- `docs/discovery/message-protocol.md` — protocol rules, naming conventions, state machine
- `docs/discovery/message-traffic.md` — volume, sender/receiver pairs, topics

### Questions to answer
- What is the complete message state machine (status values, transitions)?
- What roles/agents exist in the flag system?
- What is the typical message velocity (messages per day)?
- Which sender/receiver pairs dominate traffic?
- Are there any stale or stuck messages?
- How does the orchestration loop consume messages?

---

## Pass 3 — Memory Vault

### Target
`Obsidian_Vault/` at `/Users/nikolajhviid/Desktop/Antigravity/Obsidian_Vault/`

### Actions
- Read `CLAUDE.md` — understand memory vault purpose and rules
- Enumerate top-level structure
- Identify integration points with the memory vault (SSE, APIs, agents)
- Map any agent-specific memory requirements for each project

### Artifacts
- `docs/discovery/memory-vault.md` — structure, purpose, access patterns

### Questions to answer
- How does the memory vault relate to the knowledge vault?
- What data does it store that the ACP needs to be aware of?
- Are there existing agent interfaces that the ACP should use or complement?
- What is the overlap with the knowledge vault?

---

## Pass 4 — Project Documentation

### Targets
- `Bragi-AppKit/docs/`
- `Bragi-portal/docs/`
- `msdk-ios/Documentation/` (via symlink)
- `docs/` at the `Antigravity/` level

### Actions
- Enumerate top-level doc directories for each project
- For each project:
  - Count spec documents, ADRs, integration docs, runbooks, architecture docs
  - Identify key decision records (especially cross-repo)
  - Map documentation conventions (templates, frontmatter, naming)
- Read platform architecture documents at the `Antigravity/docs/` level:
  - `BRAGI_PLATFORM_MASTER_ARCHITECTURE.md`
  - `BRAGI_VAULT_TECHNICAL_ARCHITECTURE.md`
  - `PORTAL_BACKEND_INTERFACE.md`
- Read project-level READMEs if present

### Artifacts
- `docs/discovery/project-docs-catalog.md` — per-project doc inventory with key documents

### Questions to answer
- What documentation standards exist across projects?
- Where do cross-repo contracts live (specs, ADRs, RFCs)?
- What architecture decisions have been formalized?
- Are there gap areas with insufficient documentation?
- What is the relationship between `Bragi-AppKit/docs/decisions/` and `bragi-vault/decisions/`?

---

## Pass 5 — Cross-Project Collaboration Flows

### Targets
- `bragi-vault/coordination/`
- `bragi-vault/cross-cutting/`
- `bragi-vault/decisions/`
- Cross-repo sections in each project's CLAUDE.md
- Platform orchestration loop documentation

### Actions
- Read `ORCHESTRATION.md` completely (already started)
- Read all files in `coordination/`
- Read cross-cutting integration notes
- Read key cross-project decision records
- Document the handover package protocol
- Document the bsolve decision-making process
- Identify all cross-repo interfaces and their ownership

### Artifacts
- `docs/discovery/collaboration-flows.md` — orchestration loop, handover protocol, decision resolution

### Questions to answer
- How does Portal orchestrate multi-repo work?
- What is the handover package lifecycle?
- How are cross-repo decisions resolved (bsolve vs. unilateral)?
- What is the PR collaboration workflow?
- How does the milestone completion process work?
- What happens when work is blocked mid-task?

---

## Pass 6 — GitHub Workflows

### Targets
- `Bragi-portal/.github/workflows/`
- `Bragi-AppKit/.github/workflows/`
- `msdk-ios/.github/workflows/`

### Actions
- Enumerate all workflow YAML files per project
- Read each workflow (head 100 lines minimum, full read for critical CI gate)
- Identify: trigger events, job structure, test runners, deploy targets, artifact publishing
- Map CI pipeline topology (what runs when, what gates what)
- Identify any cross-repo CI coordination (workflow dispatch, composite actions, etc.)

### Artifacts
- `docs/discovery/github-workflows.md` — CI pipeline map per project, cross-repo dependencies

### Questions to answer
- What test suites gate merging in each repo?
- What are the deploy pipelines per project?
- Is there any cross-repo CI coordination?
- What secrets/credentials are used by CI?
- Are there scheduled workflows (nightly tests, cron jobs)?
- What artifacts are published and to where?

---

## Pass 7 — Claude/CLI Workflows

### Targets
- `~/.claude/` — user-level Claude Code config
- `~/.claude/CLAUDE.md` — user-level instructions (already read)
- `.claude/settings.json` for each project
- `.claude/settings.local.json` for each project
- `.claude/commands/` for each project
- `.claude/skills/` for each project
- `.claude/MCP-SETUP.md` for applicable projects

### Actions
- Read user-level `CLAUDE.md` and identify cross-project rules
- Read user-level `.claude/settings.json` for theme, permissions, model settings
- For each project, read `.claude/settings.json` — project-level permissions and allowed commands
- For each project, read `.claude/settings.local.json` — local overrides
- List `.claude/commands/` per project — custom slash commands
- List `.claude/skills/` per project — custom skills
- Identify any `.claude/scheduled_tasks.lock` — scheduled recurring tasks
- Identify any `.claude/worktrees/` — git worktree usage patterns

### Artifacts
- `docs/discovery/claude-workflows.md` — Claude Code configuration per project, cross-project settings

### Questions to answer
- What custom commands exist per project?
- What permissions are granted per project?
- Are there scheduled Claude tasks running?
- What worktree patterns are used?
- What model settings are configured?
- Are there project-specific skills?

---

## Pass 8 — Project-Specific Agent/Automation Conventions

### Targets
- Watcher agents in `bragi-vault/watchers/`
- GSD planning directories (`bragi-vault/appkit-gsd/`, `portal-gsd/`, `msdk-gsd/`)
- Any `.planning/` directories in project repos
- Project AUDIT documents
- Any MCP configurations

### Actions
- Read each watcher's `CLAUDE.md` — 8 watchers (appkit, ceo, consultant, cpo, cro, cto, msdk, portal)
- Read each watcher's `rules/` directory
- Read each watcher's `knowledge/` directory (head 100 each)
- List contents of GSD planning directories
- Read project AUDIT files
- Read any MCP-SETUP.md files

### Artifacts
- `docs/discovery/watcher-agents.md` — each watcher's role, rules, knowledge base
- `docs/discovery/gsd-conventions.md` — GSD planning structure and conventions

### Questions to answer
- What are the 8 watcher agents, their roles, and their rules?
- How does the GSD planning system work?
- What automation conventions exist beyond watchers?
- Are there MCP servers configured?
- What audit processes exist?

---

## Pass 9 — Access Points

### Targets
- All integration surfaces discovered in Passes 1–8

### Actions
- Compile all external service access points:
  - GitHub API (PRs, issues, releases)
  - Confluence API (token at `~/.config/bragi-confluence-token`)
  - Railway deployment endpoints
  - Supabase project references
  - OpenAI API key usage
  - Any MCP endpoints
- Document which access points the ACP will need
- Document which credentials/tokens are needed

### Artifacts
- `docs/discovery/access-points.md` — all external services, URLs, and credential requirements

### Questions to answer
- What APIs does the ACP need to call?
- What credentials are required and where do they live?
- What access is read-only vs. requires write?
- Are there rate limits or access restrictions to consider?

---

## Pass 10 — Risks and Unknowns

### Actions
- For each discovery pass, identify:
  - Risks: things that could break, degrade, or conflict
  - Unknowns: things we don't yet know how to handle
  - Dependencies: things the ACP depends on that may change
- Compile risk register
- Compile unknowns log

### Artifacts
- `docs/discovery/risks.md` — risk register with mitigation strategies
- `docs/discovery/unknowns.md` — open questions for future investigation

### Risk categories to evaluate
- **Structural risk**: Will the ACP's architecture conflict with existing patterns?
- **Security risk**: What credentials and access tokens need protection?
- **Operational risk**: Will ACP agents interfere with watcher agents?
- **Dependency risk**: What happens when a project changes its conventions?
- **Scalability risk**: Can the message/flag system handle increased traffic?
- **Integration risk**: Which integrations are fragile or undocumented?

---

## Deliverable Format

Each discovery artifact will be a markdown file in `docs/discovery/` with:

```markdown
# Discovery: {Area}

**Pass:** {Pass number}
**Date:** {YYYY-MM-DD}
**Inspector:** ACP Discovery Agent

## Summary
{2-3 sentence overview}

## Findings
### {Finding 1}
{Details}

### {Finding 2}
{Details}

## Implications for ACP
{What this means for the control plane design}

## Files Examined
{List of files read during this pass}
```

A final synthesis document `docs/discovery-synthesis.md` will consolidate findings across all passes into a single ecosystem map with actionable recommendations for ACP design.

---

## Completion Criteria

Discovery is complete when:

1. All 10 passes have been executed
2. All discovery artifacts are written to `docs/discovery/`
3. `docs/discovery-synthesis.md` consolidates findings
4. All 10 questions per pass are answered (or marked as unknown)
5. Risk register is populated with mitigations (not just risks)
6. ACP design recommendations are actionable from the synthesis alone

## Next Phase

After discovery completes, the ACP will enter design phase on a new branch (`agent/design`). The design phase will use the discovery synthesis to architect:

- ACP agent architecture (manager agents, executor agents)
- Repository structure for ACP
- Integration patterns for each project
- Cross-project orchestration improvements
- Message bus design
- Watcher agent coordination
- GitHub workflow integration

No implementation work on those designs begins during this discovery phase.
