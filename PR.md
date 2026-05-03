# Introduce Manager-Control CLI Bridge Tool

**Branch:** `agent/bootstrap-discovery`  
**23 commits** across design, implementation, validation, and operating-system definition.

---

## What This Is

A CLI tool (`scripts/bacp-bridge`) that enables a ChatGPT manager to control a CLI executor through a structured filesystem bridge at `~/.bragi/agent-control-plane/`. The manager decides what to do. The executor executes. The human copies output between ChatGPT and terminal.

The entire system is one Python script (stdlib only), one directory, four JSON schemas, and a human clipboard.

---

## What Problem It Solves

Without the bridge, manager↔CLI collaboration requires manual directory navigation, filename construction, JSON copying, and housekeeping. Every round-trip is error-prone.

The tool reduces the loop to five one-line commands:

```
INSTRUCT  echo '<json>' | bacp-bridge write-instruction
READ      bacp-bridge next-instruction
ACK       bacp-bridge ack <id>
REPORT    echo '<json>' | bacp-bridge write-report
REVIEW    bacp-bridge next-report
```

No directory navigation. No filename construction. No manual archiving.

---

## Commands (13)

| Command | Purpose |
|---------|---------|
| `status [--json]` | Bridge health and queue counts |
| `list-instructions [--json]` | List all pending manager instructions |
| `list-reports [--json]` | List all pending executor reports |
| `next-report [--json]` | Show oldest pending report |
| `next-instruction [--json]` | Show oldest pending instruction |
| `write-instruction [--archive-source <id>]` | Write manager instruction from stdin |
| `write-report [--ack-source <id>]` | Write executor report from stdin |
| `decisions [--json]` | Show pending/answered decisions |
| `archive <id>` | Move message to archive |
| `archive-all <queue> [--older-than <s>]` | Bulk archive with optional age filter |
| `ack <id>` | Archive consumed message |
| `stop` | Halt bridge (confirmation required) |
| `resume` | Reactivate bridge (confirmation required) |

---

## Example Usage

```bash
# Manager instructs executor
echo '{"command":"redirect","instruction":"Add --json flag to status","model_selection":{"manager_model":"sonnet","executor_model":"sonnet","memory_action":"keep","reason":"example"}}' \
  | bacp-bridge write-instruction

# Executor reads, acts, and reports
bacp-bridge next-instruction
bacp-bridge ack <id>
echo '{"project":"bacp","task":{"id":"example","description":"Add --json flag","status":"complete"},"repo":{"branch":"main","last_commit":"abc","working_tree":"clean"},"next_recommended_action":"awaiting review"}' \
  | bacp-bridge write-report --ack-source <id>

# Manager reviews
bacp-bridge next-report
```

---

## What Is Intentionally Not Included

- **No API integration** — the bridge is local filesystem only. ChatGPT writes via human clipboard.
- **No dashboard** — terminal-only. A dashboard reads the same filesystem.
- **No GitHub integration** — the bridge does not push, PR, or review.
- **No automated polling** — the CLI checks for instructions at conversation boundaries.
- **No project discovery** — the tool controls the CLI, it does not explore external projects.
- **No watcher health monitoring** — removed from scope by manager decision.
- **No jsonschema dependency** — validation uses manual checks (sufficient for current use).

---

## Documentation

| Document | Covers |
|----------|--------|
| `docs/manager-control-tool-spec.md` | Full tool specification and security boundaries |
| `docs/manager-control-tool-implementation-plan.md` | Build plan, remaining gaps, acceptance criteria |
| `docs/manager-operating-system.md` | Manager behavior rules, quality gates, decision framework |
| `docs/connector-requirements-from-real-usage.md` | Future connector requirements from real usage |
| `docs/examples/manager-instruction.json` | Example instruction JSON |
| `docs/examples/executor-report.json` | Example report JSON |
| `schemas/manager-control/` | 4 JSON schemas |

---

## Future Roadmap

1. **P0 — Queue inspection** (done): `list-instructions`, `list-reports`
2. **P1 — Poll daemon**: Watch inbox/ for new instructions without manual polling
3. **P2 — Template generator**: Structured report creation with less manual JSON
4. **P3 — Auto-commit**: Optional git commit integration from write-report
5. **Phase 4 — Direct API relay**: Eliminate human clipboard entirely

---

## Commit Structure (23 commits)

**Phase 1 — Bridge Design & Setup** (commits 1–7):
Bridge architecture, manager control loop, directory structure, schemas

**Phase 2 — Implementation** (commits 8–11):
Bridge helper P0 + P1 commands, initial discovery docs

**Phase 3 — Tool Consolidation** (commits 12–15):
Scope refocus, tool spec, consolidation from discovery to tool

**Phase 4 — Complete Loop** (commits 16–19):
write-report, metadata fix, queue inspection, README improvements

**Phase 5 — Operating System** (commits 20–23):
MOS definition, MOS-governed tasks, connector requirements

All commits are signed and on `agent/bootstrap-discovery`.
