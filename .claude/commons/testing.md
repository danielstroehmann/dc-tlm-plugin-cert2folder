# NFR: Testing & Quality

> Binding rules. A deviation requires an explicit, reasoned proposal and the
> author's consent. Thorough testing is a top priority — the time for it is
> always available.

## Regression tests are mandatory

- Every feature, bug fix, and change ships with **automated technical tests** —
  not just an in-session check by Claude. The tests live in the repo and run in
  CI and at the release gate.
- **Tests MUST pass before any release.** A red test blocks the release (see
  "When the gates apply" below).
- The primary purpose is **regression safety**: new work must not silently break
  existing functionality. This is the #1 recurring pain point in these projects —
  treat "does this break anything that already worked?" as a first-class concern.

## What every test set covers

- **Happy path** — the expected functionality works as intended.
- **Expected error cases** — the known failure modes fail in the expected,
  handled way (not just "it throws something").

## Honest tests only

- **NEVER write self-congratulatory or vanity tests** — no assertions that
  cannot fail, no tests written to inflate a green count. A test must genuinely
  exercise behavior and be capable of failing when the behavior is wrong.
- When reporting test results, state plainly what passed, what failed, and what
  was skipped. Do not overstate coverage.

## When the gates apply: inner loop vs. release gate

The heavy, long-running checks are a **gated check-in before a release**, not a
toll on every iteration.

- **Inner loop — no gate.** Local deployments and deployments **from the local
  repo to a dev/test system in the home-lab** do NOT require the separate test
  agent, the reviewer agent, `/security-review`, or a full test run. Iterate and
  deploy freely. Cheap, fast checks (build succeeds, quick unit tests) are the
  agent's call and never block such a deploy.
- **Release gate — mandatory.** Before a release (i.e. before the `v<VERSION>`
  tag is pushed, see [git-workflow.md](git-workflow.md)) everything in this file
  applies in full, over **all changes since the last release**:
  1. the separate test agent writes the missing tests and runs the **full**
     suite (all levels below, incl. the upgrade test from
     [deployment.md](deployment.md));
  2. the separate reviewer agent reviews the full diff since the last release;
  3. `/security-review` with OWASP coverage (see [security.md](security.md)).
  Findings are fixed (or explicitly deferred with the author's consent) and the
  suite is green **before** the tag is pushed. A red gate blocks the release.
- Tests for each change must exist **by the release gate** at the latest; they
  may be written in a batch there instead of alongside every iteration.
- **A release gate never blocks work on the next feature.** The gate runs on a
  frozen snapshot — the `release/v<VERSION>` branch (see
  [git-workflow.md](git-workflow.md)) in its own worktree — as background work,
  while feature development on `dev` continues in parallel (see
  [agent-efficiency.md](agent-efficiency.md)). Gate fixes are committed on the
  release branch and merged back into `dev`; new features do not enter a
  release that is already being gated.

## Independent test & review agents

Implementation and verification are **separated**. The agent that writes a
change must not be the only one that vouches for it — an in-session "I checked
it myself" by the implementing agent does **not** satisfy these rules.

- **Tests are written and executed by a separate test agent** — a dedicated
  agent instance distinct from the one that implemented the change. It authors
  the test cases and runs them; it is critical of the implementation and
  reports defects rather than papering over them.
- **Every change gets a separate review run by a reviewer agent** at the release
  gate, before it is released. The reviewer is independent of the implementer, reviews the
  full diff, and its findings are addressed (or explicitly deferred with the
  author's consent) before the change lands.
- **Executable test levels.** Where the stack allows it, the standard set is
  **unit**, **integration**, **component**, and **smoke** tests — all committed
  to the repo, all runnable, all mandatory in CI. Smoke tests must be runnable
  against a live endpoint so a deployment can be verified from the outside.

## Security testing

- Test against the relevant OWASP classes for the feature, not only the
  functional behavior (see [security.md](security.md)).
