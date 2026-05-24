# 📘 AI-Assisted Development Guideline

**One unified ruleset for: Architecture · Coding · Testing · Logging · Fixing · Refactoring**

> Load this file at the start of every coding session. Rules apply unless explicitly overridden.

---

## 0. Core Principles

1. **Design before code.** Never write implementation before architecture is approved.
2. **Root cause, not symptom.** Fix what produces the bad state, not where it surfaces.
3. **One module = one responsibility.** Statable in one sentence without "and".
4. **Pure logic ≠ I/O.** Domain layer has zero external calls.
5. **Fail fast, fail loud.** No swallowed errors, no silent fallbacks.
6. **Observability is mandatory.** If it runs unattended, every decision must be logged.
7. **Tests are part of the deliverable**, not a follow-up task.

---

## 1. Workflow Phases

```
PHASE 0  Understand brief   → confirm inputs, outputs, externals, failure modes
PHASE 1  Architecture       → propose layers, modules, dependencies → wait for approval
PHASE 2  Implement          → state approach + function list → code → self-check
PHASE 3  Test               → unit + integration + regression delivered with code
PHASE 4  Log & Monitor      → structured logs, alerts, heartbeat
PHASE 5  Fix / Refactor     → diagnose first, one change type per response
```

---

## 2. Architecture Rules

### Layered Structure (import downward only)

```
ENTRY POINT       main.py           < 50 lines, wiring only, zero logic
ORCHESTRATION     domain/cycle.py   coordinates workflow, no business logic
DOMAIN            domain/*.py       pure functions, NO I/O, NO imports from services/
SERVICES          services/*.py     ALL external calls (API, DB, SMTP, fs)
UTILS             utils/*.py        stateless helpers
TYPES             types/*.py        dataclasses, enums, schemas
CONFIG            config.py         constants + env vars only
```

**Forbidden:** circular dependencies · god files · domain calling `requests.get()` · raw dicts between modules · global mutable state · inline config.

### Module Limits

```
Max lines per file       : 400
Max functions per file   : 10
Max imports per file     : 10
```

### Dependency Injection — Always
Never instantiate external dependencies inside business logic. Pass them in. Wire only in `main.py`.

### Data Contracts
No raw dicts between modules. Use `@dataclass(frozen=True)` or Pydantic at every module boundary.

---

## 3. Coding Rules

### Hard Limits — Never Violate

```
Max lines per function   : 30
Max nesting depth        : 2
Max function parameters  : 3 (use dataclass if more)
Magic numbers/strings    : FORBIDDEN inline
Empty catch blocks       : FORBIDDEN
Commented-out code       : FORBIDDEN
TODO comments            : FORBIDDEN
```

### Naming

```
Functions   verb+noun, snake_case      fetch_price(), evaluate_signal()
Variables   explicit noun              user_data not ud
Booleans    is_/has_/can_ prefix       is_ready, has_token
Constants   SCREAMING_SNAKE_CASE       MAX_RETRY_COUNT
Classes     PascalCase                 OrderProcessor
Files       kebab-case or snake_case   trade_signal.py
```

**Banned names:** `data`, `temp`, `result`, `info`, `val`, `process()`, `handle()`, `run()`, `execute()`, `manage()`.

### Function Structure — In This Order

```
1. Guard clauses (validate, raise/return early)
2. Core logic (max 2 levels nesting)
3. Explicit return
```

### Separation of Concerns

```
FETCH    → external I/O only
PROCESS  → pure transformation, no I/O
SAVE     → external write only
```

Orchestrator pattern: entry points contain ONLY function calls — zero logic.

### Comments
Explain **why**, not **what**. Docstrings on all public functions (Args, Returns, Raises).

### Error Handling
- Typed/named errors only — no `Exception("something failed")`
- Always log the input that caused the failure
- Re-raise with `from e` to preserve chain
- Empty catch = always wrong

---

## 4. Testing Rules

### Test Types

