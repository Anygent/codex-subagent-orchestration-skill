---
name: subagent-orchestration
description: Workflow and guidance for using native Codex subagents through multi_agent_v1.spawn_agent with user-defined high, medium, and fast tiers. Use when the user explicitly asks for Codex native subagents, delegation, exploration, implementation by subagent, or parallel agent work, including Spanish requests such as "Ejecuta un subagente de exploracion high" or "Implementa la tarea con subagente fast".
---

# Subagent Orchestration

This document unifies the operational guidance for orchestrating implementations with native Codex subagents using `multi_agent_v1.spawn_agent`.

## Purpose

Use this workflow to deliver complex changes with high reliability while keeping orchestrator context clean.

- The orchestrator owns decisions and validation.
- Subagents execute scoped exploration or implementation stages.
- Discovery, planning, delegation, and QA are done in a strict sequence.
- User tier names map to native Codex model overrides.

## Tier Map

Use these mappings whenever the user requests a subagent tier:

| User tier | Native model | Reasoning effort |
| --- | --- | --- |
| `high` | `gpt-5.5` | `medium` |
| `medium` | `gpt-5.3-codex` | `medium` |
| `fast` | `gpt-5.4-mini` | `medium` |

Set these values on `model` and `reasoning_effort` in `spawn_agent`.

Do not use `service_tier` for the user tiers `high`, `medium`, or `fast`. Leave `service_tier` unset unless the user explicitly asks for a native service tier such as `priority`.

## Core Principles

- Orchestrator decides architecture, file scope, naming, and approach.
- Workers execute the given spec; they do not invent architecture.
- Prefer delegation when the user explicitly asks for subagents and the task can be split safely.
- Keep prompts explicit and stage-bounded.
- Validate progressively after each stage.
- Avoid redundant full test runs after each stage when subagents already report success; run final validation once at the end if needed.

## Role Separation (Critical)

| Responsibility | Orchestrator | Explorer | Worker |
| --- | --- | --- | --- |
| Architectural decisions | Decide before delegation | Supports context and evidence gathering | Must not invent |
| Implementation approach | Define step-by-step | Contrasts options and surfaces tradeoffs | Must follow spec |
| File structure and naming | Specify explicitly | Helps locate relevant files and ownership | Must not assume |
| Information quality | Validate key assumptions | Navigates noisy or dispersed data, highlights signal | Consumes finalized spec |
| Diagnosis and debugging | Owns final judgment | Performs lightweight diagnostics and logical tracing | Applies approved fixes |
| Code execution | Delegate when useful | Does not edit files unless explicitly requested | Implements the given plan |

Never send open-ended design questions to a worker.

## Native Agent Roles

What you will typically use in Codex native subagents:

- `explorer`: read-only exploration, diagnosis, codebase context, and targeted investigation.
- `worker`: autonomous implementation, coding, shell execution, and bounded fixes.
- `default`: only use when neither `explorer` nor `worker` fits the request.

## When to Delegate

Delegate when:

- The user explicitly asks for subagents, delegation, or parallel agent work.
- Multiple interconnected files require consistent updates.
- A refactor must be systematic across modules.
- Implementation is repetitive or high-volume.
- You need deep repo exploration before coding.
- Independent sidecar work can run while the orchestrator continues non-overlapping local work.

Do not delegate only because the user asks for depth, thoroughness, research, investigation, or detailed codebase analysis. Those requests do not count as permission to spawn unless they explicitly mention subagents or delegation.

# Everyday Complex Workflow

### 0) Contextualize Native Subagents

- Confirm the user explicitly requested subagents or delegation.
- Identify whether each requested subagent should be `explorer` or `worker`.
- Apply the requested tier using the Tier Map.
- Use `fork_context: true` when the subagent needs the current conversation context.
- Leave `model` and `reasoning_effort` unset only when the user requests a subagent but does not specify a tier.

### 1) Discovery Recon (Fast Explorer)

After selecting the right explorer, run a quick reconnaissance pass with the `fast` tier when the user asks for exploration and does not request a stronger tier.

- Identify likely change surfaces and ownership boundaries.
- Locate relevant modules and entry points.
- Capture only high-signal findings for planning.
- Reuse an existing explorer with `send_input` when the follow-up is highly dependent on its previous context.

### 2) Plan Exhaustively in Stages

Produce a deterministic stage plan before implementation.

For each stage define:

1. Objective
2. Exact files to modify
3. Step-by-step logic
4. Constraints and non-goals
5. Acceptance criteria

Keep stages small and verifiable.

### 3) Post-Plan Exploration Pass (Fast Explorer)

Before execution, run a targeted explorer pass to validate plan assumptions when the scope is ambiguous.

- Confirm file list completeness.
- Confirm dependency and side-effect assumptions.
- Refine worker prompts with evidence.

Resume a previous explorer only when coherent with current scope and files.

### 4) Recommend Subagent Per Stage

Attach a recommended native role and tier to each stage.

- Use `fast` workers for mechanical or high-volume edits.
- Use `medium` workers for balanced implementation tasks.
- Use `high` workers for ambiguous, cross-cutting, or complex logic.
- Use explorers for diagnostics and repo context.

### 5) Request User Authorization

If the user asked for a staged delegation plan but did not authorize execution, present:

- stage sequence,
- assigned native subagent role per stage,
- assigned tier per stage,
- any planned reuse of existing subagents.

