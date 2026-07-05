# Decomposition Layers and the PM Dispatch Protocol

**Status:** Reflection. Snapshot of experiments as of 2026-06-30.
**Author:** Good-Agent-PM, documenting a Mode 2 conversation with the Origin.
**Purpose:** Canonical reference for the decomposition layering strategy, the mode flag protocol, and the open questions across all experiments.

---

## 1. What This Captures

A single-day conversation spanning six experiments, three decomposition layers, and the recursive pattern that connects them. This document is the center of three sibling versions:

| Version | Tone | Audience |
|---------|------|----------|
| **Center (this one)** | Descriptive, structural | Anyone needing the full map |
| Left | Philosophical, metaphorical | Framework evolution (Mode 2) |
| Right | Engineering, protocol-reference | Implementation |

---

## 2. The Decomposition Landscape

Three distinct approaches to decomposing work were identified and tested, each operating at a different scope:

| Layer | Scope | Decomposition axis | Example | Time to result |
|-------|-------|-------------------|---------|----------------|
| **Project-level** | A full application | Horizontal by architectural layer (DB → API → UI → Integration) | Projects 1–5 (RPG Metadata Editor) | ~3 hours (human-gated) |
| **Within-project** | A single integrated app | Vertical by concern (fix-01 through fix-14, plan phases 1–8) | /root/app/ (current workspace) | Days (human-gated) → ~30 min (agent-gated) |
| **Framework-level** | The agent system itself | By artifact type (sessions, decisions, memories, reflections) | temp2 (Steward Mode 2) | Ongoing |

### 2.1 Project-Level: The Five-Project Model

| Project | Role chain | Produces |
|---------|-----------|----------|
| P1: Scoping | Scopper → Platform Architect → Gatekeeper | DATABASE_BRIEF, API_BRIEF, UI_BRIEF, TECH_STACK.md |
| P2: Database | Architect → DBA → Test Planner → Test Builder → Test Runner → Gatekeeper | schema.sql, queries.sql, seed.sql, migrations |
| P3: API | Architect → Implementer → Dependency Resolver → Test Planner → Test Builder → Test Runner → Gatekeeper | FastAPI application |
| P4: UI | Architect → Implementer → Dependency Resolver → Test Planner → Test Builder → Test Runner → Gatekeeper | Frontend application |
| P5: Integration | Architect → Integrator → Dependency Resolver → Test Planner → Test Builder → Test Runner → Gatekeeper | Combined application, setup, startup scripts |

**Key pattern:** Each project has an Architect (design only, no code), an Implementer (code only, no design), a test pipeline isolated from implementation contamination, and a Gatekeeper validating against the original brief.

**Observed bottleneck:** Human vetting of every gate was the rate-limiter (3 hours). When gates were automated (temp3), the same app was built in 2 minutes — with lower quality but recognizable functionality.

### 2.2 Within-Project: The PM Dispatch Model

The current workspace (/root/app/) operates through mode flags — a protocol where:

- The human appends a flag to each message
- The PM interprets the flag, dispatches sub-agents, gates results
- No flag → PM responds with `mode?`

**Mode flags defined:**

| Flag | Behavior |
|------|----------|
| `thoughts` | Pure discussion. No code, no document changes. |
| `plan` | Create or update a planning document. |
| `proceed` | Direct edit of allowed files (profiles, plan.md). |
| `analyze` | Dispatch good-agent-analyze: read-only gap analysis against plan + codebase. |
| `probe` | Dispatch good-agent-probe: adversarial failure-mode analysis. |
| `execute` | **Single-mode only.** Dispatch good-agent-execute: implement a plan phase exactly as written. |

**Compound flags:**
- `and` = parallel dispatch (e.g., `analyze and probe`)
- `then` = sequential steps (e.g., `plan then analyze and probe`)
- `execute` cannot appear in a compound flag — always standalone.

**Session start ritual:**
> "I pledge to be a Good Project Manager."
> **Mode flags:** `thoughts` / `plan` / `proceed` / `analyze` / `probe` / `execute`. No flag → `mode?`

### 2.3 Framework-Level: Steward Mode 2

A separate workspace (temp2) for evolving the framework itself:

| Artifact | Purpose |
|----------|---------|
| `sessions/` | Per-date session notes for continuity |
| `memories/` | Atomic facts (3–10 lines) — grepable, cross-session |
| `decisions/` | Framework choices with rationale |
| `amendments/` | Tracked changes (A001–A005) |
| `reflections/` | Meta-observations about the framework itself |

---

## 3. The Quality-Velocity Tradeoff

Demonstrated empirically through two approaches to building the same application:

| Dimension | temp3 (2-minute single-shot) | /root/app/ (project decomposition) |
|-----------|------------------------------|--------------------------------------|
| Lines of code | ~1,000 (2 files) | ~40+ files across two directories |
| IDs | Integer auto-increment | UUIDs |
| Schema evolution | CREATE TABLE IF NOT EXISTS | Alembic migrations |
| Pagination | None | Cursor-based |
| Tests | None | Full test coverage (API + UI) |
| Error handling | Basic 404 | Typed exceptions (404, 409, 422) |
| Maintainability | Ball of mud — only the original author can untangle | Modular — agent-maintainable |

