# Discovery Pass 5 — Cross-Project Collaboration Flows

**Pass:** 5 of 10
**Date:** 2026-05-03
**Inspector:** BACP Discovery Agent
**Status:** Complete
**Bridge Control Status:** Bridge ACTIVE — reports via Phase 2 helper (`bacp-bridge`)

---

## 1. Canonical Collaboration Flow

The cross-project collaboration flow follows an 8-step cycle. This is the canonical sequence observed across all 300+ completed messages.

### Step 1: Request Creation
A project identifies a need for collaboration. The sender writes a markdown file to `bragi-vault/messages/pending/` with YAML frontmatter containing `from`, `to`, `status: pending`, `date`, `subject`, and optional fields (`milestone`, `relates-to`, `priority`). The body uses structured sections: Context, Question/Request, and What-you've-already-checked.

**Naming convention:** `<FROM>_TO_<TO>_YYYY-MM-DD_<slug>.md`

### Step 2: Flag Creation
The sender creates an empty flag file at `bragi-vault/messages/.notify-<target>`. The flag file is zero bytes. Its mere presence signals to the target agent that a message awaits.

**Flag does not identify which message.** The target agent must scan `messages/pending/` for messages matching its identity.

### Step 3: Flag Detection (Watcher Polling)
The target project's watcher agent runs via `/loop 60s`. On each tick:
1. Check if flag file `.notify-<role>` exists
2. If flag does NOT exist: report "No flag — idle", yield immediately
3. If flag EXISTS: proceed to Step 4

**The flag is an on/off signal.** No flag = zero work. This keeps watcher costs near-zero between messages.

### Step 4: Flag Deletion and Message Reading
The watcher:
1. Deletes the flag file (atomic consumption of the signal)
2. Scans `messages/pending/` for files matching `*_TO_<ROLE>_*` or `*_TO_ALL_*`
3. Reads all matching messages

**Race condition risk:** If two watchers scan simultaneously, both could see the flag, both delete it (second delete silently fails), both read the same messages, and both respond. Mitigation: messages with unambiguous `to` field target only one role.

### Step 5: Processing
The watcher processes each message:
- **Research:** Reads code, docs, and vault as needed to answer
- **Analyze:** Determines the correct response
- **Draft:** Writes the response under `## Response` in the original message file

**Watcher rules** (from CLAUDE.md):
- Be specific. Cite file paths and line numbers.
- If the question requires reading code, read the code — don't guess.
- If you don't know, say so. Don't fabricate.
- If answering requires a code change, describe what to change but do NOT make the change. Tag as `ACTION NEEDED:`.

### Step 6: Response Creation
The watcher changes frontmatter `status: pending` → `status: responded`. The response goes in the same file under `## Response`. No new file is created.

### Step 7: Completion and Archival
The watcher moves the file from `messages/pending/` → `messages/completed/`. This is a filesystem move, not a copy. The file remains under `completed/` permanently as a git-tracked record.

### Step 8: Source Project Follow-Up
The original sender checks `messages/completed/` on their next session for files matching `<ROLE>_TO_<FROM>_*` with `status: responded`. They read the response and continue work. They may send a follow-up by starting a new cycle at Step 1 referencing the previous message via `relates-to`.

---

## 2. Message Lifecycle

```
CREATED (sender writes file + flag)
   │
   ▼
PENDING (file in messages/pending/, flag file exists)
   │
   ▼
DETECTED (watcher sees flag on /loop 60s tick)
   │
   ▼
PROCESSING (watcher deletes flag, reads file, researches, writes response)
   │
   ▼
RESPONDED (frontmatter status changed to "responded")
   │
   ▼
COMPLETED (file moved to messages/completed/)
   │
   ▼
FOLLOWED UP (sender reads response on next session)
```

### State Transitions

| State | File Location | Flag Exists? | Who Changes It |
|-------|---------------|-------------|----------------|
| `pending` | `messages/pending/` | Yes | Sender (writes) |
| `responded` | `messages/completed/` | No | Watcher (changes status + moves file) |
| `closed` | `messages/completed/` | No | Sender (optional — marks as closed in frontmatter) |

### Acknowledgement Behavior

There is no explicit "acknowledgement" message. The lifecycle implicitly acknowledges:
- Flag deletion = acknowledgement of receipt
- Status change to `responded` = acknowledgement of processing
- File move to `completed/` = acknowledgement of completion

**Implication:** There is no mechanism for the sender to confirm they read the response. If the sender never checks `completed/`, the response sits there unread indefinitely.

---

## 3. Cross-Project Payload Structure

### Message Frontmatter Fields

