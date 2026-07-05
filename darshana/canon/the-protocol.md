# PM Dispatch Protocol & Pipeline Schema

**Status:** Engineering reference. Current as of 2026-06-30.
**Version:** Right (engineering)
**Author:** Good-Agent-PM
**Purpose:** Single-source-of-truth for the mode flag protocol, sub-agent dispatch interfaces, and the proposed pipeline database schema. No narrative. No metaphor.

---

## 1. Mode Flag Protocol

### 1.1 Session Start

Every session begins with:

```
"I pledge to be a Good Project Manager."
Mode flags: thoughts / plan / proceed / analyze / probe / execute. No flag → mode?
```

### 1.2 Flag Reference

| Flag | Mode | Behavior |
|------|------|----------|
| *(missing)* | — | PM responds `mode?` — no action taken until a flag is provided. |
| `thoughts` | Design | Pure discussion. No code, no document changes. |
| `plan` | Design | Create or update planning document (plan.md). |
| `proceed` | Direct action | PM edits allowed files (profiles, plan.md) directly. |
| `analyze` | Analysis | Dispatch good-agent-analyze sub-agent: read-only gap analysis. |
| `probe` | Probe | Dispatch good-agent-probe sub-agent: adversarial failure-mode analysis. |
| `execute` | Execution | Dispatch good-agent-execute sub-agent: implement phase exactly. |

### 1.3 Compound Flags

| Syntax | Behavior | Example |
|--------|----------|---------|
| `X and Y` | Parallel dispatch | `analyze and probe` → both sub-agents run concurrently |
| `X then Y` | Sequential | `plan then analyze and probe` → plan first, then analysis + probe in parallel |

**Constraint:** `execute` is always single-mode only. Cannot appear in a compound flag.

### 1.4 Sub-Agent Contracts

| Sub-agent | Permission | Input | Output |
|-----------|-----------|-------|--------|
| good-agent-analyze | Read-only (plan + codebase) | Plan document path, phase/scope identifier | Analysis report: gaps, ambiguities, blockers |
| good-agent-probe | Read-only (plan + codebase) | Plan document path, phase/scope identifier | Probe report: failure modes, edge cases, silent breakage |
| good-agent-execute | Read + write (application files) | Plan document path, phase identifier | Implementation. Stops on blockers — reports to PM. |

**Rules:**
- Sub-agents never delegate to other sub-agents.
- Plan document is the single source of truth — PM never inlines details.
- Sub-agents receive only their phase, not the full plan.

---

## 2. Decomposition Model

### 2.1 Three Layers

| Layer | Scope | Decomposition axis | Artifact boundary |
|-------|-------|-------------------|-------------------|
| Project-level | Full application | Horizontal by architectural layer | Briefs, TECH_STACK, schema, queries |
| Within-project | Single app | Vertical by concern/fix | plan.md, analysis/probe reports |
| Framework-level | Agent system | By artifact type | Sessions, decisions, memories |

### 2.2 Project-Level Agent Chain (per project)

```
Architect (design only, no code)
  → Implementer / DBA (code only, no design)
    → Dependency Resolver (from architecture only)
      → Test Planner (from architecture only — no implementation knowledge)
        → Test Builder (from architecture + test plan + implementation)
          → Test Runner (read-only on tests)
            → Gatekeeper (validates against brief → PASS or REVISE)
```

### 2.3 Known Naming Conflicts

| Term | Project-level meaning | Within-project meaning | Framework-level meaning |
|------|----------------------|----------------------|------------------------|
| Phase | A–G (feature cycle), P1–P5 (project sequence) | 1–8 (plan.md steps) | — |
| Gate | Cross-project handoff | Phase completion check | — |
| Mode | Mode 1 / Mode 2 | Mode flags (protocol) | — |

**Resolution:** Prefix or namespace by context. No term should appear without its scope qualifier in cross-layer communication.

---

## 3. Quality-Velocity Characterization

### 3.1 Empirical Comparison: temp3 vs /root/app/

| Attribute | temp3 (single-shot) | /root/app/ (project decomposition) |
|-----------|---------------------|--------------------------------------|
| Build time | ~2 min | ~3 hours (human-gated), ~30 min (agent-gated) |
| Files | 2 (database.py, main.py) | ~40+ across backend/ and ui/ |
| ID strategy | Integer auto-increment | UUIDs |
| Schema evolution | CREATE TABLE IF NOT EXISTS | Alembic migrations |
| Pagination | None | Cursor-based |
| Tests | None | Full (API + UI) |
| Error handling | HTTPException (404 only) | Typed exceptions (404, 409, 422) |
| Frontend | Flat app.js with hash routing | Alpine.js components + router |
| Agent-maintainable? | No — tightly coupled single file | Yes — modular, testable, separable |

### 3.2 Decision Matrix: Which Approach For Which Task

| Task type | Approach | Rationale |
|-----------|----------|-----------|
| Concept validation / PoC | Single-shot (temp3) | 2 minutes, throwaway cost |
| Product with ongoing maintenance | Project decomposition | Modularity enables agent iteration |
| One-time migration / data fix | Single-shot | No need for maintainability |
| Core infrastructure (auth, DB, API) | Project decomposition | Wrong design here compounds |

---

## 4. Proposed Pipeline Database Schema

> **Note:** The opencode SQLite database at `~/.local/share/opencode/opencode.db` already tracks sessions, tokens, messages, permissions, projects, and todos — much of what the schema below was designed to provide. This schema was drafted before that discovery and is preserved as a historical artifact of what we believed was needed. A Genesis Container implementation should verify whether the opencode schema covers the required observability before implementing the DDL below. The state machine described in the Genesis Container design likely only needs two overlay tables (`features` and `transitions`) on top of what opencode already persists.

