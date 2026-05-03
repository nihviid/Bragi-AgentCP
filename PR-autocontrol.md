# Add minimum safe autocontrol relay

**Branch:** `agent/autocontrol-minimum` → `main`  
**4 commits** | **3 files changed** (+527/-7)

---

## What Autocontrol Means

Autocontrol is automated transport between the AgentCP filesystem bridge and the ChatGPT manager. It replaces the human clipboard as the message carrier while keeping the human as the decision authority.

**Executor writes a report** → autocontrol detects it → **prints a formatted manager prompt** → manager writes an instruction → autocontrol writes it to inbox → **executor reads it** → repeat.

## What Autocontrol Does NOT Mean

- No autonomous merge or deployment
- No autonomous production changes
- No bypassing the STOP file
- No bypassing human approval for high-risk decisions
- No hidden execution
- No API calls from the relay
- No decision-making — it is transport only

## Commands Added

| Command | Purpose |
|---------|---------|
| `bacp-bridge autocontrol` | Single-pass relay: reads oldest report, prints manager prompt, accepts instruction from stdin, validates and writes to inbox |
| `bacp-bridge autocontrol --watch` | Continuous poll mode: checks outbox/ every N seconds (default: 5), processes reports as they appear |
| `--interval <seconds>` | Override poll interval in watch mode |
| `--archive-report` | Archive the source report after instruction is successfully written |

## Behavior Details

**Single-pass (`--once`, which is the default):**
1. Check STOP → refuse if present
2. Find oldest pending report in `outbox/cli-to-manager/`
3. Print styled manager prompt with full report JSON
4. Read manager instruction from stdin until EOF
5. Validate using existing write-instruction validation
6. Write instruction to `inbox/manager-to-cli/`
7. Optionally archive source report (`--archive-report`)
8. Log all steps to audit log

**Watch mode (`--watch`):**
1. Same as single-pass, but loops
2. Tracks processed filenames in memory — never re-processes
3. Prints "Waiting for reports..." status during idle
4. Checks STOP before every cycle
5. Exits cleanly on Ctrl+C

## Safety Boundaries

- STOP file halts both `--once` and `--watch` before any action
- Never auto-generates instructions — always waits for human input on stdin
- Never calls APIs
- Never runs executor commands
- Never merges, deploys, or modifies files outside the bridge
- Uses existing validation (same as write-instruction)
- All actions logged to bridge.log

## Human Gates Preserved

Autocontrol does not bypass any existing human decision gates. The human still:
- Writes every instruction (autocontrol prints the prompt, human writes the response)
- Decides whether to archive the source report
- Controls the kill switch (STOP file)
- Approves merge, deploy, security, and rollback decisions

## Files Changed

| File | Change |
|------|--------|
| `docs/autocontrol-design.md` | +358 lines — Full autocontrol design specification (3 modes, safety, phases) |
| `scripts/bacp-bridge` | +174/-7 — Added cmd_autocontrol, cmd_autocontrol_watch, main routing, boolean consume_flag support |
| `README.md` | +2 lines — One sentence documenting autocontrol |

## Smoke Test Summary

- `autocontrol` with no reports: "No pending reports."
- `autocontrol` with report + valid instruction: relayed to inbox correctly
- `autocontrol --archive-report`: report archived after write
- `autocontrol` with invalid JSON: rejected with parse error
- `autocontrol --watch` with report: detected, relayed, tracked as processed
- STOP blocks `--once`: "Bridge is HALTED."
- STOP blocks `--watch`: halts mid-cycle
- Existing `write-instruction` and all other commands: unchanged

## Known Limitations

- Processed-filename tracking is in-memory only (lost on restart)
- No clipboard integration (manual copy/paste still needed terminal→ChatGPT)
- No API relay (Phase 4)
- Watch mode polls on timer, not event-driven (adequate for current volume)
- `--once` reads from stdin (blocks if piped without input)
- No deduplication across restarts

## Recommended Next Phase

**Phase 3 — Clipboard helper** (optional): auto-copy report to clipboard.

**Phase 4 — API relay**: connect to model API for direct manager→relay communication without human intermediary.

But first: merge this branch and start using autocontrol for real work.
