## Problem

The old flow relied on one update LLM call to decide add/update/delete/skip outcomes directly. In contradiction scenarios, this could silently overwrite or delete memories with limited operator control.

## Solution

### 1. Two-stage conflict pipeline

For each retrieved memory pair:

1. Run a dedicated classification LLM call that returns:
- `conflict_class`: `CONTRADICTION | NUANCE | UPDATE | NONE`
- explanation
- `proposed_action`
- `confidence_new` / `confidence_old`

2. Route by class:
- `CONTRADICTION`: send to resolution handler (auto strategy or HITL)
- `NUANCE`: no contradiction resolution action; incoming fact bypasses contradiction handler
- `UPDATE`: bypasses contradiction handler
- `NONE`: bypasses contradiction handler

### 2. Resolution strategies

User-configurable `auto_resolve_strategy` options:

- `keep-newer` -> `KEEP_NEW`: delete old memory, add incoming memory
- `keep-higher-confidence` -> `KEEP_NEW`/`KEEP_OLD` from classifier confidence scores
- `delete-old` -> `DELETE_OLD`: delete old memory, do not add incoming memory
- `merge` -> `MERGE`: merge LLM rewrites a unified memory, then old is deleted and merged memory is added
- `follow-llm` -> resolve directly from classifier `proposed_action` (`KEEP_NEW`, `KEEP_OLD`, `DELETE_OLD`, `MERGE`)

### 3. Updated HITL UX and parsing behavior

When `hitl_enabled=True` and a contradiction is detected, terminal output now follows this format:

```text
┌─ Contradiction detected ──────────────────────────────┐
│ Existing:  ...
│ Incoming:  ...
│
│ <classifier explanation>
│
│ LLM proposed: <KEEP_NEW|KEEP_OLD|DELETE_OLD|MERGE>
└────────────────────────────────────────────────────────┘

  [y]  accept proposed → <resolved action>
  [1]  <alternative action #1>
  [2]  <alternative action #2>
  [3]  <alternative action #3>

To apply a strategy to all future conflicts this session, append:
  always:keep-new | always:keep-old | always:delete-old | always:merge | always:follow-llm

Examples: "y"  "2"  "y always:follow-llm"  "1 always:keep-old"
```

Key behavior details:
- `y` accepts the normalized proposed action.
- `1/2/3` select from the three non-proposed alternatives (dynamic order).
- Optional suffix persists session override:
  - `always:keep-new`
  - `always:keep-old`
  - `always:delete-old`
  - `always:merge`
  - `always:follow-llm`
- If a session override exists, prompts are skipped for that session.
- `always:follow-llm` re-evaluates each contradiction from current `proposed_action`.
- Invalid input is reprompted once, then defaults to `KEEP_OLD`.

### 4. Main pipeline behavior (sync + async)

Both `Memory._add_to_vector_store` and `AsyncMemory._add_to_vector_store` now:

- classify each retrieved pair using `get_conflict_classification_messages(...)`
- invoke `hitl_prompt_sync/async` when HITL is enabled; otherwise call `apply_auto_resolution(...)`
- execute resolution mutations:
  - `KEEP_NEW`: delete old, add incoming
  - `KEEP_OLD`: no mutation
  - `MERGE`: merge LLM call, delete old, add merged
  - `DELETE_OLD`: delete old, do not add incoming
- directly create non-contradiction facts
- intentionally bypass the previous single-pass update-memory LLM stage

## Scope

### Core files

- `mem0/configs/base.py`
- `mem0/configs/prompts.py`
- `mem0/memory/conflict.py`
- `mem0/memory/main.py`

### Tests and manual coverage

- `tests/memory/test_conflict.py`
- `tests/memory/test_conflict_sqlite.py`
- `tests/memory/test_main.py`
- `tests/memory/manual/test_conflict_real.py`
- `tests/memory/manual/test_conflict_e2e.py`

## Backward Compatibility

- `Memory.add()` and `AsyncMemory.add()` response shapes remain unchanged.
- Behavior changes are focused on contradiction internals and strategy/HITL configuration paths.

## Risks / Notes

- `follow-llm` depends on classifier `proposed_action` quality.
- `MERGE` introduces an extra LLM call for contradiction pairs resolved as merge.
- `always:follow-llm` intentionally allows per-conflict action variation while skipping future prompts.
