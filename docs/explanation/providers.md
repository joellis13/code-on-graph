# Providers

**Type:** Explanation · **Status:** Draft v1 · **Last updated:** 2026-10-15
**Diagram:** [`providers.drawio`](../diagrams/providers.drawio)
**Decisions:** [ADR-0004](../decisions/0004-two-provider-interfaces.md), [ADR-0005](../decisions/0005-direct-providers-first.md), [ADR-0011](../decisions/0011-platform-support-and-implementation-execution-environment.md)

Facts about vendors in this page were checked on **2026-10-06** and change quickly. Items marked **(verify)** must be rechecked at build time. Signatures below are illustrative, not project code.

---

## 1. Two kinds of "provider"

|                       | `LLMClient`                                                                                | `AgentRunner`                                                           |
| --------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------- |
| What it is            | A model endpoint: messages in, completion out                                              | An agent product or loop that edits files and runs commands             |
| Who controls the loop | cog                                                                                        | The adapter (its own tools and prompts), or cog if it is `NativeRunner` |
| Used by               | Planner, query generation, plan review, gap resolution                                     | Implementation workflow (executing task packets)                        |
| Needs                 | Structured output, a small read-only tool budget, per-call logging, untrusted text as data | A working directory, a policy, a budget, acceptance results             |
| Examples              | Fireworks, OpenRouter, local servers                                                       | `NativeRunner`, Copilot, Antigravity, Claude Code                       |

Keeping them separate is what lets Planning keep principle 8 (LLM outputs are proposals, not facts) while Implementation can still use agent products. Planning needs **no** agent CLI.

```mermaid
flowchart LR
  N1[planner · query_gen · review] --> LC[LLMClient]
  N2[Implementation executor] --> AR[AgentRunner]
  LC --> T[Tier router<br/>config: tier → adapter + model]
  AR --> T
  T --> A1[OpenAI-compatible adapter<br/>Fireworks · OpenRouter · local]
  T --> A2[NativeRunner<br/>uses LLMClient + tools]
  T --> A3[Copilot runner]
  T --> A4[Antigravity runner]
  A2 --> LC
```

---

## 2. `LLMClient`

### Contract (illustrative)

> generate(tier, messages, schema=None, tools=None, tool_budget=0, cache_key=None) -> LLMResult

- **Structured output:** when a schema is given, the result is parsed and **validated with Pydantic by cog** whatever the provider enforces. Invalid output enters a bounded repair loop; exhaustion returns a status, not an exception.
- **Tool loop:** tools are read-only, typed functions with a hard call budget (Planning allows at most 8 planner tool calls). Every call is logged.
- **Result:** parsed object or text, `Usage` (input, output, cached tokens; cost, or `None` when unknown, never `0`), the prompt version id, the resolved model, latency, and a status.
- **Defaults:** temperature 0 for extraction and validation-style tasks.

### Adapters

**OpenAI-compatible adapter (first).** One adapter serves Fireworks, OpenRouter, and local servers (Ollama, vLLM). Provider differences are _config_, recorded as quirks:

| Provider                 | Structured output                             | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------------------ | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Fireworks**            | JSON mode with a JSON schema, or grammar mode | Using a JSON-schema response format disables reasoning output on models that emit it; an older docs page also says the model does not "see" the schema in that mode, so **include the schema in the prompt too**. Function calling enables JSON mode automatically. **(verify per model)**                                                                                                                                                                 |
| **OpenRouter**           | `response_format` of type `json_schema`       | Support is **per endpoint, not just per model**: the same model may be served by several providers, only some supporting structured output. Set `require_parameters: true` in provider preferences so you are never silently routed to an endpoint that ignores your schema. A response-healing plugin exists for non-streaming calls. Provider preferences (ordering, fallbacks, data-collection policy) matter for egress; **(verify)** current options. |
| **Local (Ollama, vLLM)** | Varies by server and model                    | Cheap option for small tiers and query generation.                                                                                                                                                                                                                                                                                                                                                                                                         |

Rules for every provider:

- Capabilities are **declared in config per model** (structured output mode, tool calling, streaming, context window), not assumed. A tier whose model lacks a needed capability fails at `preflight`, not mid-run.
- Always validate and repair.
- Keep per-model-family **prompt overrides**: the same system prompt rarely performs equally across models.