| Field | Required | Values | Purpose |
|-------|----------|--------|---------|
| `from` | Yes | `portal`, `appkit`, `msdk`, `audioapp` | Sender identity |
| `to` | Yes | `portal`, `appkit`, `msdk`, `all` | Target identity |
| `type` | Yes | `question`, `response`, `announcement`, `status-update`, `smoke-report`, `decision`, `request`, `blocked` | Message category |
| `status` | Yes | `pending`, `responded`, `closed` | Lifecycle state |
| `date` | Yes | `YYYY-MM-DD` | Creation date |
| `subject` | Yes | Free text | One-line summary |
| `priority` | No | `normal`, `high`, `urgent` | Urgency indicator |
| `milestone` | No | e.g., `M1`, `M2`, `m1-followup`, `ci-hygiene` | Milestone tracking |
| `relates-to` | No | Paths to related messages | Threading |

### Body Structure

The message body follows a consistent pattern:

```
## Context
{What the sender is working on, why this message exists}

## Question / Request
{The specific thing needed from the target}

## What you've already checked
{What the sender investigated so the responder doesn't duplicate effort}

---

## Response
{Written by the watcher — answers, recommendations, CODE: tags}
```

### Response Conventions

From watcher CLAUDE.md files:
- **Be specific.** Cite file paths and line numbers.
- **Recommend, don't decide.** Prefix with `RECOMMENDATION:` for architecture calls.
- **Tag required actions.** Prefix blocking items with `ACTION NEEDED:`.
- **Reference evidence.** Include relevant code snippets, test output, commit hashes.

### Cross-Project Evidence in Messages

Messages frequently include:
- GitHub PR URLs (`https://github.com/nihviid/bragi-portal/pull/23`)
- Commit SHAs (`621769d`, `5e893b1`)
- Curl commands with output (for API verification)
- Test results (pass/fail counts, test names)
- Configuration snippets (JSON, YAML)
- Timeline references (deploy times, milestone dates)

---

## 4. Project Roles

### Portal (Orchestrator)

**Responsibilities:**
- Initiates the orchestration loop (PLAN → RESOLVE → CODE → TEST → DEPLOY → ALIGN)
- Ships handover packages to AppKit, mSDK, AudioApp
- Owns cross-repo decisions (bsolve orchestrator)
- Publishes `@bragi-ai/types` — shared contract source of truth
- Runs integration tests after all CODE handbacks received
- Owns schema changes (migrations before code)
- Deploys Portal code before AppKit consumes new endpoints

**Messages sent:** PORTAL_TO_APPKIT, PORTAL_TO_ALL, PORTAL_TO_MSDK

**Communication style:** Authoritative on contracts and deployment order. `status-update` messages reference specific PR commits and CI results.

### AppKit (Primary Executor)

**Responsibilities:**
- Implements runtime behavior (Kits, audio-app engine, Bragi API)
- Consumes Portal configs, drivers, and manifests
- Runs mSDK via Capacitor plugin bridge
- Reports device capabilities to Portal
- Ships MCP server for developer tooling

**Messages sent:** APPKIT_TO_ALL (milestone announcements), APPKIT_TO_PORTAL (status, questions), APPKIT_TO_AUDIOAPP

**Communication style:** Detailed milestone completion reports with test counts (7203 backend, 1102 frontend). Technical questions include reproduction steps and error output.

### mSDK (Transport Layer)

**Responsibilities:**
- Provides native BLE/MFi/iAP2 transport through Capacitor plugin
- Verifies Portal-signed driver bundles (Ed25519)
- Reports device capabilities
- Ships independently (device-side, no server dependency)

**Messages sent:** MSDK_TO_ALL (authoritative ground-truth responses)

**Communication style:** Ground-truth authority on transport-layer questions. Responds with code evidence. Responses address multiple repos simultaneously (combined response to Portal + AppKit parallel questions).

### Router

**Not yet created.** Listed as a managed project but has no repository, no code, no documentation, and no messages. No role defined.

### Project Role Summary

| Project | Role | Orchestrates? | Consumes From | Ships To |
|---------|------|---------------|---------------|----------|
| Portal | Orchestrator, contract authority | Yes | — | All |
| AppKit | Runtime executor | No | Portal, mSDK | Portal, AudioApp |
| mSDK | Transport plugin | No | Portal | AppKit (via bridge) |
| Router | Not yet | — | — | — |

---

## 5. Watcher Behavior

### Watcher Agent Roster

