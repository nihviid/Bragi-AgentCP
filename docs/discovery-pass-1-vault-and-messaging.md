# Discovery Pass 1 — Vault and Messaging System Structure

**Pass:** 1 of 10
**Date:** 2026-05-03
**Inspector:** BACP Discovery Agent
**Status:** Complete
**Bridge Status Report:** `20260503T120719Z~bacp~discovery~p00432~f1a7.status.json`

---

## 1. Vault Location and Top-Level Structure

### Location
`/Users/nikolajhviid/Desktop/Antigravity/bragi-vault/`

### Type
Obsidian vault serving as the cross-project knowledge graph for the Bragi AI platform. Contains both vault-original content and symlinked `docs/` directories from three repos.

### Top-Level Directory Breakdown

| Directory | Source | Contents | File Count (approx.) |
|-----------|--------|----------|---------------------|
| `appkit-docs/` | Symlink → `Bragi-AppKit/docs/` | Architecture, ADRs, specs, plans, integration, communication | 100+ |
| `portal-docs/` | Symlink → `Bragi-portal/docs/` | Portal architecture, integration, runbooks, specs, superpowers | 100+ |
| `msdk-docs/` | Symlink → `msdk-ios/Documentation/` | Protocol docs, vendor references | 50+ |
| `appkit-gsd/` | Symlink → `Bragi-AppKit/.planning/` | GSD codebase analysis | 50+ |
| `portal-gsd/` | Symlink → `Bragi-portal/.planning/` | GSD project state + codebase analysis | 50+ |
| `msdk-gsd/` | Symlink → `msdk-ios/.planning/` | GSD project state + codebase + phases | 50+ |
| `notes/` | Vault-original | Atomic permanent notes (Zettelkasten) | 8 |
| `cross-cutting/` | Vault-original | Cross-repo integration notes | 11 |
| `journal/` | Vault-original | Daily session logs | 8 |
| `decisions/` | Vault-original | Cross-project architectural decisions | 46 |
| `messages/` | Vault-original | Cross-repo async message passing | 100+ |
| `messages/pending/` | Vault-original | Unprocessed messages | 4 |
| `messages/completed/` | Vault-original | Processed messages | 100+ |
| `watchers/` | Vault-original | 8 autonomous Claude Code agent configs | 24+ |
| `coordination/` | Vault-original | Orchestration loop, milestone status | 3 |
| `business/` | Vault-original | Whitepapers, GTM, roadmaps | Unknown |
| `inbox/` | Vault-original | Quick capture, triage later | Unknown |
| `templates/` | Vault-original | Templates for notes, decisions, messages | 6 |
| `tooling/` | Vault-original | Confluence sync scripts, plans | 5+ |
| `assets/` | Vault-original | Icons, device assets, tokens | Unknown |

### Symlink Map

```
bragi-vault/
├── appkit-docs/  →  ../Bragi-AppKit/docs/        (read-only from vault)
├── portal-docs/  →  ../Bragi-portal/docs/         (read-only from vault)
├── msdk-docs/    →  ../msdk-ios/Documentation/     (read-only from vault)
├── appkit-gsd/   →  ../Bragi-AppKit/.planning/     (read-only from vault)
├── portal-gsd/   →  ../Bragi-portal/.planning/     (read-only from vault)
├── msdk-gsd/     →  ../msdk-ios/.planning/         (read-only from vault)
```

**Key rule:** Symlinked dirs are read-only from the vault's perspective. Edits happen in the originating repo. Changes appear instantly via the symlink.

### Vault Git
The vault is a git repository (at `bragi-vault/.git/`). It tracks all vault-original content and git submodules or standalone tracking for symlinked content would need investigation.

---

## 2. Obsidian Graph Conventions

### Obsidian Version
Community plugins enabled: **Dataview**, **Templater**, **Obsidian Git**

### Core Plugins Active (30 total, 22 active)
Key active: graph view, backlinks, outgoing links, tag pane, canvas, daily notes, templates, properties, bookmarks, page preview, outline, file recovery, sync

### Graph Configuration
- Collapse filter: on
- Show orphans: yes (unlinked notes visible)
- Hide unresolved: no
- Node size: 1
- Line size: 1
- Link distance: 250
- Graph is closed by default (manual open)

### Frontmatter Convention

Every vault-original note uses YAML frontmatter with:

| Field | Purpose | Example |
|-------|---------|---------|
| `type` | Note classification | `note`, `daily`, `decision`, `integration`, `moc` |
| `date` | Creation date | `2026-04-14` |
| `tags` | Categorization | `[market, strategy, platform-economics]` |
| `repos` | Related repos | `[portal]`, `[appkit, portal]` |
| `source` | Where the idea came from | `Bragi Strategy Whitepaper (March 2026)` |

### Note Types

| Type | Purpose | Template |
|------|---------|----------|
| `note` | Atomic permanent note, one idea | `templates/note.md` |
| `daily` | Daily capture in `journal/` | `templates/daily.md` |
| `decision` | Cross-project architectural decision | `templates/decision.md` |
| `integration` | Cross-cutting integration point | `templates/cross-cutting.md` |
| `moc` | Map of Content — landing page per project | `AppKit.md`, `Portal.md`, `mSDK.md` |

### Atomic Note Convention (Zettelkasten)

From `README.md`: "One idea per note in `notes/`, titled as a claim or statement, not a topic."

Structure:
```markdown
# Bilateral integration is structurally impossible at scale

## The idea
{one clear claim}

## Evidence
{data, sources}

## Connected to
- [[wiki-link]]
- [[wiki-link]]

## So what?
{implications for action}
```

Only 8 atomic notes exist in `notes/`. This is a small but high-quality corpus.

### MOC Pages
- `AppKit.md` — AppKit docs, key areas, connections
- `Portal.md` — Portal docs, key areas, connections
- `mSDK.md` — mSDK docs, protocols, references
- `2026-04-15.md` — Cross-project dashboard

### Wiki-Link Convention
Symlinked docs are referenced as `[[appkit-docs/path/to/file|Display Name]]`. Vault-original content with `[[Note Title]]`. These links drive the Obsidian graph.

### Journal Cadence
8 journal entries spanning 2026-04-14 to 2026-04-23. Capture cadence: roughly daily during active development periods.

---

## 3. Memory Vault Conventions

### Location
`/Users/nikolajhviid/Desktop/Antigravity/Obsidian_Vault/`

### Current State
- Contains a single folder: `Bragi AI/`
- Has a `CLAUDE.md` (attempted read returned empty — file may be empty or have minimal content)
- Purpose: Ambient AI memory system. Records audio, transcribes, classifies, extracts entities, builds knowledge graph, enables natural language search.
- Stack: React 19 + Tauri 2 frontend, Python FastAPI backend, SQLite persistence.

### Key Integration Points
- SSE event bus (`GET /stream`) for real-time updates
- Blackbox recorder pipeline: VAD → turn detection → diarization → STT → entity extraction → graph storage
- Backend at port 8009

### Relationship to Knowledge Vault
The memory vault and knowledge vault are separate:
- **Memory Vault** (Obsidian_Vault): Operational memory — stores recordings, transcripts, entities, embeddings. The "what happened" layer.
- **Knowledge Vault** (bragi-vault): Permanent knowledge — architecture, decisions, integration notes, atomic ideas. The "what we know" layer.

### Current Status
The memory vault has a full CLAUDE.md (visible in the project-level CLAUDE.md defined at `~/Desktop/CLAUDE.md`). Key points: 1,931-line monolith backend, 19 feature modules, 81 direct `fetch()` calls in frontend, zero abstract interfaces, orphaned components.

---

## 4. Message Folder Structure

### Location
`bragi-vault/messages/`

### Directory Structure
```
messages/
├── .notify-appkit          # Flag file (empty, zero content)
├── .notify-portal          # Flag file
├── .notify-msdk            # Flag file
├── .notify-ceo             # Flag file
├── .notify-cto             # Flag file
├── .notify-cpo             # Flag file
├── .notify-cro             # Flag file
├── .notify-consultant      # Flag file
├── .notify-all             # Flag file
├── .notify-orchestrator    # Flag file
├── .notify-memory-vault    # Flag file
├── .notify-audioapp        # Flag file
├── .notify-bragi-audioapp  # Flag file
├── .notify-bragi-team      # Flag file
├── pending/                # Unprocessed messages
│   ├── PORTAL_TO_APPKIT_2026-04-29_*.md
│   ├── PORTAL_TO_APPKIT_2026-04-29_*.md
│   └── PORTAL_TO_APPKIT_2026-04-30_*.md
└── completed/              # Processed messages (300 entries)
```

