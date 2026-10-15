# ADR-0020: Task file scope is detect-and-review

- **Status:** Accepted (direction); details belong to the Implementation design
- **Date:** 2026-10-15

## Context

A plan lists `files_to_modify` and `files_to_create` per task. Plans are written without running any code, so real changes routinely need files the planner could not foresee: barrel or index files, fixtures, snapshots, generated code, and callers of a changed signature. If edit tools refuse every path outside the declared lists, an agent fails, stops, or contorts its solution, and the failure is not a capability problem, so a stronger model does not help. Each such failure becomes a human takeover, which undermines cheap executors.

The lists still have value. They keep blast radius small, and they make the plan-time check that parallel tasks in a batch have disjoint write-sets meaningful.

## Decision

- **Safety refusals stay hard** and are independent of the plan: the path deny-list (secrets, `.git`, CI definitions, and so on) is always enforced inside the edit tools.
- **An undeclared edit is allowed, not refused.** It is recorded as a **scope expansion** with the task, path, and reason.
- **Collisions are refused.** An undeclared edit to a file that another task in the same batch declares or has edited is refused, so parallel work stays conflict-safe. The integration step may serialize the colliding tasks.
- **Too many expansions pause the run** with reason `scope`, so a human reviews before more work is done. The threshold is configurable.
- **The mode is configurable:** `strict` (refuse undeclared edits), `expand` (the default behavior above), or `open` (record only). A repo's config can only tighten the mode.
- **The rule is replaceable.** It lives behind one function that takes the task, path, plan, and batch context and returns allow, record, refuse, or pause. Glob lists ("always allow adjacent test files", "never allow `migrations/**`") are configuration; richer rules are code behind the same interface ([ADR-0013](0013-scale-seams-not-scale-features.md)).
- **Every decision is logged as an event**, and the count of expansions per plan is an eval metric (it measures how complete plans are).

## Alternatives considered

- **Hard allow-list:** the strongest plan fidelity, but brittle for the reasons above.
- **No scope check:** removes the batch-safety and plan-fidelity signals.

## Consequences

- The Planning validator's disjoint write-set check remains useful but is advisory about reality; the integration step is the arbiter.
- The Implementation design must specify the thresholds, the event shape, and how expansions appear in the final report.

## Revisit when

The Implementation design is written, or expansions in real runs show the default `expand` mode is too loose or too tight.
