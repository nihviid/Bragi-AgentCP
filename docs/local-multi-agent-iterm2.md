# Local Multi-Agent System — iTerm2 Architecture

**Date:** 2026-05-03  
**Target:** Prove portal management with Manager / Executor / Terminal  

---

## 1. Architecture Overview

One iTerm2 window, three panes, one message bus. Zero backend infrastructure.

```
┌────────────────────────────────────────────────────────┐
│                    iTerm2 Window                         │
│                                                          │
│  ┌──────────────────────┬──────────────────────────────┐│
│  │  MANAGER              │  EXECUTOR                    ││
│  │                       │                              ││
│  │  Access:              │  Access:                     ││
│  │  - message bus only   │  - message bus               ││
│  │  - ~/.bragi/agent-    │  - project files             ││
│  │    control-plane/     │  - terminal output logs      ││
│  │                       │  - bacp-bridge commands      ││
│  │  Cannot:              │                              ││
│  │  - read project files │  Cannot:                     ││
│  │  - run terminal cmds  │  - manage other executors    ││
│  │  - access git/npm/etc │  - create project strategy   ││
│  │                       │                              ││
│  ├───────────────────────┴──────────────────────────────┤│
│  │  TERMINAL                                             ││
│  │                                                       ││
│  │  Access:                                               ││
│  │  - project files (cd ~/portal)                        ││
│  │  - git, npm, pytest, npx playwright                   ││
│  │  - Chrome (via Playwright)                            ││
│  │                                                       ││
│  │  Owned by Executor — Executor writes commands,         ││
│  │  Terminal executes, output is captured back to         ││
│  │  Executor.                                             ││
│  └───────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────┐
│  MESSAGE BUS (~/.bragi/agent-control-plane/)             │
│                                                          │
│  inbox/manager-to-cli/     ← Manager writes instructions │
│  outbox/cli-to-manager/    ← Executor writes reports     │
│  decisions/pending/        ← Decision requests           │
│  decisions/answered/       ← Resolved decisions          │
│  archive/                  ← All messages preserved      │
│  STOP                      ← Kill switch                 │
│                                                          │
│  Manager reads from outbox/. Writes to inbox/.           │
│  Executor reads from inbox/. Writes to outbox/.          │
│  They never share a filesystem beyond the bus.           │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Pane Roles and Boundaries

### Manager Pane

**Purpose:** Pure decision-making agent. Reads reports, calls LLM, writes instructions.

**Access granted:**
- `~/.bragi/agent-control-plane/` (message bus only)
- `bacp-bridge` commands: `status`, `next-report`, `write-instruction`, `decisions`, `list-*`, `ack`, `autocontrol`
- LLM API (OpenAI / Anthropic) for generating instructions
- No project files, no git, no npm, no Playwright, no terminal

**Why this restriction:**
- Prevents the manager from drifting into implementation
- Forces clean task decomposition (manager must write clear instructions, cannot just "fix it")
- Keeps the manager small, stateless, and replaceable
- Enables swapping the manager model (Haiku for routing, Sonnet for planning, Opus for architecture) without touching execution

**Startup:** `cd /tmp && bacp-bridge autocontrol --watch --interval 5`

### Executor Pane

**Purpose:** Implementation agent. Reads instructions, writes code, runs terminal commands, reports results.

**Access granted:**
- `~/.bragi/agent-control-plane/` (message bus)
- Project files (Portal, AppKit, etc.)
- `bacp-bridge` commands: all
- LLM API for code generation
- Can read Terminal pane output (from log file)
- Can write commands to Terminal pane stdin

**Cannot:**
- Manage other executors
- Create project strategy or architecture decisions
- Merge or release without explicit approval

**Startup:** New `bacp-bridge execute --watch` command (see Section 5)

### Terminal Pane

**Purpose:** Pure execution environment. Runs commands, shows output.

**Access granted:**
- Project files
- git, npm, pytest, Playwright, Chrome, any CLI tool
- No message bus access
- No LLM calls
- No decision-making

**Output:** stdout/stderr continuously logged to `~/.bragi/agent-control-plane/logs/terminal/` for Executor to read.

---

## 3. iTerm2 Layout Script

A Python script using the iTerm2 Python API that:

1. Opens a new window
2. Creates three panes in the layout above
3. Sets each pane's working directory and environment
4. Sources project-specific setup (virtualenv, .env, nvm)
5. Starts the appropriate watch loop in each pane
6. Names each pane for identification

```python
# Pseudocode for the layout script
import iterm2

