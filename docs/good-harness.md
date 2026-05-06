# Good Harness Design Patterns

This note is a local startup reference for the AutoAgent meta-agent. Read it
before changing `agent.py` so each experiment starts from a stable baseline and
changes the harness deliberately.

## What a good harness optimizes

A good autonomous-agent harness improves task success without making the system
brittle. For this repository, that means:

- Keep the Harbor adapter deterministic and stable.
- Improve the editable harness above the `FIXED ADAPTER BOUNDARY` in `agent.py`.
- Add general capabilities that can help many tasks, not task-specific hacks.
- Prefer simple changes that can be evaluated, kept, or discarded cleanly.
- Preserve the configured model unless `program.md` or a human explicitly says
  otherwise.

The first benchmark run on a branch should be the unmodified baseline. Without
that baseline, later score changes cannot be attributed to a harness idea.

## Experiment loop

Use one clear hypothesis per experiment:

1. Read the current instructions, harness, recent run logs, and failed-task
   traces.
2. Identify a recurring failure mode, such as weak file inspection, noisy shell
   output, missing verification, or insufficient domain-specific structure.
3. Make the smallest general harness change that targets that failure mode.
4. Run the same benchmark path used for the baseline.
5. Record the result in `results.tsv` with the commit, score, pass count, cost
   if available, status, and a short description.
6. Keep the change only when `passed` improves, or when performance is equal and
   the harness becomes simpler.

Avoid bundling several unrelated ideas into one run. A large mixed change makes
regressions hard to diagnose and makes useful ideas harder to keep.

## Tool design patterns

Start with tools when repeated task failures show that raw shell access is too
expensive or too error-prone. A strong tool has a narrow job and returns a
model-friendly observation.

Good tool surfaces usually have these properties:

- **Descriptive name:** choose names that match the model's expected action,
  such as `inspect_workbook`, `read_cells`, or `write_validated_csv`.
- **Typed arguments:** use simple argument names and type hints. Add constraints
  when invalid inputs are common.
- **Actionable output:** return compact structured summaries, not huge raw
  dumps. Include paths, row counts, changed fields, and clear error messages.
- **Bounded execution:** set timeouts or internal limits for slow operations and
  truncate overly large observations with guidance on how to request more.
- **Local validation:** check inputs before mutating files, and report exactly
  what was changed.
- **Idempotence:** make repeated calls safe where possible, especially for file
  creation and patch-style edits.

Keep `run_shell` available as an escape hatch, but do not force the model to
write boilerplate shell pipelines for common inspection or mutation operations.

## Tool anti-patterns

Avoid changes that make the harness look stronger in one trace while reducing
general reliability:

- Tool names that are vague, overloaded, or misleading.
- Huge tool outputs that bury the relevant facts.
- Silent exception handling that returns success-like text on failure.
- Mutable global state that leaks between tasks.
- Hardcoded assumptions about a specific benchmark task, file name, or verifier.
- Tools that modify task tests, verifier code, or fixed adapter behavior.
- New dependencies that are not required by the selected harness change.

If a tool needs a dependency, update `pyproject.toml` or `Dockerfile.base` only
when the experiment cannot be implemented reliably with existing dependencies.

## Orchestration patterns

The main agent should remain responsible for completing the user task and
producing the final artifact. Add orchestration only when it reduces repeated
failure modes.

Useful patterns include:

- **Plan-and-act main agent:** keep a single main agent that plans, executes,
  inspects results, and decides when the task is complete.
- **Verification pass:** add a bounded checker that reviews the final artifact
  against the instruction before the run finishes.
- **Specialist subtask:** delegate a narrow cognitive task, such as reading a
  complex spreadsheet or checking a generated report, while the main agent keeps
  control.
- **Handoff:** transfer the conversation to a specialist only when that
  specialist should take over the remainder of the run.

For verification and other bounded subtasks, prefer exposing the specialist via
`agent.as_tool()` so the main agent receives the specialist's answer and can
continue. Use handoffs when the destination agent should own the conversation
after delegation.

## AutoAgent-specific guardrails

This repository's runtime boundary matters:

- Edit only the harness section above the `FIXED ADAPTER BOUNDARY` unless the
  ticket or human explicitly asks for adapter changes.
- Keep `AutoAgent.run` and ATIF trajectory serialization stable so Harbor can
  evaluate the harness consistently.
- Remember that `environment.exec(...)` runs commands in the task environment,
  not as a host-side convenience command.
- Keep task artifacts in the task workspace and avoid host-only assumptions.
- Preserve enough logging for diagnosis without bloating trajectories.

When adding tools around `environment.exec(...)`, return observations the model
can act on directly: command status, relevant stdout/stderr excerpts, paths
created, and suggested recovery steps for common failures.

## Validation checklist

Before keeping a harness change:

- Confirm `agent.py` imports cleanly.
- Run the unmodified baseline first on the same task set.
- Rebuild the base image if dependencies or Docker files changed.
- Run the target benchmark path from `program.md`.
- Inspect `run.log`, task trajectories, and verifier output for regressions.
- Record the result in `results.tsv` before deciding to keep or discard.

Documentation-only changes, like this helper note, do not require a benchmark
rerun because they do not alter AutoAgent runtime behavior.