### Flag Files
14 `.notify-*` files. Each is a zero-byte marker file. **Presence** means "there is a message for you." **Absence** means "idle — no new messages."

| Flag | Target Agent | Currently Present? |
|------|-------------|-------------------|
| `.notify-all` | All agents | Yes |
| `.notify-appkit` | AppKit watcher | Yes |
| `.notify-portal` | Portal watcher | No |
| `.notify-msdk` | mSDK watcher | No |
| `.notify-ceo` | CEO watcher | No |
| `.notify-cto` | CTO watcher | No |
| `.notify-cpo` | CPO watcher | No |
| `.notify-cro` | CRO watcher | No |
| `.notify-consultant` | Consultant watcher | No |
| `.notify-orchestrator` | Orchestrator | No |
| `.notify-memory-vault` | Memory vault system | No |
| `.notify-audioapp` | AudioApp watcher | No |
| `.notify-bragi-audioapp` | Bragi AudioApp watcher | No |
| `.notify-bragi-team` | All Bragi team agents | No |

---

## 5. Pending/Completed Message Flow

### State Machine

```
CREATED (sender writes to pending/)
   → PENDING (visible in pending/, flag file exists)
   → RESPONDED (receiver writes ## Response, changes frontmatter status)
   → COMPLETED (file moved to completed/)
```

### Lifecycle

1. **Sender** writes message to `messages/pending/` with frontmatter `status: pending`
2. **Sender** creates the corresponding `.notify-<target>` flag file
3. **Receiver** (polling in a `/loop` session) sees flag file on next tick
4. **Receiver** deletes flag file
5. **Receiver** scans `pending/` for matching messages
6. **Receiver** reads message, processes request
7. **Receiver** writes response under `## Response` section in the same file
8. **Receiver** changes frontmatter `status: pending` → `status: responded`
9. **Receiver** moves file from `pending/` → `completed/`
10. **Sender** checks `completed/` on next session for their messages with `status: responded`

### Current State
- **4 pending messages** — all from Portal to AppKit, all from 2026-04-29/30
- **300+ completed messages** — spanning AppKit→All, AppKit→AudioApp, Portal→AppKit, MSDK→Portal, etc.

### Message File Naming Convention
```
<FROM>_TO_<TO>_YYYY-MM-DD_<slug>.md
```

Examples:
```
APPKIT_TO_ALL_2026-04-23_phase-23-mcp-server-complete.md
PORTAL_TO_APPKIT_2026-04-30_phase41-device-descriptor-confirmed.md
APPKIT_TO_AUDIOAPP_2026-04-21_phase-19-asset-bundle-ask.md
```

---

## 6. Flag Conventions

### How Flags Work

A `.notify-<role>` file is an empty marker file. Its mere existence signals to the target agent that there are pending messages. Its absence means "nothing to do."

### Agent Loop Pattern (from watcher CLAUDE.md files)

Every watcher agent runs via `/loop 60s`. On each iteration:

1. Check for flag file at `~/Desktop/Antigravity/bragi-vault/messages/.notify-<role>`
2. If flag does NOT exist: report "No flag — idle", yield
3. If flag EXISTS:
   a. Delete the flag file
   b. Scan `pending/` for matching messages
   c. Process each: read → respond → change status → move to `completed/`
4. Yield

### Flag Scope

- `.notify-appkit` → AppKit-specific messages
- `.notify-portal` → Portal-specific messages
- `.notify-msdk` → mSDK-specific messages
- `.notify-all` → Messages that target all agents
- `.notify-ceo`, `.notify-cto`, etc. → Advisory agent messages
- `.notify-orchestrator` → Orchestrator loop messages
- `.notify-memory-vault` → Memory vault system messages

### Current State
Only `.notify-all` and `.notify-appkit` flags are currently present. All other flags are absent, meaning no watcher agents have pending work.

---

## 7. Cross-Project Collaboration Messages

### Traffic Analysis

**300+ completed messages** observed. Dominant sender/receiver pairs:

| Sender → Receiver | Volume (estimated) | Purpose |
|-------------------|-------------------|---------|
| APPKIT → ALL | Highest | Milestone completion announcements, phase updates |
| PORTAL → APPKIT | High | Task assignments, contract updates, realignment requests |
| APPKIT → AUDIOAPP | Low | Phase-specific ask (e.g., asset bundles) |
| MSDK → PORTAL | Medium | Transport-level updates |

