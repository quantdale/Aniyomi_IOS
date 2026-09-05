# Aniyomi_IOS — Agent Instructions

These instructions are the canonical repository-level contract for any coding or planning agent working here. Harness-specific adapters are subordinate to this file.

## Repository truth

This repository is currently a **project scaffold, not an implemented iOS application**. The current `main` branch contains documentation and agent/planner adapters but no application source tree, package manifest, build system, tests, or runnable target. Never invent missing architecture, commands, features, dependencies, or completion status.

Evidence precedence:

`current repository contents and executable validation` > `current source/configuration` > `active agent state and execution prompt` > `living documentation` > `historical discussion` > `assumptions`.

If documentation conflicts with stronger evidence, update the living documentation instead of rationalizing the discrepancy.

## Required reading

Before substantial work:

1. Read `README.md`.
2. Read `.agent/STATE.md` and `.agent/PLANNER_HANDOFF.md`.
3. If `.agent/EXECUTION_PROMPT.md` exists, determine whether it is still active and reconcile it against current `main` before continuing.
4. Inspect the repository tree and recent commits before making architecture or status claims.

## Planning and implementation discipline

- Do not start implementation from the repository name alone. Establish an explicit product goal and architecture first.
- For `/goal continue`, resume only genuinely incomplete work from an active execution prompt or native state; do not redo landed work.
- Break substantial work into verifiable milestones. Validate each milestone with repository-provided commands once such commands exist.
- Never claim a build, test, simulator, device, or feature result that was not actually executed and observed.
- If required tooling or platform access is unavailable, record the limitation precisely rather than converting it into a pass.

## Documentation discipline

Keep living documentation synchronized with changes that affect setup, architecture, commands, behavior, limitations, validation, or current status. Preserve historical records as historical evidence; do not rewrite old decisions or reports to make them look current.

Avoid volatile persisted claims such as "current HEAD is <sha>" unless a document is explicitly a historical checkpoint. Discover live Git state from Git whenever needed.

## Git safety

- Work from the repository's current default branch unless an explicit task requires a branch.
- Inspect the diff before committing.
- Do not discard unrelated work, rewrite history, force-push, or use destructive recovery merely to obtain a clean tree.
- Commit only validated, intentional changes and report what remains unverified.

More specific future instructions in a subdirectory may add stricter constraints for that subtree; they must not silently weaken this repository-level contract.
