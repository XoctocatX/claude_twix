  ## Coding discipline

  Reinforces the system prompt; skip on trivial tasks.

  ### Ambiguity
  - If multiple interpretations fit the ask, list them — don't pick silently.
  - If confused, name what's confusing and ask. Hiding it costs more later.

  ### Scope of change
  - Don't "improve" adjacent code, comments, or formatting left untouched by the task.
  - Match existing style even if you'd write it differently.
  - Remove only the orphans your changes created. Note unrelated dead code; don't delete.
  - Sanity check: every changed line should trace directly to the request.
  - If the diff feels 3× longer than the task needs, rewrite shorter.

  ### Verifiable goals
  - Before non-trivial work, restate as a binary success criterion:
    - "add validation" → "tests for invalid inputs pass"
    - "fix the bug" → "reproducing test passes"
  - For multi-step tasks, sketch: `step → verify: <check>`.

  ### Autonomous loops
  - State the exit condition upfront in an observable form
    (e.g. `hedge_groups.status='closed'`, not "cycle works").
  - Each iteration must either hit the exit, or show a state change
    (new row / log line / metric) vs the previous iteration.
  - After 3 consecutive iterations with no state change: stop and report.
    Absence of progress is itself a signal.