### Message Types Found

| Type | Purpose | Examples |
|------|---------|----------|
| `announcement` | Milestone/phase completion | APPKIT_TO_ALL phase complete messages |
| `status-update` | In-flight status with cross-repo impact | Portal→AppKit e2e suite updates |
| `request` | Ask another project for work | Portal→AppKit task assignments |
| `blocked` | Stuck, needs input | (observed frontmatter option) |
| `decision` | Cross-repo decision | (observed in answered decisions) |

### Current Pending Messages
All 4 pending messages are Portal→AppKit, suggesting a backlog of Portal instructions waiting for AppKit watcher processing.

---

## 8. Examples of Cross-Project Collaboration

### Example 1: Portal asks AppKit to realign contracts (PENDING)

**File:** `messages/pending/PORTAL_TO_APPKIT_2026-04-29_e2e-driver-registry-contract-realignment.md`

Portal sent a detailed technical message to AppKit explaining that Portal's e2e test suite was realigned to match the AF-09 v0.2 contract after a Portal PR changed the driver registry public-key endpoint. The message:
- References specific PR commits by hash
- Identifies 5 assertions that can never pass against the new route
- Explains the contract change (SPKI → PEM, `ok` envelope removed)
- Provides technical context AppKit's DriverVerifier needs
- References dependency relationships between Portal and mSDK messages

### Example 2: AppKit announces to all projects (COMPLETED)

**File:** `messages/completed/APPKIT_TO_ALL_2026-04-23_phase-23-mcp-server-complete.md`

AppKit announced Phase 23 (MCP Server & Developer Access) to all projects. The message:
- Lists 6 new MCP tools with descriptions
- Documents CLI commands (`bragi-devkit`)
- Reports test coverage (7203 backend, 1102 frontend, zero regressions)
- Provides endpoint URLs and auth details

### Example 3: Portal confirms decisions to AppKit (COMPLETED)

**File:** `messages/completed/PORTAL_TO_APPKIT_2026-04-29_af09-followups-merged.md`

Portal responded to AppKit's questions with detailed technical answers, including a 9-category field ownership table showing which fields Portal owns vs. AppKit can report.

### Messaging Conventions Observed

- Frontmatter includes `milestone`, `relates-to`, and `in_reply_to` fields for threading
- Messages reference external systems: GitHub PR URLs, commit hashes, test counts
- Responses are written under `## Response` in the original file
- Completed messages remain in `completed/` permanently (git-tracked history)

---

## 9. Files and Folders That Must Never Be Modified Automatically

### Hard Rules

| Path | Reason | Modifiable by BACP? |
|------|--------|---------------------|
| `bragi-vault/appkit-docs/` | Symlink to AppKit repo | **Never** — read-only from vault |
| `bragi-vault/portal-docs/` | Symlink to Portal repo | **Never** — read-only from vault |
| `bragi-vault/msdk-docs/` | Symlink to mSDK repo | **Never** — read-only from vault |
| `bragi-vault/appkit-gsd/` | Symlink to AppKit .planning | **Never** — read-only from vault |
| `bragi-vault/portal-gsd/` | Symlink to Portal .planning | **Never** — read-only from vault |
| `bragi-vault/msdk-gsd/` | Symlink to mSDK .planning | **Never** — read-only from vault |
| `bragi-vault/messages/pending/` | Has active watcher agents | **Never** — BACP messages use own bridge |
| `bragi-vault/messages/.notify-*` | Watcher agent control signals | **Never** — BACP uses own flag system |
| `Bragi-AppKit/` | Managed project | **Never** during discovery |
| `Bragi-portal/` | Managed project | **Never** during discovery |
| `msdk-ios/` | Managed project | **Never** during discovery |
| `~/.bragi/agent-control-plane/STOP` | Kill switch — human only | **Never** — only human creates/deletes |

### BACP Bridge Boundary

The BACP bridge at `~/.bragi/agent-control-plane/` is the ONLY write target for BACP agents. No BACP agent writes to `bragi-vault/messages/` directories — those are for the existing watcher agent system.

### Read Access

BACP discovery agents may read any file in any project or vault. This is explicit read-only permission for discovery purposes. After discovery ends, read access follows the same "never modify" rules.

---

## 10. How This Should Connect to BACP Later

### Recommended Integration Points

