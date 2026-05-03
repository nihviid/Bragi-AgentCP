# Discovery Pass 2 — Project Documentation and Operating Models

**Pass:** 2 of 10
**Date:** 2026-05-03
**Inspector:** BACP Discovery Agent
**Status:** Complete

---

## 1. Repository Locations

| Project | Path | Remote | Stack | Exists? |
|---------|------|--------|-------|---------|
| **AppKit** | `~/Desktop/Antigravity/Bragi-AppKit/` | `git@github.com:nihviid/Bragi-AppKit.git` | Python FastAPI + React/Capacitor | Yes |
| **Portal** | `~/Desktop/Antigravity/Bragi-portal/` | `git@github.com:nihviid/bragi-portal.git` | Next.js 16 + Supabase, pnpm monorepo | Yes |
| **MSDK** | `~/Desktop/Antigravity/msdk-ios/` | `https://github.com/nihviid/msdk-ios.git` | Swift (SPM + CocoaPods) | Yes |
| **Router** | Not yet created | N/A | TBD | **No** — does not exist yet |
| **Bragi-Nerve** | `~/Desktop/Antigravity/Bragi-Nerve/` | Not checked | AI-mediated decision routing | Exists but is a separate project (spec approved, not yet built) |

### Repository Relationships

```
AppKit (native runtime)
  │
  ├── consumes from: Portal (config, drivers, manifests)
  ├── talks to: MSDK (via Capacitor plugin bridge)
  └── delivers: SDK, API runtime for host apps

Portal (cloud control plane)
  │
  ├── publishes: @bragi-ai/types (shared contracts)
  ├── delivers to: AppKit (resolved config, driver bundles, manifests)
  └── serves: Bragi Builder API at /api/v1/*

MSDK (BLE transport)
  │
  ├── consumed by: AppKit (26 Capacitor methods)
  ├── receives from: Portal (signed driver bundles)
  └── talks to: Headphone hardware (BLE/MFi/iAP2)

Router (planned)
  │
  └── future: API routing, message relay between projects
```

---

## 2. Documentation Structure

### AppKit Documentation

| Doc Section | Contents | Key Documents |
|-------------|----------|--------------|
| `docs/architecture/` | 15 architecture documents | Platform capabilities, 10 Kits, audio pipeline, HAL, SDUI, memory vault |
| `docs/decisions/` | 12+ ADRs (ADR-001 through ADR-012) | SQLite single-writer, response envelope, etc. |
| `docs/integration/` | 10+ integration documents | API reference, Portal contracts, OAuth, handshake |
| `docs/specs/` | Product specs, PRDs | Shortcut module, superpowers design specs |
| `docs/plans/` | Implementation plans | Build sequences, superpowers plans |
| `docs/audits/` | Quality audits, gap analyses | AI pipeline performance, architecture audits |
| `docs/onboarding/` | Developer setup | Developer quickstart, deployment runbook |
| `docs/mcp/` | MCP server documentation | MCP tool specs |
| `docs/communication/` | Archived cross-team communications | Dated subdirectories |
| `docs/business/` | GTM, brand guides | Engineering whitepaper |
| `docs/legacy/` | Archived phase documents | Reference only |
| `docs/releases/` | Release notes | Version history |
| `docs/superpowers/` | Superpowers skills & plans | Design specs, plans |

### Portal Documentation

| Doc Section | Contents | Key Documents |
|-------------|----------|--------------|
| `docs/architecture/` | 10+ docs | System design, deployment model, HAL concepts, tech stack |
| `docs/decisions/` | 11 ADRs (001-011) | Draft/live, readiness, BYOK, simulator kill, CSS |
| `docs/integration/` | 8+ docs | Activation flow, go-live checklist, integration overview |
| `docs/onboarding/` | 3 docs | Getting started, codebase tour, migration guide |
| `docs/specs/` | UX redesign spec | Canonical UX spec v3 |
| `docs/runbooks/` | Operational runbooks | Runbooks for common tasks |
| `docs/contracts/` | Cross-repo contracts | Shared interface definitions |
| `docs/guides/` | Developer guides | How-to guides |
| `docs/security/` | Security documentation | Security model |
| `docs/communication/` | Historical team discussions | Dated subdirectories |
| `docs/superpowers/` | Superpowers skills | Developer tooling skills |
| `docs/presentations/` | Deck exports | Presentation materials |

### MSDK Documentation