async def main(connection):
    app = await iterm2.async_get_app(connection)
    window = await iterm2.Window.async_create(connection)

    # Split vertically: left = Manager, right = Executor + Terminal
    left = await window.async_get_current_session()
    right = await left.async_split_pane(vertical=True)

    # Split right pane horizontally: top = Executor, bottom = Terminal
    terminal = await right.async_split_pane(vertical=False)

    # Configure each pane
    await setup_pane(left, "manager", "~/portal", "bacp-bridge autocontrol --watch")
    await setup_pane(right, "executor", "~/portal", "bacp-bridge execute --watch")
    await setup_pane(terminal, "terminal", "~/portal", "echo 'Terminal ready'")

    # Start terminal capture
    await start_terminal_capture(terminal)
```

---

## 4. Terminal Capture Service

A small background process that captures all Terminal pane output and makes it available to the Executor.

**Behavior:**
1. Appends every line from Terminal pane to `~/.bragi/agent-control-plane/logs/terminal/session.log`
2. Rotates log per task (new file when new instruction is read)
3. Executor can read the log to see command output
4. iTerm2 Python API streams pane content via async callbacks

**Why not just let the Executor run commands directly?**
- The Terminal pane gives a live view for the human watching
- Commands that need interaction (Playwright, Chrome) need a real terminal
- Captured output persists after the command finishes

---

## 5. New bacp-bridge Commands

### `bacp-bridge execute --watch`

New command for the Executor pane. Behavior:

1. Poll `inbox/manager-to-cli/` for new instructions
2. When instruction appears, print it with a clear header
3. Optionally run a pre-task script (git pull, nvm use, source venv)
4. Wait for Executor to write code / prepare commands
5. Write commands to the Terminal pane (via iTerm2 Python API or stdin relay)
6. Wait for Terminal output to stabilize (timeout + output detection)
7. Read captured terminal output
8. Call LLM to interpret results (tests passed? errors?)
9. Write structured report to `outbox/cli-to-manager/`
10. ACK the instruction
11. Loop

### `bacp-bridge terminal capture`

New command that starts terminal output capture.

### `bacp-bridge pane <name>`

List or switch focus between panes.

---

## 6. Flow for a Typical Task

```
1. Manager writes instruction:
   "Add a status badge to the Portal README showing
    test status. Only modify README.md."

2. Executor reads instruction → writes code →
   writes to Terminal pane:  echo '...' >> README.md

3. Terminal executes → output captured →
   Executor reads captured output → confirms file changed

4. Executor writes report:
   { task: "add status badge", status: "complete",
     file: "README.md", diff: "+3 lines" }

5. Manager reads report → verifies scope respected →
   issues next instruction or approves

6. Human watches all three panes in real time.
   If something looks wrong, types Ctrl+C in the
   Executor or Terminal pane.
```

---

## 7. Security and Boundaries

| Boundary | Enforced by |
|----------|-------------|
| Manager cannot touch project files | Runs from `/tmp`, no `cd ~/portal` |
| Manager cannot run terminal commands | No shell access in pane |
| Executor writes to Terminal, not directly | Terminal pane is the only execution context |
| Terminal has no LLM access | No API keys in environment |
| Kill switch stops all agents | STOP file checked before every action |
| Manager cannot approve its own work | All reports verified by human or separate gate |
| No cross-project contamination | Per-project pane sets (no shared state) |

---

## 8. Implementation Phases

### Phase 1 — Layout + Basic Flow

- iTerm2 layout script (3 panes)
- Manager pane: `autocontrol --watch` works (already built)
- Terminal pane: manual command entry
- Executor pane: reads instructions, human types commands in Terminal manually
- Terminal capture disabled

### Phase 2 — Terminal Capture

- Terminal pane output automatically logged
- Executor can read terminal output
- Executor can suggest commands (human reviews before execution)

### Phase 3 — Executor Watch Loop

- `bacp-bridge execute --watch` implemented
- Executor reads instructions and runs commands autonomously
- Terminal output feeds back into report

### Phase 4 — Manager Clean Room

- Manager pane runs from isolated directory
- No project access verified
- Manager prompt hardened (MOS rules enforced by system)

### Phase 5 — First Real Task on Portal

- Full cycle: Manager → Executor → Terminal → report
- Real Portal task (documentation, test, or small feature)
- Human observes all three panes

---

## 9. Key Design Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| iTerm2 vs tmux | iTerm2 | Python API, native macOS, pane scripting |
| Manager isolation | Directory-based | Simple, no container needed, verifiable |
| Terminal capture | File-based log | Works with any terminal, no IPC complexity |
| Message bus | Filesystem (existing) | Already works, zero new infrastructure |
| Executor LLM | API call | Fastest path, model flexibility |
| Human override | Direct terminal input | Any pane can be typed in directly |