**Claude models.** OpenRouter serves Claude models, so a Claude model can fill any role through the adapter above from Phase 1 (for example, as the cross-family reviewer). A **native Anthropic adapter** using the official `anthropic` SDK is the better route if a Claude model becomes a regular planner or reviewer, because structured outputs, prompt caching, and effort controls are exposed directly there rather than through a compatibility layer. Decide from the comparison in the verification log (V45).

**Other wrappers.**

- `FakeLLM`: returns canned (including deliberately invalid) outputs for tests.
- `RecordReplay`: modes `live`, `record`, `replay`, `cache-only`, keyed by a hash of model, prompt version, rendered messages, and parameters. Powers evals and the node 11 query cache without re-spending tokens.

---

## 3. `AgentRunner`

### Contract (illustrative)

> run(task_packet, workdir, policy, budget, session=None) -> AgentResult
> capabilities: structured_output · sessions · mcp · reports_cost · takeover · native_shell · shell

- **Input:** a task packet from an approved plan, a working directory (a git worktree), the active policy, and a budget.
- **Output:** status (`succeeded`, `failed`, `needs_human`, `budget_exceeded`), the diff and changed files, a transcript reference, usage, and **acceptance results run by cog**, never trusted from the agent's own claims.
- **Policy** is enforced by cog's `ExecutionEnvironment` and tool layer wherever the runner allows, and by sandbox boundaries where it does not.

### Runners

| Runner                             | Role                                                                                                                                                                           | Status                                                                                                                                                                                                                                                                                                           |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`NativeRunner`**                 | cog's own tool-calling loop on `LLMClient`, with tools such as `read_file`, `grep`, `symbol_lookup`, `apply_patch`, and allow-listed `run_command`, all passing through policy | The intended Implementation runner, and a purpose of the project: cog owns the whole loop, with no third-party prompts nested inside its own. The hard parts are reliable edits and context management, which commercial CLIs sell; the other runners are baselines to measure it against. Built after Planning. |
| **Copilot** (SDK or CLI)           | Optional runner; also a baseline to compare `NativeRunner` against                                                                                                             | See below                                                                                                                                                                                                                                                                                                        |
| **Antigravity** (SDK or CLI)       | Optional runner, mainly where models are cheaper through the owner's subscription                                                                                              | See below                                                                                                                                                                                                                                                                                                        |
| **Claude Code** (CLI or Agent SDK) | Optional runner; a baseline to compare `NativeRunner` against; usable by hand for the Phase 3 execution trial                                                                  | See below                                                                                                                                                                                                                                                                                                        |
| **`FakeRunner`**                   | Tests                                                                                                                                                                          | Planned                                                                                                                                                                                                                                                                                                          |

### Copilot

- A Copilot SDK became generally available on 2026-06-02, with Python among the supported languages, and the Python package bundles the CLI. **(verify)** current status and package names.
- The CLI has non-interactive (`-p`) and interactive (`-i`) modes. Non-interactive use requires an allow-all permissions setting, which makes the sandbox mandatory.
- **BYOK** lets the CLI use your own provider through environment variables: `COPILOT_PROVIDER_BASE_URL`, `COPILOT_PROVIDER_API_KEY` (optional for some local setups), and `COPILOT_PROVIDER_TYPE` (`openai`, the default, `azure`, or `anthropic`). OpenAI-compatible endpoints, including Ollama and vLLM, are supported.
- **BYOK value:** it gives you Copilot's agent loop (editing, shell, permissions, sessions) paid per token to Fireworks or OpenRouter. It is the **fastest stopgap for Implementation** while `NativeRunner` matures, and a baseline for comparison. It adds nothing to Planning.
- **BYOK costs:** the model must support tool calling and streaming; `--effort` is rejected for custom models; the CLI cannot send custom request headers; you do not control its system prompt or tools, so Planning's tool-budget and logging rules cannot apply inside it; and it is a moving target (pin versions). Whether BYOK mode still requires a GitHub login is unclear in the sources: **(verify by testing on the owner's account)**.
- **Windows:** works interactively from Git Bash for the owner today. `cog` runs inside WSL2 ([ADR-0018](../decisions/0018-wsl2-home-and-container-execution.md)), so the CLI must be installed and signed in there, and unattended runs need verification inside WSL2.