| Doc Section | Contents |
|-------------|----------|
| `Documentation/` | Protocol docs, vendor references |
| `Documentation/bosrpc/` | BOS RPC protocol docs |
| `Documentation/coreplugin/` | Core plugin reference |
| `Documentation/ctkd/` | CTKD protocol docs |
| `Documentation/voice-control/` | Voice control protocol (PlantUML diagrams) |
| `Documentation/voice-id/` | Voice ID protocol (PlantUML diagrams) |
| `Documentation/security/` | Security documentation |
| `Documentation/other/` | Miscellaneous docs |

### Router Documentation

**Does not exist.** Router is a planned project with no repository, no documentation, and no code.

---

## 3. CLAUDE.md and Agent Instruction Files

### AppKit CLAUDE.md
- **Location:** `Bragi-AppKit/CLAUDE.md`
- **Length:** 104 lines
- **Key sections:** Traps (9 documented), WHY (architectural principles), Cross-Repo Contracts, Cross-Repo Message Protocol, Non-Negotiables (9 rules)
- **Commands:** Not in CLAUDE.md — project-level CLAUDE.md at `~/Desktop/CLAUDE.md` contains development commands
- **Dev commands (from project CLAUDE.md):**
  ```bash
  # Backend
  cd backend && source venv/bin/activate && uvicorn main:app --reload --port 8009
  # Frontend
  cd frontend && npm run dev  # port 1422
  # Tests
  cd backend && PYTHONPATH=. ./venv/bin/pytest
  # Type check
  cd frontend && npx tsc --noEmit
  # Build
  cd frontend && npx vite build
  ```
- **Message check:** Must check `messages/pending/` for `*_TO_APPKIT_*` or `*_TO_ALL_*` every session

### Portal CLAUDE.md
- **Location:** `Bragi-portal/CLAUDE.md`
- **Length:** 426 lines
- **Key sections:** Spike Findings, Project overview, Commands, Architecture quick map, Bragi Builder Service Layer, Non-Negotiables, Developer Profile, Canvas bundle reference
- **Commands:**
  ```bash
  pnpm install                    # Install
  pnpm dev                        # Dev (mock data, no Supabase needed)
  pnpm build                      # Build types then portal
  pnpm typecheck                  # Type check all packages
  cd apps/portal && pnpm test     # Portal tests (vitest)
  cd apps/portal && pnpm test:e2e # Playwright E2E
  cd packages/types && pnpm test:coverage
  cd apps/portal && npx tsc --noEmit
  cd apps/portal && pnpm lint
  ```
- **Message check:** Must check vault messages (references `bragi-vault/messages/completed/`)

### MSDK CLAUDE.md
- **Location:** `msdk-ios/CLAUDE.md`
- **Length:** ~100 lines
- **Key sections:** Non-obvious constraints (Capacitor 8, MFi, AirohaRACEChannel), Cross-repo contracts, Release, Pending items, Vault messaging protocol
- **Commands:** Few explicit commands — uses Xcode + CocoaPods. CI via GitLab (see `gitlab.md` in docs)
- **Message check:** Every session start must check `messages/pending/` for `*_TO_MSDK_*` or `*_TO_ALL_*`
- **CI:** GitLab (referenced in `Documentation/gitlab.md`), not GitHub Actions
- **Remote:** `https://github.com/nihviid/msdk-ios.git` (personal namespace, to be moved to `bragi/` org)

---

## 4. README and Setup Instructions

### AppKit
- **README:** Not explicitly checked, but `docs/README.md` provides documentation reading guide
- **Onboarding:** `docs/onboarding/DEVELOPER_QUICKSTART.md` (5-minute setup guide)
- **Dependencies:** Python venv, Node.js, npm
- **Stack:** FastAPI + React 19 + Capacitor 8 + Tauri 2

### Portal
- **READ ME:** Not at root level, but `docs/README.md` provides documentation map
- **Onboarding:** `docs/onboarding/getting-started.md`, `docs/onboarding/codebase-tour.md`
- **Dependencies:** pnpm 10.8.1, Node 20, Supabase (optional for dev — mock data fallback)
- **Stack:** Next.js 16, React 19, TypeScript 5, Tailwind 4, shadcn/ui, Supabase

### MSDK
- **README:** `README.md` at repo root — detailed setup instructions (Xcode, CocoaPods, JIRA env vars)
- **Setup:** `dev-setup.sh` at repo root
- **Dependencies:** Xcode, CocoaPods (via Homebrew), JIRA credentials
- **Stack:** Swift, SPM, CocoaPods, Capacitor 8

### Router
- **None.** Does not exist.

---

## 5. Build/Test/Lint Commands