| Watcher | Role | Poll Interval | Message Pattern |
|---------|------|---------------|-----------------|
| `appkit` | AppKit codebase expert | `/loop 60s` | `*_TO_APPKIT_*`, `*_TO_ALL_*` |
| `portal` | Portal codebase expert | `/loop 60s` | `*_TO_PORTAL_*`, `*_TO_ALL_*` |
| `msdk` | mSDK codebase expert | `/loop 60s` | `*_TO_MSDK_*`, `*_TO_ALL_*` |
| `cto` | Architecture advisory | `/loop 60s` | `*_TO_CTO_*`, `*_TO_ALL_*` |
| `ceo` | Business advisory | `/loop 60s` | `*_TO_CEO_*`, `*_TO_ALL_*` |
| `cpo` | Product advisory | `/loop 60s` | `*_TO_CPO_*`, `*_TO_ALL_*` |
| `cro` | Revenue advisory | `/loop 60s` | `*_TO_CRO_*`, `*_TO_ALL_*` |
| `consultant` | Consulting advisory | `/loop 60s` | `*_TO_CONSULTANT_*`, `*_TO_ALL_*` |

### Flag-Delete-Then-Process Pattern

Every watcher follows the exact same pattern:

```python
# Pseudocode for the watcher loop
while True:
    if not os.path.exists(f"messages/.notify-{role}"):
        print("No flag — idle")
        sleep(60)
        continue

    os.remove(f"messages/.notify-{role}")  # Delete flag
    messages = glob("messages/pending/*_TO_{ROLE}_* messages/pending/*_TO_ALL_*")
    for msg in messages:
        content = read(msg)
        response = process(content)
        write_response(msg, response)  # Writes under ## Response
        set_status(msg, "responded")
        move(msg, "messages/completed/")
    
    sleep(60)
```

### Race Condition Risks

| Risk | Scenario | Likelihood | Current Mitigation |
|------|----------|------------|-------------------|
| Double-processing | Two agents detect same flag before either deletes it | Low (single-writer /loop) | Flag targeting by role reduces collisions |
| Stale flag | Flag created but no watcher online | Medium | Flag persists until deleted by watcher. If watcher never returns, flag stays forever. |
| Lost message | Message written before flag, flag created but watcher doesn't scan pending/ | Low (flag and message write are sequential in practice) | None |
| Orphaned flag | Flag created but message writing failed | Low | Human must notice and clean up |

---

## 6. Failure and Retry Behavior

### Failed Messages

The system has no formal failure state. Messages that cannot be processed stay in `messages/pending/` indefinitely. There is no timeout, no expiry, and no retry mechanism.

**Observed failure patterns:**

| Pattern | Cause | Resolution |
|---------|-------|------------|
| Stale pending | Watcher offline, or message targeted wrong role | Human notices and either resolves or moves to completed manually |
| Unclear ownership | Message targets `all` but requires specific expertise | Advisory watchers (CTO, CEO) pick it up, or nobody does |
| Partial response | Watcher answers some questions but not all | Mitigation in watcher CLAUDE.md: "If you don't know, say so" |
| Repeated attempts | Sender sends same question multiple times | No deduplication mechanism. Sequential messages pile up in completed/. |

### Retry Behavior

There is no automatic retry. The only retry mechanism is the sender:
1. Checks `completed/` — if no response found, the message is still in `pending/`
2. Checks `pending/` — if message is still there, the watcher hasn't processed it
3. May re-create the flag file to trigger re-processing
4. May send a follow-up message referencing the original

### Stale Request Handling

Stale pending messages accumulate. As of 2026-05-03, 4 pending messages exist (all PORTAL_TO_APPKIT from 2026-04-29/30). These were not processed because the AppKit watcher did not poll during that period. There is no cleanup mechanism — stale messages remain in `pending/` forever unless a human notices and acts.

---

## 7. GitHub References

### PR Linking Convention

Messages reference PRs with full GitHub URLs:
```
https://github.com/nihviid/bragi-portal/pull/23
https://github.com/nihviid/Bragi-AppKit/pull/42
```

### Commit References

Commits are referenced by short SHA (`621769d`, `5e893b1`). Messages include commit messages inline.

### Branch Naming

No formal cross-project branch naming convention observed. Each project uses its own conventions:
- Portal: `feature/descriptive-name`
- AppKit: Follows phase naming (`phase-41-*`)
- mSDK: semver tags (`2.1.0-preview.1`)

### Review Expectations

- Portal describes specific commits in messages to AppKit
- Test results are cited as evidence
- Cross-repo contracts require coordination before merging either side
- Milestone checklists are tracked in `bragi-vault/coordination/MILESTONE-STATUS.md`

---

## 8. BACP Control Implications

### What BACP Should Observe