### Antigravity

- The owner holds an Antigravity subscription and no other agent subscription.
- Google's docs describe an `agy` CLI with a headless mode for CI and scheduled automation, and a separate Python SDK (`google-antigravity`) with structured output, policies, hooks, MCP, subagents, and persistence. The SDK authenticates with a Gemini API key or Vertex, **not** the subscription. **(verify)**
- The CLI uses interactive sign-in and subscription quotas. Plans differ in model access: all plans include Gemini models, and third-party models are listed under the Ultra tier, so check the current Models page for the owner's tier. **(verify)**
- A public forum thread asks Google whether orchestrating the official CLI headlessly from a third-party multi-agent app is permitted under the terms. **No official answer** was found in the sources read. Treat subscription-backed automation as unconfirmed.
- The `agy` CLI replaced Gemini CLI for individual developers in mid-2026, a reminder that flags and tools churn.

### Claude Code

- **Two forms.** The Claude Code CLI in headless mode (`claude -p`) with machine-readable output (`--output-format json` or `stream-json`), a tool allow-list (`--allowedTools`), a permission mode, a turn limit (`--max-turns`), and a model choice (`--model`). And the **Claude Agent SDK** (Python package `claude-agent-sdk`), which packages the same harness as a library with hooks and permission callbacks. **(verify flags and SDK options, V44)**
- **Value:** an edit loop tuned together with Claude models, which makes it the strongest baseline for `NativeRunner`'s edit reliability. The SDK's hooks and permission callbacks may let `cog` enforce policy at each tool call rather than only at the sandbox boundary **(verify, V44)**.
- **Models:** Claude models only (for example, Claude Haiku 5.5 as a small tier, Claude Sonnet 5.5 or Claude Opus 5.5 as larger ones). It is not a way to run Fireworks or OpenRouter models; Copilot BYOK covers that.
- **Limits:** like every vendor runner, its own system prompt and tools sit around `cog`'s task packet, so it is a runner and a baseline, never part of Planning ([ADR-0004](../decisions/0004-two-provider-interfaces.md)).
- **Windows:** installed and signed in inside WSL2, like the other CLIs.

### Claude terms of use

Anthropic's Claude Code documentation says subscription sign-in is for ordinary use of Claude Code and native apps, developers building products or services (including with the Agent SDK) should use API keys, and routing requests through Free, Pro, or Max credentials on behalf of users is not permitted. A planned credit change for `claude -p` and Agent SDK usage was paused in June 2026. Terms have changed repeatedly; **read the current text before relying on anything here**. An API key (`ANTHROPIC_API_KEY`) is the unambiguous route for scripted use. Whether scripted runs of the unmodified CLI on the owner's own subscription count as ordinary personal use is settled before the Phase 3 trial (V20).

### Rules for every subscription-backed runner