### AppKit
| Command | Purpose | Location |
|---------|---------|----------|
| `cd backend && PYTHONPATH=. ./venv/bin/pytest` | Run all backend tests | Repo root |
| `cd frontend && npx tsc --noEmit` | Frontend type check | Repo root |
| `cd frontend && npx vite build` | Frontend production build | Repo root |
| `cd backend && source venv/bin/activate && uvicorn main:app --reload` | Start backend dev | Repo root |
| `cd frontend && npm run dev` | Start frontend dev | Repo root |
| Lint baseline: 108 errors / 337 warnings (pre-existing) | Lint enforced via pre-commit | Repo root |

### Portal
| Command | Purpose |
|---------|---------|
| `pnpm install` | Install dependencies |
| `pnpm dev` | Dev server (mock data) |
| `pnpm build` | Build types + portal |
| `pnpm typecheck` | TypeScript type check |
| `cd apps/portal && pnpm test` | Run tests (vitest) |
| `cd apps/portal && pnpm test:e2e` | Playwright E2E tests |
| `cd apps/portal && pnpm lint` | Lint check |
| `cd packages/types && pnpm test:coverage` | Types package tests (97.51% coverage) |
| `supabase db push` | Apply migrations to cloud |

### MSDK
| Command | Purpose |
|---------|---------|
| Xcode build | Build via Xcode project |
| `pod install` | Install CocoaPods dependencies |
| SPM integration | Swift Package Manager for Capacitor |
| No explicit test commands in CLAUDE.md | Tests likely run via Xcode/xcodebuild |

### Router
No commands — does not exist.

---

## 6. Project-Specific Automation Conventions

### Shared Across Projects

**CLAUD E.md file pattern:** Every project has a `CLAUDE.md` at repo root containing:
- Project description and architecture
- Cross-repo contracts and interfaces
- Cross-repo message protocol (check `messages/pending/` first)
- "Never modify" rules
- Token discipline rules (identical across all projects — CI log retrieval, file reading, cross-repo grep, investigation discipline)

**Watcher agents:** Each project has a dedicated watcher agent in `bragi-vault/watchers/` that polls for messages via `/loop 60s` and flag files.

**Cross-repo messaging:** Every project's CLAUDE.md includes instructions to check `bragi-vault/messages/pending/` at session start.

### AppKit-Specific

- Pre-commit hooks enforce lint `--max-warnings 0` on new code
- CI blocks on new lint violations, not the baseline
- Has two separate requirements files (`requirements.txt` vs `requirements.docker.txt`)
- ML models guarded with `try/except ImportError`
- MCP server at `/mcp/sync` with 6 tools
- `@bragi-ai/devkit` CLI for manifest operations

### Portal-Specific

- Monorepo with pnpm workspaces (5 packages)
- Draft/live deployment model
- Supabase RLS for auth
- Bragi Builder service layer at `/api/v1/*`
- 14 unversioned API routes scheduled for sunset 2026-07-04
- Portal E2E tests validate against `ResolvedConfigContract`

### MSDK-Specific

- Releases via semver tags (no `v` prefix), current `2.1.0-preview.1`
- CI via GitLab (not GitHub Actions)
- Uses CocoaPods + SPM for dependency management
- Capacitor 8 native plugin architecture
- Hardware-gated stable releases

---

## 7. GitHub PR Conventions

### AppKit CI Workflows (20 workflows)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci.yml` | PR/push | Main CI pipeline |
| `contract-nightly.yml` | Schedule | Nightly contract tests |
| `frontend-checks.yml` | PR/push | Frontend checks |
| `gitlink-hygiene.yml` | PR/push | Git link hygiene |
| `localhost-url-lint.yml` | PR/push | No localhost URLs |
| `nightly-e2e.yml` | Schedule | Nightly E2E |
| `publish-manifest-renderer.yml` | Release | Publish manifest renderer |
| `quality-gate.yml` | PR/push | Quality gate |
| `test-*.yml` (12 workflows) | PR/push per domain | Test suites by domain |

### Portal CI Workflows (6 workflows)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `test.yml` | PR/push + schedule | Main test suite |
| `contract-tests.yml` | PR/push | Cross-repo contract tests |
| `docker-build.yml` | PR/push | Docker image build |
| `publish-cli.yml` | Release | Publish CLI tool |
| `publish-eslint-config.yml` | Release | Publish ESLint config |
| `publish-sdk.yml` | Release | Publish SDK |

### MSDK CI

**Uses GitLab CI** (referenced in `Documentation/gitlab.md`). No GitHub Actions workflows. The `msdk-ios/.github/` directory does not exist.

---

## 8. Existing Cross-Project References

### Contract Ownership