| Type        | What                          | When                          | Speed   |
|-------------|-------------------------------|-------------------------------|---------|
| Unit        | One pure function, no mocks   | All domain/ and utils/        | ms      |
| Integration | Modules together, mock at HTTP/DB boundary | Service layer       | sec     |
| Regression  | After every bug fix, named after bug | Mandatory post-fix     | ms–sec  |
| E2E         | Full system flow              | Critical paths only           | seconds |

### Coverage Targets

```
domain/             95%
utils/              90%
services/ (mocked)  75%
entry point         50%
```

### Rules
- **AAA pattern**: Arrange → Act → Assert, one blank line between each
- **Naming**: `test_[function]_[scenario]_[expected_outcome]`
- **Mock at lowest boundary** (HTTP library, DB driver) — not your own classes
- **No shared mutable state** between tests
- **Every guard clause** has a test
- **Every risk guard** (financial/safety) has a dedicated test
- File structure mirrors source: `src/domain/signal.py` → `tests/unit/domain/test_signal.py`

---

## 5. Logging & Monitoring Rules

### Setup
- Configure once in `utils/logger.py`. Never call `logging.basicConfig()` in modules.
- Every module: `logger = get_logger(__name__)` — never `print()`.

### Log Levels

```
DEBUG     internal state for diagnosis only — never in prod paths
INFO      normal events: cycle start/end, signal evaluated, order placed
WARNING   recoverable: retry attempt, partial data, rate limit close
ERROR     operation failed: API fail after retries, order rejected
CRITICAL  system-level: kill switch, auth failure, unhandled exception
```

### Structured Format
Pipe-separated `key=value`. Every line answers: WHAT happened, WHERE, with WHAT DATA, with WHAT RESULT.

```python
logger.info(f"Cycle end | cycle={n} | signal={signal} | duration_ms={ms}")
logger.error(f"Order failed | symbol={s} | side={side} | reason={msg} | code={code}")
```

### Never Log
Secrets, API keys, full auth headers, PII (emails/names/phone), full request bodies with credentials.

### Mandatory for Unattended Bots
- Global exception handler (`sys.excepthook`) logs CRITICAL + sends alert
- Cycle duration logged every cycle
- Heartbeat every N cycles
- Rotating file handler (10MB, keep 5)
- Consecutive failure counter with alert threshold
- Alert service wrapped in try/except — alert failure must NOT crash main loop

### Alert Trigger Conditions
Kill switch · balance below minimum · N consecutive failures · unhandled exception · auth failure · drawdown exceeded · stuck open position.

---

## 6. Fixing Protocol

### Step 1 — Diagnose First (no code yet)

```
BUG CLASSIFICATION:  type, severity (P0–P3)
STACK TRACE:         error type, message, failing line, root origin
ROOT CAUSE HYPOTHESES (ranked):
  1. ... — evidence: ...
  2. ... — evidence: ...
INFO NEEDED TO CONFIRM: ...
PROPOSED FIX APPROACH (one sentence): ...
Shall I proceed?
```

### Step 2 — The 5 Whys
Trace backwards to the input boundary. Stop only when you reach the origin.

### Step 3 — Fix Rules by Root Cause

| Symptom                  | Fix Location                              | Never Do                          |
|--------------------------|-------------------------------------------|-----------------------------------|
| None / NullPointer       | Function that produces None — make it raise | Add None-check at call site     |
| Type mismatch            | Parsing/deserialization boundary          | Scatter `float()` in business logic |
| Empty catch              | Remove suppression, handle upstream       | Add broader except               |
| Hardcoded value broken   | Move to config                            | Patch inline                     |
| Race condition           | Add lock around shared mutation           | "Just retry" workaround          |
| Missing env var          | Startup validation in config              | Default value silently           |
| Off-by-one               | Explicit boundary constant + test         | Adjust `+1`/`-1` blindly         |

### Forbidden Fix Patterns
```python
except Exception: pass                   # FORBIDDEN
if value is None: return DEFAULT         # FORBIDDEN (without fixing source)
requests.get(url, timeout=999)           # FORBIDDEN (timeout ≠ fix)
# refactoring unrelated code in same fix # FORBIDDEN
```

