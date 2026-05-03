# Bragi Agent Control Plane

A local GitHub-connected control plane for managing multiple Bragi projects with AI manager/executor loops.

## Managed Projects

- **Router** — API routing and message relay
- **Portal** — Cloud control plane and orchestration
- **MSDK (mSDK)** — Native BLE transport plugin
- **AppKit** — Native library, audio-app runtime, Bragi API

## Phase: Discovery

We are in the initial discovery phase on branch `agent/bootstrap-discovery`. Before any code is written, we are mapping the existing ecosystem: vault structure, message passing, project docs, CI workflows, agent conventions, and more.

See [`docs/discovery-plan.md`](docs/discovery-plan.md) for the full exploration roadmap.

## Discovery Artifacts

Discovery findings will be documented in `docs/discovery/`:
- `vault-topology.md` — Knowledge vault structure
- `message-protocol.md` — Cross-repo message passing
- `memory-vault.md` — Memory vault structure
- `project-docs-catalog.md` — Per-project documentation inventory
- `collaboration-flows.md` — Orchestration and handover patterns
- `github-workflows.md` — CI/CD pipeline topology
- `claude-workflows.md` — Claude Code configuration per project
- `watcher-agents.md` — Autonomous agent roles and knowledge bases
- `gsd-conventions.md` — GSD planning conventions
- `access-points.md` — External service endpoints
- `risks.md` — Risk register
- `unknowns.md` — Open questions

## Key Ecosystem Paths

| Resource | Path |
|----------|------|
| Knowledge Vault | `~/Desktop/Antigravity/bragi-vault/` |
| Memory Vault | `~/Desktop/Antigravity/Obsidian_Vault/` |
| Messages | `~/Desktop/Antigravity/bragi-vault/messages/` |
| Cross-repo Decisions | `~/Desktop/Antigravity/bragi-vault/decisions/` |
| Platform Architecture | `~/Desktop/Antigravity/docs/` |
| Confluence | [Bragi AI 2](https://bragi.atlassian.net/wiki/spaces/BRAGIAI2) |