| Contract | Owner | Consumed By |
|----------|-------|-------------|
| `ResolvedConfigContract` (`@bragi-ai/types`) | Portal | AppKit, Portal E2E |
| `DeviceDescriptor` | Portal (designed) + AppKit (firmware-reported) | Portal, AppKit, mSDK |
| Protocol drivers (JS) | Portal (registry) | mSDK (DriverVerifier), AppKit (DriverLoader) |
| Ed25519 public key | mSDK (compiled-in) | AppKit (verification) |
| Brand i18n locks (`common.*`, `error.*`, etc.) | Portal (config) | AppKit (runtime) |
| Canvas bundles (`CanvasBundleSlot` + `SurfaceMountFn`) | Portal (registry) | AppKit (runtime) |

### Cross-Repo Interface Map

| Interface | Provider | Consumer(s) | Protocol |
|-----------|----------|-------------|----------|
| `GET /api/v1/config/resolved` | Portal | AppKit runtime | HTTPS + API key |
| `GET /api/v1/drivers/registry/*` | Portal | mSDK DriverVerifier | HTTPS + Ed25519 verification |
| `POST /platform/devices/{id}/capabilities` | AppKit | Portal | HTTPS |
| 26 Capacitor plugin methods | mSDK (native) | AppKit (JS) | Capacitor bridge |
| `POST /mcp/sync` | AppKit | Portal, Claude Code, external | FastMCP 3.0 |
| Bragi Builder `/api/v1/*` | Portal | AppKit, mSDK, AudioApp | HTTPS + API key/JWT |

---

## 9. Dependency Relationships

### Build/Release Order

```
Portal (types) ────► Portal (app)
       │
       ├────────────► AppKit (consumes config, drivers)
       │
       └────────────► mSDK (consumes signed driver bundles)

AppKit ────────────► mSDK (via Capacitor plugin bridge, at runtime)

All projects ──────► Bragi-portal (for E2E contract validation)
```

### Key Dependency Rules (from CLAUDE.md files)

1. **Portal first** — Portal schema changes (migrations) before Portal code changes. Portal code before AppKit.
2. **mSDK is independent** — mSDK ships independently (device-side, no server dependency).
3. **AppKit waits for Portal** — AppKit consumes Portal endpoints; new endpoints come before new AppKit features.
4. **@bragi-ai/types is the contract glue** — type changes need coordination before merging either side.
5. **New SoC = Portal push only** — no app release needed. Protocol logic lives in Portal-delivered JS drivers, not compiled into mSDK.
6. **Cross-repo changes go through `bragi-vault/messages/`** — never commit across repos.

---

## 10. Where Each Project Expects Collaboration Messages