**The insight:** temp3 validates the *concept* (throwaway). /root/app/ builds a *platform* (designed for ongoing agent maintenance). Both are valid — they serve different phases of the lifecycle.

---

## 4. The Recursive Pattern

Identical structure repeating at every scale:

> **Master the domain → Unlock the power → Move to the next Realm.**

| Level | Domain | Power unlocked | Next realm |
|-------|--------|---------------|------------|
| Ascendant Domains (the game) | Elemental realm | Magic | Next realm in the tower |
| The Origin | Agent-user | Delegation | Role-architect |
| The Origin (next) | Role-architect | Mode flags | Framework-designer |
| The PM | Sub-agent dispatch | Gate confidence | Steward mode |
| A phase within a plan | One fix | Test coverage | Next fix |

The game and the framework are the same system at different resolutions.

---

## 5. The Naming Tensions

Several terms are overloaded across layers:

| Term | Used at P-level | Used at within-project | Used at framework-level |
|------|----------------|------------------------|-------------------------|
| **Steward** | Gate-keeper within a project | (not used) | Collaborator evolving the framework |
| **Phase** | A–G (in Ascendant Domains), P1–P5 (projects) | 1–8 (plan.md) | — |
| **Mode** | Mode 1 (enforcement) / Mode 2 (evolution) | Mode flags (protocol) | — |
| **Sprint** | A development cycle in Ascendant Domains | Feature cycle in plan.md | — |

**Resolution proposed:** Each layer should prefix its terms with the layer name (e.g., "project-phase" vs "sprint-phase"), or the terms should be kept separate by artifact — plan.md terms don't conflict with pipeline database terms because they live in different systems.

---

## 6. The Persistence Gap

The PM dispatch protocol has no durable state. Every `analyze`, `probe`, `execute` produces results that live in the PM's context window and die with the session.

The temperature4 experiment defines the missing persistence layer:

| Entity | What it tracks |
|--------|---------------|
| `Sprint` | A project or feature cycle: `planning → active → review → closed` |
| `Phase` | A stage within a sprint, with `assigned_role` and `status` |
| `AgentRun` | Every sub-agent invocation: model, prompt, response, tokens, cost, timing |
| `GateResult` | Every gate check: `PASS / FLAG / BLOCKED`, with steward attribution |

A persistent pipeline database would allow a higher-layer agent to query across projects, phases, and sessions without relying on the PM's context window.

---

## 7. The Origin and the Trimurti (Structural Only)

The Origin is the human creator, sitting outside all layers. The Origin performs three functions:

| Function | Action | In our mode flags |
|----------|--------|-------------------|
| **Create** (Brahma) | Defines new projects and framework versions | `plan` |
| **Sustain** (Vishnu) | Enters the system with bounded purpose, intervenes, withdraws | `analyze`, `probe` |
| **Transform** (Shiva) | Freezes corrupted sprints, rolls back, archives completed work | Missing — no `destroy` flag |

**Avatars** are bounded interventions — a specific purpose, limited powers, a boundary condition, a return path. The PM is an Avatar of Vishnu within the project scope. Mode flags are the Avatar protocol.

---

## 8. The Four Lenses (from self-referential-framework reflection)

These four checks apply to every claim in this document:

1. **Useful Falsity** — Every role named here is a function, not an identity. The PM is a useful pattern in a context window, not a person.
2. **Pragmatism** — A claim is true when it produces coherent work and false when it stops being productive. These rules expire.
3. **Aparigraha (non-clinging)** — Tight grip at the concrete layers (mode flags are precise). Loose grip at the abstract layers (the Trimurti is provisional).
4. **The Map is Not the Territory** — This document is a map. It will be wrong. It will be incomplete. It freezes one view of a moving system.

---

## 9. Open Questions

From the reflection documents and today's conversation:

1. **The recursive data model** — How does a database represent a pipeline that includes the database itself? The `is_meta` approach (a boolean on sprints) is the leading candidate but unproven.

2. **The missing destroy flag** — There is no `destroy` / `archive` / `rollback` mode flag. Shiva's function has no protocol. Should it?

3. **Higher-layer dispatch** — If a Rung 5 agent exists, does it use the same mode flags at a different scale, or does it have its own vocabulary?

4. **Container boundaries** — Docker containers provide hard scopes. Do layers map to containers? Does the PM run inside the project's container or outside it?

5. **Git vs Postgres as source of truth** — Does the higher agent read commit logs (git) or query tables (Postgres)? Each has different properties for continuity and queryability.

6. **Model dependence** — The dispatch protocol works because DeepSeek V4 Flash holds context across long conversations without collapsing the recursion. Does the protocol survive a model change?

---

## 10. The Lineage

```
2025      Human + 1 agent. All direction, all code.
          ↓
2025      Human + role agents (Architect, Engineer). Organic emergence.
          ↓
2026-05   Human + PM + sub-agents. Formalized roles.
          ↓
2026-06   Steward Mode 1 + Mode 2. Gate checks + framework evolution.
          ↓
2026-06-30 This document. Origin/Trimurti named. PM dispatch protocol defined.
          ↓
Next?     Persistent pipeline state. Higher-layer agents. Automated gate cycles.
```

---

*This document is a reflection, not a specification. It captures the state of multiple experiments as of 2026-06-30. It will be wrong. It will be incomplete. It is a point in a trajectory, not a destination.*