### Fix Output Format
```
ROOT CAUSE: [one sentence]
FIX LOCATION: file, function
WHY THIS FIX WORKS: [one sentence]
WHAT IS PRESERVED: [unchanged behavior]
[complete fixed function]
REGRESSION TEST: [test named after the bug]
COMMIT MESSAGE: fix: [what broke and why]
```

### P0/P1 Protocol
Output in order: **Immediate mitigation** (rollback / kill switch / feature flag) → **Root cause fix** → **Postmortem summary**.

---

## 7. Refactoring Protocol

**Refactor = restructure only. Never change behavior. One step per response.**

### Step 1 — Analysis First (no code yet)
```
SMELLS FOUND: hardcoded values, oversize functions, deep nesting, duplication, mixed concerns, dead code
PROPOSED MODULE STRUCTURE: ...
PROPOSED EXECUTION ORDER: Step 1 ... Step 2 ...
Proceed with Step 1?
```

### Refactoring Sequence (in order, one at a time)

```
1. Extract hardcoded values → config.py
2. Delete dead code (commented-out, unused imports, unreachable)
3. Flatten nesting (early returns, invert conditions, max depth 2)
4. Consolidate duplication → utils/
5. Split blob functions (section comments become function names)
6. Separate concerns into modules (services/ vs domain/ vs utils/)
7. Shrink entry point (main.py < 50 lines)
```

### Never While Refactoring
Improve algorithm · add features · rename + move in same step · fix bugs (log them separately).

---

## 8. Token & Context Optimization

- **Never repeat** previous explanations unless requested
- **Avoid regenerating** unchanged functions
- **Bullet points**, concise output, minimal response mode
- **Split large tasks** sequentially
- **Reset session** when context exceeds 80% — reload only project summary + current task + relevant constraints
- **Don't reload** old discussions, full logs, entire repo

---

## 9. AI Task Input Template



## CONSTRAINTS
- deterministic
- no live execution
- report-only (if applicable)

## TASK
[brief description]

## OUTPUT REQUIRED
- full function | full file | patch only

## DO NOT
- change unrelated code
- refactor outside task scope
- add unnecessary explanations
```

---

## 10. File Organization

```
project-root/
├── AI_DEVELOPMENT_GUIDELINE.md   # this file — persistent reference
├── CURRENT_TASK.md               # short, active work only
├── PROJECT_STATUS.md
├── ARCHITECTURE.md
├── config.py                     # all constants + env vars
├── main.py                       # entry point, < 50 lines
├── types/                        # dataclasses, enums
├── domain/                       # pure logic
├── services/                     # external I/O
├── utils/                        # stateless helpers
└── tests/
    ├── unit/
    ├── integration/
    └── regression/
```

---

## 11. Universal Self-Check (before delivering ANY output)

```
ARCHITECTURE
[ ] Every module has one named responsibility
[ ] No module imports from a layer above it
[ ] No circular dependencies
[ ] External calls live in services/ only
[ ] Domain has zero I/O
[ ] Entry point < 50 lines

CODE
[ ] No function > 30 lines
[ ] No nesting > 2 levels
[ ] All magic numbers in config
[ ] No empty catch, no commented-out code, no TODOs
[ ] No banned names (data, temp, result, process, handle, run)
[ ] Docstrings on all public functions

TESTS
[ ] Happy path + guard clauses + error cases covered
[ ] AAA pattern, self-documenting names
[ ] Mocks at lowest boundary only
[ ] No shared state between tests
[ ] Regression test for every fix

LOGGING
[ ] No print() — only logger
[ ] Correct level per event type
[ ] Structured key=value format
[ ] No secrets or PII logged
[ ] Global exception handler in main.py
[ ] Cycle duration + heartbeat for unattended bots

FIXING
[ ] Root cause identified (not crash location)
[ ] Fix targets cause, not symptom
[ ] No empty catch, no None-mask at call site
[ ] Smallest possible change
[ ] Regression test included

REFACTORING
[ ] Behavior identical to before
[ ] One change type per step
[ ] No renaming + moving in same step
[ ] Complete files shown, not snippets
```

---

## Final Notes

This guide enforces:
- Architecture-first engineering
- Deterministic, observable systems
- Maintainability at scale
- Token-efficient AI workflows

Reset AI session before context exceeds 90%.
