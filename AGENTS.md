# Agents

## SDLC workflow

Every change to code in `backend/` or `frontend/` follows this pipeline. The main agent orchestrates: it delegates each step to the named subagent in `.claude/agents/`, passes the output on, and does the implementation itself. Edits to docs or `.claude/` config skip the pipeline.

| Step | Owner | Model | Input | Output |
|---|---|---|---|---|
| 1. Analyze | `feature-analyst` | Haiku | the task and the affected area | features and business rules with `path:line` |
| 2. Specify | `bdd-writer` | Sonnet | step 1 report, pasted into the prompt, plus the task | `backend/tests/features/<area>.feature` |
| 3. Implement | main agent | session model | the `.feature` files | code and pytest tests that cover the scenarios |
| 4. Simplify | `simplify-agent` | Haiku | the files changed in step 3 | ranked simplification suggestions |
| 5. Review | `review-agent` | Sonnet | the `.feature` paths and the changed files | scenario coverage table, findings, verdict |

Rules:

- Run the steps in order. Each step starts only after the previous one reports.
- Subagents start with no context. Give each one the task, the scope, and the previous step's output in its prompt.
- Step 3: do not implement behavior that has no scenario. A scenario that turns out wrong goes back to step 2.
- Step 4: apply the suggestions that keep behavior identical, then run `cd backend && uv run pytest`.
- Step 5: fix every critical and major finding, run the tests, and run `review-agent` again. Stop when the verdict is "ready" or after two review rounds, then report what is still open.
- Report to the user after step 5: the feature files, the changed files, the test result, and the review verdict.
