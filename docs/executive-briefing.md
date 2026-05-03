# AgentCP — Executive Briefing

**Date:** 2026-05-03  
**Owner:** Nikolaj Hviid  
**Status:** Transition from Local CLI System → Distributed Control Plane

---

## 1. Executive Summary

AgentCP is evolving from a local CLI-based manager/executor system into a distributed AI control plane.

The system's purpose is to:
- Replace repetitive developer supervision
- Enable structured, autonomous execution
- Maintain strict quality and control through a manager layer
- Scale across multiple projects with clear governance

**Core Idea:** A Manager (AI) controls Executors (CLI agents) through a structured system with visibility, auditability, and guardrails.

---

## 2. What Has Already Been Built

### 2.1 Local Control System (Working Today)

A fully functional system exists.

**Components:**
- CLI tool: `scripts/bacp-bridge` (14 commands)
- Filesystem message bus at `~/.bragi/agent-control-plane/`
- Manager Operating System (MOS) — task decomposition, quality gates, failure handling
- Autocontrol relay (`--once`, `--watch`)

**Capabilities:**
- Full manager→CLI→manager control loop
- Zero manual file handling required
- Audit logging and structured JSON communication
- 24/7 metadata fallback, queue inspection, bulk archive with age filtering
- Kill switch (STOP file)

**Status:** System works in real usage. Multiple full cycles executed. Stable and predictable.

### 2.2 Manager Operating System (MOS)

Defines how the manager behaves.

**Core Principles:**
- Manager owns direction, quality, verification
- Executor owns bounded execution only
- One task = one objective
- No scope expansion by executor
- Every task must have objective, constraints, success criteria

**Failure Handling:** 1st failure → retry. 2nd → redirect. 3rd → stop and analyze.

### 2.3 Autocontrol

Two modes:
- `--once`: Single-pass relay. Detects report, prints manager prompt, reads instruction, writes to inbox.
- `--watch`: Continuous poll loop. Same relay, repeated. Tracks processed reports. Checks STOP before every cycle.

**Result:** Full loop working. No manual queue interaction. Still requires human in the middle (stdin / copy-paste for the instruction itself).

---

## 3. Current Limitation (Critical)

The system is functionally complete but not scalable.

**Problems:**
1. No direct ChatGPT ↔ CLI connection (human clipboard still required)
2. No UI or visibility into system state
3. No multi-project orchestration
4. Filesystem bridge is local only — cannot scale across machines or teams
5. No persistent shared state beyond flat files
6. No real-time monitoring
7. Human still required for transport

---

## 4. Strategic Pivot

**Rejected:** Filesystem-based connector (Phase 3 poll daemon).

**New Architecture:** Build a true control plane using:
- **Supabase** — state + messaging
- **Railway** — backend + UI
- **Custom GPT** — manager interface
- **CLI workers** — execution layer

---

## 5. Target Architecture

### 5.1 Core Components

**Supabase (Source of Truth)**
Stores: projects, agents, sessions, messages, instructions, reports, decisions, audit logs, terminal events, model usage.

**Railway (Execution Layer)**
Hosts: API service, web dashboard, worker orchestration.

**Custom GPT (Manager)**
Operates from ChatGPT: reads reports, issues instructions, approves decisions, controls execution, routes models.

**CLI Workers (Executors)**
Run locally: execute tasks, stream terminal output, submit reports, respect STOP and human gates.

### 5.2 Three Core Roles

| Role | Responsibilities |
|------|-----------------|
| **Manager** | Decision-making, task definition, quality control, model selection |
| **Executor** | Implementation, testing, reporting |
| **Terminal** | Command execution, logs, git state, test output |

### 5.3 UI Model

Each project should display: Manager panel, Executor panel, Terminal panel.

Dashboard: project status, pending reports, decisions, releases, documentation health.

---

## 6. Cross-Project System

Support: project-to-project communication, dependency tracking, collaboration requests, contract enforcement.

---

## 7. Human Gate System

Actions requiring approval: merge, release, deploy, security changes, rollback, cross-project conflicts, high-cost model usage.

---

## 8. Model Routing Strategy

| Model | Role |
|-------|------|
| Haiku | Classification / routing |
| Sonnet | Default execution |
| Opus | Deep reasoning (restricted) |

---

## 9. Migration Strategy

| Phase | What | Status |
|-------|------|--------|
| Current | Filesystem bridge (working) | ✅ Live |
| Future | Supabase-backed control plane | 🔜 Design |

Keep CLI tool as fallback. Move production to Supabase. Add API layer, GPT control, UI.

---

## 10. Implementation Phases

| Phase | Description |
|-------|-------------|
| 0 | Architecture (this document) |
| 1 | Supabase schema |
| 2 | Railway API |
| 3 | Custom GPT integration |
| 4 | CLI worker integration |
| 5 | Web UI |
| 6 | Cross-project orchestration |

---

## 11. Key Insight

> This is no longer a tool. This is an AI operating system for software execution.

---

## 12. Final State Vision

A system where you open ChatGPT, see all projects, issue commands, the system executes autonomously, and you only intervene for high-level decisions.

---

## 13. One-Line Summary

> AgentCP becomes a central nervous system for AI-driven development across projects, with full control, visibility, and safety.
