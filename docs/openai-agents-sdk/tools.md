# OpenAI Agents SDK Tools, Handoffs, and Agents as Tools

This note is the local startup reference that `program.md` expects for OpenAI
Agents SDK tool design. It is tailored to the Python SDK usage in `agent.py`,
where AutoAgent builds an `Agent`, registers tools, and runs it with
`Runner.run(...)`.

Official SDK references used for this summary:

- <https://openai.github.io/openai-agents-python/tools/>
- <https://openai.github.io/openai-agents-python/handoffs/>

## Baseline in this repository

The current harness imports `Agent`, `Runner`, and `function_tool` from
`agents`. The editable flow is:

- `create_tools(environment)` wraps callable capabilities for the model.
- `create_agent(environment)` constructs the main `Agent` with instructions,
  tools, and model configuration.
- `run_task(environment, instruction)` calls `Runner.run(...)` with the main
  agent and the task instruction.

Keep this shape in mind when adding SDK features. A good change should improve
the editable harness while leaving the fixed Harbor adapter untouched.

## Choosing the right SDK pattern

Use the smallest SDK feature that matches the need:

- **Function tool:** deterministic action or inspection implemented in Python.
- **Agent as tool:** bounded specialist reasoning where the main agent should
  receive a result and continue.
- **Handoff:** specialist should take over the conversation after delegation.
- **Manual `Runner.run(...)` inside a tool:** advanced orchestration that needs
  custom retries, chaining, fallback logic, or output extraction beyond
  `agent.as_tool()` defaults.

For AutoAgent benchmark work, function tools and agents-as-tools are usually the
safest first options because the main agent keeps responsibility for final task
completion.

## Function tools

`@function_tool` exposes a Python function to the model. The SDK can derive the
tool name, description, and argument schema from the function name, docstring,
and type hints. Use this for capabilities that should run in the task
environment or compute a deterministic summary for the agent.

Design function tools so the model can call them correctly on the first try:

- Use a verb-based function name that describes the exact action.
- Give every argument a clear type and a simple name.
- Put model-facing behavior, limits, and return shape in the docstring.
- Return compact observations with explicit failure states.
- Use timeouts or internal limits for slow or high-volume operations.
- Validate before mutating files, then report what changed.

Example shape for this harness:

```python
from shlex import quote

@function_tool
async def inspect_path(path: str) -> str:
    """Summarize a file or directory in the task environment."""
    result = await environment.exec(command=f"ls -la {quote(path)}", timeout_sec=30)
    if result.stderr:
        return f"ERROR inspecting {path}: {result.stderr}"
    return result.stdout or "(no output)"
```

When implementing real tools, avoid interpolating untrusted strings into shell
commands without quoting. Prefer Python file APIs or a structured command
builder when possible.

## Agents as tools

`agent.as_tool()` exposes an agent as a callable tool for another agent. The
outer agent keeps control of the run, calls the specialist when useful, receives
the specialist's final output as the tool result, and then decides what to do
next.

This is a strong fit for bounded cognitive subtasks:

- Verify a produced artifact against the original instruction.
- Summarize a long inspection result into risks and required fixes.
- Translate a domain-specific representation into an implementation plan.
- Review a generated file for consistency before final answer.

Minimal pattern:

```python
review_agent = Agent(
    name="artifact_reviewer",
    instructions=(
        "Review the proposed task result against the user's instruction. "
        "Return only concrete issues and a pass/fail recommendation."
    ),
    model=MODEL,
)

main_agent = Agent(
    name="autoagent",
    instructions=SYSTEM_PROMPT,
    tools=[
        *create_tools(environment),
        review_agent.as_tool(
            tool_name="review_artifact",
            tool_description="Check the final artifact before finishing.",
        ),
    ],
    model=MODEL,
)
```

By default, an agent-as-tool receives a single string input from the outer
agent. Use structured parameters when the specialist needs fields such as
`instruction`, `artifact_path`, and `rubric` instead of one free-form string.
Use custom output extraction when the outer agent needs only a specific payload,
such as normalized JSON or a short pass/fail summary.

Prefer `agent.as_tool()` over handoff when the main agent must keep ownership of
the task, run additional tools afterward, or decide whether the specialist's
output is sufficient.

## Handoffs

Handoffs let one agent delegate the active conversation to another agent. In the
SDK, handoffs are presented to the model as tools. A handoff is exposed with a
generated transfer-style tool name unless overridden.

Basic pattern:

```python
from agents import Agent, handoff

spreadsheet_agent = Agent(
    name="spreadsheet_agent",
    instructions="Complete spreadsheet-focused tasks accurately.",
    model=MODEL,
)

main_agent = Agent(
    name="autoagent",
    instructions=SYSTEM_PROMPT,
    tools=create_tools(environment),
    handoffs=[handoff(spreadsheet_agent)],
    model=MODEL,
)
```

Use handoffs when the destination agent should own the next phase of the
conversation. Avoid them when the main agent needs to inspect the specialist's
answer and continue orchestrating the task.

Useful handoff controls:

- `handoff_description` on a plain agent helps the model know when to delegate.
- `handoff(...)` can override the generated tool name and description.
- `on_handoff` can run setup or logging when the transfer happens.
- `input_type` asks the model for small structured metadata, such as a reason,
  priority, language, or summary.
- `input_filter` controls what conversation history the receiving agent sees.
- Recommended handoff prompt helpers can be added to agent instructions when
  models need clearer delegation behavior.

Remember that a handoff is not just a subroutine call. By default, the receiving
agent sees conversation history and takes over the run, so the pattern should be
used intentionally.

## Applying these patterns to AutoAgent

When improving this harness:

- Start with the observed failure mode, not with a feature idea.
- Add a function tool for repeated deterministic work that raw shell handles
  poorly.
- Add an agent-as-tool for bounded review or specialist reasoning where the main
  agent should continue afterward.
- Add a handoff only for a specialist that should take over the conversation.
- Keep tool output short enough for the next model turn to use directly.
- Preserve `MODEL = "gpt-5"` unless the human explicitly changes that
  constraint.
- Update imports and type hints only as required by the SDK feature actually
  used.

For documentation-only tickets, do not edit `agent.py`; verifying that these
local references exist is sufficient.