| Observation Point | What to Watch | Why |
|------------------|---------------|-----|
| `messages/pending/` | New messages and their targets | Detect collaboration requests entering the system |
| `messages/completed/` | Response times and patterns | Measure cycle time, detect stale requests |
| `messages/.notify-*` | Flag creation events | Trigger for manager awareness of active collaboration |
| `coordination/MILESTONE-STATUS.md` | Milestone completion state | Manager needs to know what phase each project is in |
| Watcher `/loop` output | Watcher health and activity | Detect offline watchers before they cause backlog |

### What BACP Should Not Interfere With

| Do Not Touch | Why |
|-------------|-----|
| `messages/pending/` files | Active watcher agents own these. BACP has its own bridge at `~/.bragi/`. |
| `messages/.notify-*` flags | Watcher control signals. BACP writes `.notify-manager` if needed, never `.notify-appkit` etc. |
| Watcher CLAUDE.md files | Existing agent instructions. BACP does not modify them. |
| `coordination/` documents | Portal owns the orchestration loop. BACP is a separate layer. |
| Project source code | Read-only during discovery. Never modify. |

### Where BACP Should Create Decision Requests

| Decision Type | BACP Role | Route To |
|--------------|-----------|----------|
| Cross-project coordination timing | Observe and report | Manager |
| Watcher health degradation | Flag to manager | Manager |
| Stale pending messages | Alert to manager | Manager |
| Contract conflict detected | Create decision request | Manager → Human |
| Architecture questions | Forward to CTO watcher | Manager via vault messages |

### Where BACP Should Integrate Later

| Integration | Phase | Description |
|-------------|-------|-------------|
| Message lifecycle awareness | Phase 3+ | BACP reads `messages/pending/` to detect new collaboration requests involving BACP-managed projects |
| Watcher health monitoring | Phase 3+ | BACP checks watcher `/loop` output for signs of failure or backlog |
| Cross-loop notification | Phase 3+ | When BACP manager makes a decision affecting a project, optionally notify via vault messages |
| Dashboard visualization | Phase 3 | Show active vault messages alongside bridge state |
| Relay bridge → vault | Phase 4 | When a BACP exchange produces a durable insight, optionally write to `bragi-vault/decisions/` |

### Risks That Must Be Guarded by Manager Control

| Risk | Guard | Mechanism |
|------|-------|-----------|
| BACP conflicts with watcher agents | Never write to `messages/pending/` or touch `.notify-*` flags | CLAUDE.md rules + bridge isolation |
| BACP misses cross-project decisions | Manager reads coordination/ and decisions/ regularly | Manager instruction cycle |
| BACP creates parallel communication channels | All BACP communication goes through `~/.bragi/` bridge | Architecture constraint |
| Watcher backlog grows undetected | Manager requests periodic status on watcher health | Manager instruction |
| Stale messages corrupt decision process | Manager reviews pending messages before issuing instructions | Verification loop step |

---

## 9. Non-Interference Rules

| Rule | Rationale |
|------|-----------|
| **Never write to `messages/pending/`** | Active watcher agents own this directory. BACP messages go through `~/.bragi/`. |
| **Never create `.notify-*` flags** | Watcher control signals. BACP creates no flags in `bragi-vault/messages/`. |
| **Never modify messages in `completed/`** | Permanent git-tracked history. Read-only. |
| **Never modify watcher agent configs** | Each watcher's CLAUDE.md and knowledge base are owned by the watcher. |
| **Never modify coordination documents** | Portal owns the orchestration loop documents. |
| **Never move files between pending/completed** | Only the watcher that processed the message moves it. |
| **Never send cross-project messages via vault** | BACP is not a project. It does not participate in the vault message protocol. |
| **Never merge or branch in project repos** | BACP works in its own repo. Changes to managed projects happen through the bridge. |

---

## 10. Future Control-Plane Design Implications

### What the Data Tells Us

1. **The vault message protocol is well-defined but fragile.** No timeout, no retry, no staleness detection. BACP can address these at the bridge level without modifying the vault.

2. **Watcher agents are the right abstraction but single-threaded.** Each project has one watcher. If it goes offline, messages pile up. BACP manager-level monitoring can detect this.

3. **Portal is the existing orchestrator.** BACP does not replace Portal. BACP operates at a higher level (manager → executor coordination) while Portal operates at the project level (task definition, handover packages, milestone tracking).

4. **Coordination is document-driven, not API-driven.** Collaboration works through files, flags, and git-tracked markdown. BACP's file-based bridge is architecturally consistent with this pattern.

5. **There is no cross-project notification.** Projects discover collaboration requests by polling flags. There is no push mechanism. BACP's manager->CLI instruction flow is architecturally different (push via the conversation / inbox polling).