| Vault/Messaging Feature | BACP Connection | Phase |
|------------------------|----------------|-------|
| **Message format conventions** | BACP bridge extends the naming pattern (`<FROM>_TO_<TO>_<DATE>_<SLUG>.md`) with JSON schema and filename uniqueness | Phase 1 (done) |
| **Flag polling pattern** | BACP agents use the same flag-based poll pattern as watcher agents, but with their own `.bridge/` directory | Phase 1 (done) |
| **Watcher agent identities** | BACP discovers and registers all 8 watcher agents as known entities. Future BACP manager could coordinate watchers | Phase 9 |
| **Orchestration loop** | BACP manager reads `bragi-vault/coordination/ORCHESTRATION.md` to understand the existing orchestration protocol before proposing changes | Pass 5 |
| **Decision records** | BACP reads `bragi-vault/decisions/` to understand past cross-project decisions before making new ones | Pass 5 |
| **Symlinked docs** | BACP respects symlink boundaries. It reads through symlinks but never writes through them | Phase 1 (ongoing) |
| **Journal entries** | BACP could write journal summaries to `bragi-vault/journal/` — but only if explicitly requested. Not automatic. | Future |
| **Cross-cutting notes** | BACP findings that span multiple projects could eventually become `cross-cutting/` notes. But not during discovery — read-only. | Future (post-discovery) |
| **GitHub PR linking** | BACP bridge already carries `git_status` and `files_created/modified` — these are Phase 1 fields ready for GitHub integration | Phase 1 (done) |

### What BACP Brings That Doesn't Exist

| Capability | Gap Today | BACP Solution |
|-----------|-----------|---------------|
| Manager↔CLI communication | Manual copy/paste | Phase 1 structured bridge |
| Cross-agent coordination | Portal as orchestrator (human-driven) | ChatGPT manager + BACP bridge |
| Automated message routing | Flag-based (agent pulls) | BACP bridge with project-aware routing |
| Decision tracking | `bragi-vault/decisions/` is permanent | BACP bridge `decisions/pending/answered/` with lifecycle |
| Kill switch | None | BACP `STOP` file |
| Audit trail | None for agent ops | BACP `logs/bridge.log` |

### Integration Principle

**BACP augments, does not replace.** The existing vault, message, and flag systems continue operating as they do today. BACP adds a new communication layer for manager↔CLI orchestration. The two systems coexist:
- **bragi-vault/messages/** — cross-repo developer knowledge (permanent, git-tracked)
- **~/.bragi/agent-control-plane/** — operational bridge messages (ephemeral, 90-day archive)

Cross-over: when a BACP bridge exchange produces a durable insight (decision, integration note), a summary should be written to `bragi-vault/` — but only with explicit human approval. Not automatic.

---

## Discovery Notes

- Total vault files examined: ~40
- Total messages examined: 5 (1 pending, 4 completed)
- Watcher agent configs examined: 3 (appkit, portal, cto)
- Obsidian plugin list confirmed: 3 community plugins (Dataview, Templater, Obsidian Git)
- No unintended modifications were made
- All exploration was read-only

## Files Examined

- `bragi-vault/CLAUDE.md`
- `bragi-vault/README.md`
- `bragi-vault/.obsidian/core-plugins.json`
- `bragi-vault/.obsidian/community-plugins.json`
- `bragi-vault/.obsidian/graph.json`
- `bragi-vault/.obsidian/app.json`
- `bragi-vault/.obsidian/appearance.json`
- `bragi-vault/templates/message.md`
- `bragi-vault/templates/note.md`
- `bragi-vault/templates/daily.md`
- `bragi-vault/templates/decision.md`
- `bragi-vault/templates/cross-cutting.md`
- `bragi-vault/notes/Bilateral integration is structurally impossible at scale.md`
- `bragi-vault/Portal.md`
- `bragi-vault/coordination/ORCHESTRATION.md`
- `bragi-vault/messages/pending/PORTAL_TO_APPKIT_2026-04-29_e2e-driver-registry-contract-realignment.md`
- `bragi-vault/messages/pending/PORTAL_TO_APPKIT_2026-04-30_phase41-device-descriptor-confirmed.md`
- `bragi-vault/messages/completed/APPKIT_TO_ALL_2026-04-23_phase-23-mcp-server-complete.md`
- `bragi-vault/watchers/appkit/CLAUDE.md`
- `bragi-vault/watchers/portal/CLAUDE.md`
- `bragi-vault/watchers/cto/CLAUDE.md`