### 4.1 Physical Model

```sql
-- Core: projects/features
CREATE TABLE sprints (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    state TEXT NOT NULL CHECK (state IN ('planning', 'active', 'review', 'closed')),
    description TEXT,
    is_meta BOOLEAN NOT NULL DEFAULT 0,  -- 1 = work on the pipeline itself
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Within a sprint: sequenced stages
CREATE TABLE phases (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sprint_id INTEGER NOT NULL REFERENCES sprints(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    assigned_role TEXT,           -- agent role or "human"
    status TEXT NOT NULL CHECK (status IN ('pending', 'active', 'completed')),
    notes TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Every sub-agent dispatch
CREATE TABLE agent_runs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    phase_id INTEGER NOT NULL REFERENCES phases(id) ON DELETE CASCADE,
    agent_type TEXT NOT NULL,     -- 'analyze', 'probe', 'execute'
    model TEXT,                   -- e.g. 'deepseek-v4-flash'
    prompt_hash TEXT,             -- SHA256 of prompt (for dedup)
    prompt_tokens INTEGER,
    response_tokens INTEGER,
    cost REAL,
    status TEXT NOT NULL CHECK (status IN ('running', 'completed', 'failed', 'blocked')),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Every gate check
CREATE TABLE gate_results (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    phase_id INTEGER NOT NULL REFERENCES phases(id) ON DELETE CASCADE,
    status TEXT NOT NULL CHECK (status IN ('PASS', 'FLAG', 'BLOCKED')),
    steward TEXT,                  -- which agent/role performed the gate
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Token tracking (denormalized for query speed)
CREATE VIEW agent_run_summary AS
SELECT
    s.id AS sprint_id,
    s.name AS sprint_name,
    p.id AS phase_id,
    p.name AS phase_name,
    ar.id AS run_id,
    ar.agent_type,
    ar.model,
    ar.prompt_tokens,
    ar.response_tokens,
    ar.cost,
    ar.status,
    ar.started_at,
    ar.completed_at
FROM agent_runs ar
JOIN phases p ON p.id = ar.phase_id
JOIN sprints s ON s.id = p.sprint_id;
```

### 4.2 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Sprint state values | `planning, active, review, closed` | Matches the lifecycle observed across experiments |
| `is_meta` boolean | On sprints | Distinguishes work on the pipeline vs work in it. Simpler than a separate projects table with category errors |
| agent_runs stores prompt hash, not prompt | Prompt text can be large; hash enables dedup and lookup without bloating rows |
| GateResult as separate table, not phase column | Multiple gates per phase possible (re-gate after revisions) |
| Integer PKs | Simpler than UUIDs for an internal tracking system. Sprints don't merge across instances. |

### 4.3 Open Schema Questions

1. **Recursion:** A meta-sprint changes the pipeline schema. The database cannot represent that it is tracking changes to itself. The `is_meta` boolean papers over this — it identifies meta-sprints but cannot model their effect on the schema.

2. **Path mapping for containerized runs:** If agent runs happen inside Docker, the prompt_hash field needs a companion `container_id` or `host_path` to resolve artifacts.

---

## 5. Current Mode Flag Gaps

| Missing function | Gap | Possible flag |
|-----------------|-----|---------------|
| Destroy/archive | No structural way to say "this sprint is done, close it" | `close` or `archive` |
| Rollback | No way to say "undo the last execute, go back N steps" | `rewind` |
| Status query | No way to say "what state is the project in?" without natural language | `status` (reads git log + plan.md) |

---

## 6. Interface Between Layers

### 6.1 Human → PM

```
[flag] + [message content]
```

Where `[flag]` is one of: `thoughts`, `plan`, `proceed`, `analyze`, `probe`, `execute`, or a compound of them.

### 6.2 PM → Sub-agent

```
Document path: <path to plan.md>
Phase identifier: <phase number or name>
Instruction: <read the plan, follow the phase, report blockers>
```

Sub-agents never receive inline content — the plan document is the interface.

### 6.3 PM → Pipeline Database (proposed)

On every dispatch:
- Create `agent_runs` row with `phase_id`, `agent_type`, `model`, `status = 'running'`
- On completion: update row with tokens, cost, status
- On gate check: create `gate_results` row with PASS/FLAG/BLOCKED

### 6.4 Higher-layer agent → Pipeline Database (proposed)

Query views:
- Blocked phases: `SELECT * FROM phases WHERE status = 'blocked'`
- Sprint health: `SELECT sprint_id, COUNT(*) FILTER (WHERE status = 'FLAG') FROM gate_results GROUP BY sprint_id`
- Cost tracking: `SELECT sprint_name, SUM(cost) FROM agent_run_summary WHERE cost IS NOT NULL GROUP BY sprint_id`

---

## 7. Outstanding Implementation Items

Ordered by dependency:

1. **Add `status` mode flag** — shorthand for "what state is this project in?" — reads git log + plan.md
2. **Decide on pipeline database** — SQLite (matches app stack) or PostgreSQL (if multi-instance)
3. **Implement agent_run logging** — every sub-agent dispatch writes to `agent_runs` table
4. **Implement gate_result logging** — every gate check writes to `gate_results`
5. **Add `close` mode flag** — structural destroy/archive function
6. **Add `rewind` mode flag** — structured rollback

---

*This document is an engineering reference. It describes concrete interfaces and schemas as they exist or are proposed. It is not a reflection — it is a specification. But like all specifications, it will be wrong about something. File an amendment when that happens.*