### BACP's Role in the Ecosystem

```
┌────────────────────────────────────────────────────┐
│                  Human                             │
│  (overrides all via STOP, direct instruction)       │
└────────────────────┬───────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────┐
│            BACP Manager (ChatGPT)                   │
│  - Issues instructions to CLI executors              │
│  - Reads bridge state for awareness                  │
│  - Creates decision requests to human                │
│  - Monitors watcher health (read-only)               │
└──┬────────────────┬────────────────┬───────────────┘
   │                │                │
   ▼                ▼                ▼
┌────────┐    ┌────────┐     ┌──────────────┐
│ CLI    │    │ CLI    │     │ Watcher      │
│ AppKit │    │ Portal │     │ Agents       │
│        │    │        │     │ (existing,   │
│ bridge │    │ bridge │     │ unchanged)   │
│ reports│    │ reports│     │              │
└────────┘    └────────┘     └──────────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │ bragi-vault       │
                            │ messages/pending/ │
                            │ messages/completed/│
                            │ flags             │
                            └──────────────────┘
```

**Key insight:** BACP and the vault protocol are parallel, not layered. BACP does not sit between projects. It sits beside the vault protocol, providing a different communication channel (manager↔executor) for a different purpose (operational coordination vs. cross-project knowledge).

---

## Canonical Collaboration Flow

**Step 1:** Sender writes a markdown message to `messages/pending/` with frontmatter (`from`, `to`, `status: pending`, `date`, `subject`, optional `milestone`, `relates-to`, `priority`) and body sections (`Context`, `Question/Request`, `What you've already checked`). Naming convention: `<FROM>_TO_<TO>_YYYY-MM-DD_<slug>.md`.

**Step 2:** Sender creates an empty `.notify-<target>` flag file in `messages/`. The flag is a zero-byte marker. Its presence is the only signal — it does not identify which message.

**Step 3:** Target watcher agent, running via `/loop 60s`, detects the flag file exists on its next polling tick. If no flag, the agent reports "No flag — idle" and yields immediately (near-zero cost between messages).

**Step 4:** Watcher deletes the flag file (consuming the signal, preventing re-processing), then scans `messages/pending/` for files matching `*_TO_<ROLE>_*` or `*_TO_ALL_*`. Reads all matching messages.

**Step 5:** Watcher processes each message: reads code/docs as needed for research, analyzes the request, drafts the response. Rules: cite file paths, don't guess, don't fabricate, tag required changes as `ACTION NEEDED:`.

**Step 6:** Watcher writes the response under `## Response` in the same file, then changes frontmatter `status: pending` → `status: responded`.

**Step 7:** Watcher moves the file from `messages/pending/` to `messages/completed/` via filesystem move. The file in `completed/` is permanent (git-tracked).

**Step 8:** On their next session, the original sender checks `messages/completed/` for files matching `<ROLE>_TO_<FROM>_*` with `status: responded`. Reads the response. May send a follow-up referencing the previous message via `relates-to` field. No mechanism exists for the sender to confirm they read the response.

---

## Discovery Notes

- Total messages observed: 300+ completed, 4 pending
- Total watcher agent configs examined: 8 (appkit, portal, msdk, cto, ceo, cpo, cro, consultant)
- Coordination documents examined: 3 (ORCHESTRATION.md, MILESTONE-STATUS.md, types-release-status.md)
- No files were created or modified in external projects
- All exploration was read-only

## Files Examined

- `bragi-vault/coordination/ORCHESTRATION.md`
- `bragi-vault/coordination/MILESTONE-STATUS.md`
- `bragi-vault/messages/completed/APPKIT_TO_PORTAL_2026-04-29_af09-smoke-complete.md`
- `bragi-vault/messages/completed/APPKIT_TO_PORTAL_2026-04-18_v2-route-loader-regression.md`
- `bragi-vault/messages/completed/APPKIT_TO_ALL_2026-04-23_phase-23-mcp-server-complete.md`
- `bragi-vault/messages/completed/MSDK_TO_ALL_2026-04-28_driver-pubkey-format-ground-truth-response.md`
- `bragi-vault/messages/pending/PORTAL_TO_APPKIT_2026-04-29_e2e-driver-registry-contract-realignment.md`
- `bragi-vault/messages/pending/PORTAL_TO_APPKIT_2026-04-30_phase41-device-descriptor-confirmed.md`
- `bragi-vault/watchers/appkit/CLAUDE.md`
- `bragi-vault/watchers/portal/CLAUDE.md`
- `bragi-vault/watchers/msdk/CLAUDE.md`
- `bragi-vault/watchers/cto/CLAUDE.md`
