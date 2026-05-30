# Codex Subagent Orchestration Skill

A Codex skill for orchestrating native Codex subagents with a simple tier vocabulary:

- `high`
- `medium`
- `fast`

The skill defines when to use native Codex subagents, how to split exploration and implementation work, and how to map human-friendly tiers to `multi_agent_v1.spawn_agent` parameters.

## What It Does

This skill provides an operational workflow for native Codex subagents:

- Uses `explorer` subagents for read-only codebase investigation.
- Uses `worker` subagents for scoped implementation tasks.
- Keeps the main Codex agent as the orchestrator responsible for planning, integration, and validation.
- Enforces explicit prompts with file ownership, constraints, and acceptance criteria.
- Maps user tier names to native Codex model settings.

## Tier Mapping

| User tier | Codex model | Reasoning effort |
| --- | --- | --- |
| `high` | `gpt-5.5` | `medium` |
| `medium` | `gpt-5.3-codex` | `medium` |
| `fast` | `gpt-5.4-mini` | `medium` |

These tiers map to the `model` and `reasoning_effort` fields of `multi_agent_v1.spawn_agent`.

They are not `service_tier` values.

## Example Prompts

```text
Run a high exploration subagent to find where the form validation is implemented.
```

```text
Implement the task with a fast subagent.
```

```text
Use medium subagents in parallel to review the independent modules, then integrate the results.
```

## Installation

Copy the skill folder into your Codex skills directory:

```powershell
Copy-Item -Recurse -Force `
  ".\.codex\subagent-orchestration" `
  "$env:USERPROFILE\.codex\skills\subagent-orchestration"
```

After installing, restart Codex or reload the session so the skill metadata is discovered.

## Repository Structure

```text
.codex/
  subagent-orchestration/
    SKILL.md
    agents/
      openai.yaml
```

## Usage Notes

The skill is designed for explicit delegation requests. It should trigger when you ask Codex to use subagents, delegate work, run an exploration subagent, implement with a subagent, or run parallel agent work.

The orchestrator should still own the final decision-making:

- Decide scope before delegating.
- Assign exact files or modules to workers.
- Avoid overlapping write scopes.
- Review and validate subagent results.
- Close subagents when they are no longer needed.

## License

MIT. See [LICENSE](LICENSE).
