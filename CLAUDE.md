# Mini shop – instructions for Claude Code

1. Before any work, read `docs/project-spec.md` (source of truth: scope, stack, rules, specification map).
2. Load the other specifications (`docs/*-spec.md`) **only when they concern the functionality being implemented** – the "Specification map" table in `project-spec.md` says which ones and when.
3. Work plan: `docs/implementation-plan.md`. One step = one coherent change. Do not go beyond the scope of the step.
4. Code and process rules (the "Rules" section in `project-spec.md`) always apply: clean code, good practices, YAGNI.
5. If a specification is ambiguous, incomplete or contradicts the state of the code – **ask, don't guess**.
6. After finishing a step: build and tests green, mark the step in `docs/implementation-plan.md`, update the specification if a contract changed.
7. Do not commit or push without an explicit request from the user. Commits:
   - never add Claude as an author or co-author (no `Co-Authored-By` trailer, no "Generated with Claude Code" lines);
   - consistent, concise messages: Conventional Commits (`type(scope): summary`), imperative mood, English, summary ≤ 72 characters; an optional body only with short bullet points explaining the "why".
8. Modules marked as **optional** (currently: Elastic Stack – `docs/observability-spec.md`; Prometheus + Grafana – `docs/monitoring-spec.md`; steps `O.*` in the plan) **are not implemented** until the user explicitly asks for it.
9. The English documents are authoritative. Polish versions in `docs/pl/` are reference copies – do not load them while working.