1. cog **never extracts or reuses subscription tokens** to call vendor APIs. It uses an official SDK, shells out to the unmodified official CLI, or uses an API key.
2. Credentials are environment variables (or the CLI's own login). cog never stores a subscription credential.
3. Each runner has a **live contract test** (`tests/live/`, skipped by default) run on the owner's machine: does an unattended run behave like an interactive one, with the intended shell, permission mode, and output format?
4. Pin CLI versions. Record the version in the run record.
5. Results are normalized into `AgentResult`. Unknown usage is `None`.
6. A subscription runner may be disabled by config at any time without code changes.

### Takeover

For human escalation, `cog takeover <run_id>` can open an interactive session in the task's worktree seeded with the task packet (for Copilot, its interactive mode; for Claude Code, an interactive `claude` session). Runners declare `takeover` in their capabilities. This is a fallback when the escalation ladder is exhausted, not a chat interface. Antigravity's equivalent is unverified.

---

## 4. Tiers and the escalation ladder

Config maps tiers to `(adapter, model)` pairs. Plans refer to tiers by name (`executor_tier`). The **escalation ladder** is the ordered list a failed task climbs before a human is involved.

```toml
# illustrative user config
[providers.fireworks]
kind = "openai_compat"
base_url = "https://api.fireworks.ai/inference/v1"
key_env = "FIREWORKS_API_KEY"

[providers.openrouter]
kind = "openai_compat"
base_url = "https://openrouter.ai/api/v1"
key_env = "OPENROUTER_API_KEY"
quirks = { require_parameters = true }

[tiers.small]
adapter = "fireworks"
model = "..."
[tiers.large]
adapter = "openrouter"
model = "..."
family = "..."      # lets config warn when the reviewer and planner share a family

[escalation]
ladder = ["small", "medium", "large"]
```

Workflows do not name tiers directly for completions. They request a **role**, and config resolves role → tier → `(adapter, model)`:

| Role        | Used by (Planning) | Default tier                                                                                                                         |
| ----------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `planner`   | Node 15            | large                                                                                                                                |
| `query_gen` | Node 11 (optional) | small                                                                                                                                |
| `verdict`   | Node 13 (optional) | small                                                                                                                                |
| `reviewer`  | Node 19 (optional) | medium, or an explicit different model, preferably from another family; config warns if it resolves to the planner's model or family |

A reviewer from another family is often another vendor, so the egress check ([Policy and sandbox](policy-and-sandbox.md), Section 8) applies to it like any provider.

`preflight` (node 1) validates that each role's tier exists, is reachable, and has the required capabilities.

---

## 5. Errors

Adapters classify failures so callers can react correctly.

| Class                 | Typical cause                                                | Default reaction                                      |
| --------------------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| `AuthError`           | Missing or invalid key or login                              | Stop; tell the user which variable or login           |
| `RateLimited`         | HTTP 429 with a retry hint                                   | Back off, retry within budget                         |
| `QuotaExceeded`       | Subscription or credit exhausted, possibly with a reset time | Stop or fall back to another tier; surface reset time |
| `Timeout`             | Slow call or hung process                                    | Retry once; kill the process tree                     |
| `MalformedOutput`     | Output fails schema validation                               | Repair loop, then status                              |
| `ContentFiltered`     | Provider refusal                                             | Surface; do not retry blindly                         |
| `ProviderUnavailable` | 5xx or network                                               | Retry with backoff, then fall back                    |
| `CapabilityMissing`   | Tier lacks a required feature                                | Fail at `preflight`                                   |

---

## 6. Budgets and cost control

- **Per-run and per-node caps** on tokens and dollars, set in config with conservative defaults.
- **Estimate before long runs** (Implementation especially) and show it for confirmation.
- **Kill switch:** `cog cancel <run_id>` stops at the next safe point.
- **Accounting:** every call records usage and, where known, cost into the event store. Unknown cost is `None`, and the run report says so. Prices come from a config table, not from memory.
- A budget breach is an **interrupt** (the human decides), not a silent stop.

---

## 7. Prompts

- Prompts are **files**, not strings in code, stored with each workflow and **versioned**. The prompt version id is recorded in every run record.
- Templates are rendered with a small, auto-escaping templating step. A single helper wraps **untrusted text** (issue bodies, comments, docs, code) in clear delimiters, and prompts state that delimited content is data, not instructions.
- **Per-model overrides** are allowed and explicit.
- **Neutrality:** prompts contain no code-style preferences. Any default is labeled overridable. Reviews of prompt changes check for smuggled style opinions ([ADR-0014](../decisions/0014-repo-convention-precedence.md)).
- Prompt changes are treated like code changes: snapshot-tested, and evaluated before merge when they affect recall or planning quality.

---

## 8. Data egress and secrets

- A policy decides which repos may be sent to which provider (for example, private repos only to approved providers).
- Context packs are scanned and **secrets redacted** before leaving the machine.
- OpenRouter provider preferences may restrict routing to providers with acceptable data policies. **(verify)**
- Keys never appear in config files, logs, or run records. Logs pass through a redaction filter.

---

## 9. Verification items for this page

This page's **(verify)** items are V12 to V20, V44, and V45 in the [roadmap's verification log](../roadmap.md#4-verification-log), which is the one list of record. Runner items are settled by live contract tests on the owner's machine.