All projects use the same protocol (defined in `bragi-vault/` and in every project's CLAUDE.md):

### Sending to a Project

Write to `bragi-vault/messages/pending/` with naming convention:
```
<FROM>_TO_<TO>_YYYY-MM-DD_<slug>.md
```

Then create the corresponding `.notify-<target>` flag file.

### Where Each Project Checks

| Project | Check Location | Check Frequency | CLAUDE.md Reference |
|---------|---------------|-----------------|---------------------|
| **AppKit** | `messages/pending/` for `*_TO_APPKIT_*` or `*_TO_ALL_*` | Every session start | AppKit CLAUDE.md |
| **Portal** | `messages/pending/` for `*_TO_PORTAL_*` or `*_TO_ALL_*` | Every session start | Portal CLAUDE.md |
| **MSDK** | `messages/pending/` for `*_TO_MSDK_*` or `*_TO_ALL_*` | Every session start | mSDK CLAUDE.md |
| **Watcher agents** | `.notify-<role>` flag in `messages/` | Every `/loop 60s` tick | Watcher CLAUDE.md files |

### BACP Bridge vs. Vault Messages

| Aspect | Vault Messages | BACP Bridge |
|--------|---------------|-------------|
| **Location** | `bragi-vault/messages/` | `~/.bragi/agent-control-plane/` |
| **Format** | Markdown + YAML frontmatter | JSON |
| **Actors** | Project developers, watchers | ChatGPT manager, CLI executors |
| **Lifecycle** | Permanent (git-tracked knowledge) | Ephemeral (90-day archive) |
| **CLI agents check** | Yes (session start) | Yes (task boundaries) |

---

## 11. Files/Folders Never to Modify Automatically

| Path | Reason | Status |
|------|--------|--------|
| `Bragi-AppKit/.git/` | Git internals | Never |
| `Bragi-AppKit/node_modules/` | Dependencies | Never |
| `Bragi-AppKit/frontend/node_modules/` | Dependencies | Never |
| `Bragi-AppKit/backend/venv/` | Python virtual env | Never |
| `Bragi-portal/.git/` | Git internals | Never |
| `Bragi-portal/node_modules/` | Dependencies | Never |
| `Bragi-portal/apps/portal/node_modules/` | Dependencies | Never |
| `msdk-ios/.git/` | Git internals | Never |
| `msdk-ios/Pods/` | CocoaPods dependencies | Never |
| `bragi-vault/appkit-docs/` | Symlink to AppKit | Never (read-only) |
| `bragi-vault/portal-docs/` | Symlink to Portal | Never (read-only) |
| `bragi-vault/msdk-docs/` | Symlink to mSDK | Never (read-only) |
| `bragi-vault/appkit-gsd/` | Symlink to AppKit `.planning/` | Never (read-only) |
| `bragi-vault/portal-gsd/` | Symlink to Portal `.planning/` | Never (read-only) |
| `bragi-vault/msdk-gsd/` | Symlink to mSDK `.planning/` | Never (read-only) |
| `bragi-vault/messages/pending/` | Active watcher agents | Never (BACP has own bridge) |
| `bragi-vault/messages/.notify-*` | Watcher agent control | Never (BACP has own flags) |
| `~/.bragi/agent-control-plane/STOP` | Kill switch | Never — human only |

### Read-Only During Discovery

Everything in `Bragi-AppKit/`, `Bragi-portal/`, `msdk-ios/`, `bragi-vault/`, and `Obsidian_Vault/` is read-only during discovery. The only write target is `~/.bragi/agent-control-plane/` and `Bragi-AgentCP/`.

---

## 12. How BACP Should Create Manager/Executor Configs

### Per-Project Manager Config Template

Each project needs a BACP manager configuration that captures:

```json
{
  "project": "appkit",
  "repo_path": "~/Desktop/Antigravity/Bragi-AppKit/",
  "remote": "git@github.com:nihviid/Bragi-AppKit.git",
  "claude_config": "Bragi-AppKit/CLAUDE.md",
  "docs_path": "Bragi-AppKit/docs/",
  "watcher_agent": "watchers/appkit/",
  "message_pattern": "*_TO_APPKIT_*",
  "flag_file": ".notify-appkit",
  "allowed_bridge_messages": ["instruction", "decision", "acknowledgment", "override"],
  "constraints": [
    "Never modify symlinked bragi-vault/appkit-docs/",
    "Never modify files outside Bragi-AppKit/",
    "Cross-repo changes go through bragi-vault/messages/",
    "Do not edit bragi-vault/ from this repo"
  ],
  "critical_paths": {
    "never_modify": [
      ".git/", "node_modules/", "venv/", "Pods/"
    ],
    "read_only": [
      "bragi-vault/appkit-docs/",
      "bragi-vault/appkit-gsd/"
    ]
  }
}
```

### Configuration Lifecycle

| Phase | Config Source | Purpose |
|-------|--------------|---------|
| **Discovery** | Hardcoded in BACP docs | Maps the landscape |
| **Design** | Structured JSON in `~/.bragi/agent-control-plane/` | Machine-readable |
| **Operation** | Sourced from config files, consumed by BACP agents | Active use |

### Recommended Config Directory

```
~/.bragi/agent-control-plane/projects/
├── appkit.json       # AppKit project config
├── portal.json       # Portal project config
├── msdk.json         # mSDK project config
└── router.json       # Router project config (future)
```

These configs should be created during the BACP design phase (not now).

---

## Discovery Notes

- All explorations were read-only
- No files were created in any managed project
- No messages were sent, moved, or modified
- AppKit, Portal, and MSDK are fully documented. Router does not exist.
- MSDK uses GitLab CI, not GitHub Actions — noted for cross-platform CI planning
- All projects enforce the same cross-repo messaging protocol from `bragi-vault/`

## Files Examined

- `Bragi-AppKit/CLAUDE.md` (full)
- `Bragi-AppKit/docs/README.md`
- `Bragi-AppKit/.github/workflows/` (20 files listed)
- `Bragi-portal/CLAUDE.md` (full, 426 lines)
- `Bragi-portal/docs/README.md`
- `Bragi-portal/.github/workflows/` (6 files listed)
- `msdk-ios/CLAUDE.md` (full)
- `msdk-ios/README.md` (full)
- `msdk-ios/Documentation/` (listed contents)
- `~/Desktop/CLAUDE.md` (project-level, dev commands)
- `Bragi-Nerve/CLAUDE.md` (head 30)
- `bragi-vault/watchers/appkit/CLAUDE.md`
- `bragi-vault/watchers/portal/CLAUDE.md`
