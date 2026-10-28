# NFR: Agent Efficiency

> Binding rules. A deviation requires an explicit, reasoned proposal and the
> author's consent.

## Conserve tokens

- **Conserve tokens.** Treat token usage as a cost to minimize, the same way
  load time or boilerplate is.
- **Offload mechanical work to scripts or tools instead of doing it with model
  reasoning.** Anything deterministic and repeatable — bulk renames, format
  conversions, data transformation, report generation, repeated checks,
  multi-file edits following one pattern — is done by a script or an existing
  CLI tool, not step by step through the model.
- **Whether it is worthwhile is the agent's call.** Offload when the script is
  cheaper than doing the work directly or will be reused; do a genuine one-off
  directly when writing a script would cost more than it saves. No need to ask.
- A script that will be needed again is **kept in the project** (e.g. `scripts/`
  or `bin/`) so later runs cost nothing; a throwaway script is not committed.
- Prefer an existing, well-known tool over writing a new script — see the
  [code-style.md](code-style.md) "no NIH" rule.

## Parallelize independent work

The author often drops several feature ideas at once or in quick succession.
Waiting for them to be built one after another wastes wall-clock time.

- **Split, then fan out.** When a request (or a run of messages) contains
  several work packages, break it into packages and run the **independent** ones
  **in parallel** as subagents — launched together, in the background — instead
  of sequentially. This is the agent's call; no need to ask. State the split in
  one or two lines so the author can correct it.
- **Independent means:** disjoint files/modules and no ordering dependency.
  Packages that touch the same code, or where one needs the other's result, run
  **sequentially** or in one agent. A shared foundation (schema, shared module,
  API contract) is built **first**, then the dependent packages fan out on top.
- **Isolate each parallel package in its own git worktree / feature branch**
  branched from `dev`, so agents never edit the same checkout. The coordinating
  agent merges finished branches back into `dev` and resolves conflicts (see
  [git-workflow.md](git-workflow.md)).
- **The main agent coordinates, it does not duplicate.** It writes each
  subagent a self-contained brief (goal, scope, relevant files, applicable NFRs,
  what is already decided), waits for results instead of redoing the work, and
  integrates them. Separate test and review agents run at the **release gate**
  over the integrated result, not per package (see [testing.md](testing.md)).
- **Keep accepting new ideas while work runs.** A new, independent idea from the
  author starts as an additional background package instead of queuing behind
  the running ones; a dependent idea is queued behind what it depends on.
- **Balance against token cost** (see above): do not spawn an agent for work
  that is only a handful of tool calls — do it inline. Keep the fan-out to what
  actually saves time (typically 2–5 parallel packages).