Execute only after user approval in that case.

If the user directly asks to execute a specific subagent task, execute without asking again.

### 6) Delegate Sequentially or in Safe Parallel

Run stages one by one unless they have disjoint write scopes and can safely run in parallel.

- Do not merge multiple stages in one request unless explicitly planned.
- Do not issue multiple agents against the same write scope.
- Do not wait reflexively; while a subagent runs, continue meaningful non-overlapping local work.
- Keep execution logs concise and focused on stage outcomes.

### 7) QA After Each Stage

After each delegation:

- Review changed files and scope.
- Verify the stage met its acceptance criteria.
- Fix small local gaps directly when faster than re-delegating.
- Keep global test execution minimal during intermediate stages if subagents already ran build or tests successfully.
- Close subagents when they are no longer needed.

### 8) Final Validation and Wrap-up

After all stages:

- Run one final comprehensive validation if needed.
- Ensure docs and config remain consistent with runtime behavior.
- Summarize outcomes, risks, and optional next steps.

## Golden Rule: Absolute Specificity in Prompts

Vague prompts cause agents to overwrite critical logic. All architectural decisions, naming conventions, and methodologies must be fixed by the orchestrator before invoking a worker. Write the prompt like a finalized, non-negotiable ticket, not the start of a conversation. Always write subagent prompts in English, even if the user speaks another language or the codebase comments are in another language.

### Required Prompt Structure

1. **Objective** - What to build or fix, in 1-2 precise sentences.
2. **Key Files** - Exact absolute paths of every file the agent is allowed to touch.
3. **Step-by-Step Logic** - The algorithm or flow you expect, in ordered steps.
4. **Functional Constraints** - Function names, signatures, return types, existing variables and patterns to preserve.
5. **UI/UX Specs** (if frontend) - Visual hierarchy, disabled states, component libraries, and button labels verbatim.
6. **Done When** - A short checklist the agent must satisfy before considering the task complete.

Always tell workers:

- They are not alone in the codebase.
- They must not revert edits made by others.
- They must adjust their implementation to accommodate existing changes.
- They must list every changed file path in the final answer.

### Bad Prompt

> "Implement LLM-optimized exports in DownloadActions.tsx and shuffle speaker codenames better."

Why it fails: "LLM-optimized" is undefined, the exact target file path is missing, the algorithm is unspecified, and no boundaries are set. The agent will invent an architecture, likely overwrite existing logic, and may touch unrelated files.

### Good Prompt

> We need to implement LLM-optimized Text and SRT exports in
> `d:\your-project\src\components\DownloadActions.tsx`.
>
> Goal: consolidate consecutive segments from the same speaker to avoid repeating the speaker name on every sentence. The combined segment takes `start` from the first and `end` from the last segment; texts are joined with a space.
>
> Requirements:
> 1. Create `consolidateSegments(segments: Segment[]): Segment[]`.
> 2. Use it to produce `llmTranscriptText` and `llmSrtText` variants.
> 3. Add secondary UI under the existing export buttons:
>    - "Copy transcript for LLMs"
>    - "Copy SRT for LLMs"
>    - "Download SRT for LLMs"
> 4. Use existing Tailwind/Lucide patterns. Add Sparkles or Bot icons. Respect `hasSegments` disabled states.
> 5. Do not modify any files other than `DownloadActions.tsx`.
>
> Done When:
> - `consolidateSegments` is implemented and used for both LLM variants.
> - The three new buttons render correctly with proper disabled behavior.
> - Styling matches existing Tailwind/Lucide patterns.
> - Only `DownloadActions.tsx` was changed.

Why it works: Every architectural decision is made upfront: the exact file, the algorithm, the output variables, the UI copy, the icon library, the disabled-state logic, and what not to touch.

## Using `explorer` for Exploration

Prefer `explorer` over manual file browsing when:

- The user explicitly asks for a subagent of exploration.
- Locating where a feature, endpoint, or component is implemented.
- Understanding how a function, class, or data flow works.
- Discovering cross-cutting dependencies before making an architectural change.
- Onboarding to an unfamiliar codebase.
- Investigating failing tests, logs, pipelines, or documentation questions in parallel with local work.

Example `spawn_agent` call for "Ejecuta un subagente de exploracion high":

```json
{
  "agent_type": "explorer",
  "model": "gpt-5.5",
  "reasoning_effort": "medium",
  "fork_context": true,
  "message": "How are Server-Sent Events (SSE) for job streaming implemented in this frontend? Which hooks and components listen to real-time transcription status updates? Do not edit files. Return concise findings with file paths."
}
```

## Using `worker` for Implementation

Prefer `worker` when:

- The user explicitly asks to implement with a subagent.
- The write scope is bounded and can be specified exactly.
- The task can be executed from a finalized spec.
- Parallel workers can own disjoint files or modules.

Example `spawn_agent` call for "Implementa la tarea con subagente fast":

```json
{
  "agent_type": "worker",
  "model": "gpt-5.4-mini",
  "reasoning_effort": "medium",
  "fork_context": true,
  "message": "Objective: implement the requested change. Key Files: list exact absolute paths here. Step-by-Step Logic: list exact steps here. Functional Constraints: preserve existing public APIs and style. Done When: run the specified checks and list every changed file path. You are not alone in the codebase; do not revert edits made by others, and adjust to existing changes."
}
```
