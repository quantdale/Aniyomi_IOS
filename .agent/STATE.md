# Current State

**Status:** SCAFFOLD_ONLY

The current `main` branch contains repository documentation and cross-harness planner/executor adapters only. No iOS application source, project/workspace file, package manifest, build command, test command, simulator target, or runnable product has been established in Git.

## Durable truth

- `AGENTS.md` is the canonical repository instruction entrypoint.
- `.agent/PLANNER_HANDOFF.md` defines how a planner may create an execution prompt and how an executor resumes one.
- No `.agent/EXECUTION_PROMPT.md` is present at this checkpoint.
- No product functionality is validated or implied by the repository name.

## Next legitimate transition

A future campaign must first define the intended iOS product scope and architecture from explicit requirements, then record an execution prompt with concrete milestones, dependencies, validation commands, and acceptance gates. Implementation should begin only after that planning step.

Update this file whenever the repository moves beyond scaffold-only state or when an active execution prompt is created, completed, blocked, or superseded.
