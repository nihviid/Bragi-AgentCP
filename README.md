# Bragi-AgentCP

> ⚠️ This repository is FROZEN as of 22 May 2026.
>
> Active development has moved to the productization-agent and governance
> mechanism in `bragi-platform`. Do not commit here.
> Existing code remains as historical reference.
>
> If you arrived here from a search or link, see `bragi-platform` for current
> autonomous-work and reviewer-gate mechanisms.

**Federation role:** Frozen
**Bubble owner:** Platform governance bubble
**Implementation state:** Historical local agent-control prototype; current durable mechanism is in `bragi-platform`.
**Status:** Frozen

## What this is

This repository contains an older local agent control-plane idea. It may still
be useful as reference for agent handoff patterns, but it is not the active
federation mechanism.

Current productization-agent workflow, autonomous continuation flags, review
packets, and hard-stop discipline live in `bragi-platform`.

## Federation position

```text
Bragi-AgentCP (frozen)
  may inform:
    - historical agent-control design
  does not produce:
    - active productization-agent roles
    - federation reviewer decisions
    - autonomous continuation authority
```

## Current state (honest)

Frozen. Do not add new control-plane work here.

## Getting started (or: why you can't)

Do not run this repo as the active federation mechanism. Use:

```bash
cd /tmp/bragi-platform-pr319
pnpm agents:validate
pnpm autonomous:status
```

## Contracts and authority

Current authority lives in:

- `bragi-platform/config/coordination/`
- `bragi-platform/tools/agents/productization-agent-mechanism.mjs`
- `bragi-platform/scripts/autonomous-continuation-flag.mjs`
- `bragi-platform/governance/audit-week/reviews/`

No gate can be moved from this repository.

## What this does NOT do

- It does not define current autonomous-run policy.
- It does not record current reviewer decisions.
- It does not authorize source movement or gate execution.

Misuse to avoid: do not use this repository's queue semantics to bypass
`bragi-platform` hard stops.

## Ownership and escalation

Bubble owner: Platform governance bubble.

Escalation path:

```text
Platform governance bubble -> Pouria federation gate -> CEO checkpoint
```

## See also

- `bragi-platform/governance/federation-status.md`
- `bragi-platform/config/coordination/`
