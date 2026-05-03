---
title: "Cutting-edge agent research radar — May 2026"
source:
  - "https://modelcontextprotocol.io/specification/2025-03-26/server/tools"
  - "https://www.anthropic.com/index/building-effective-agents"
  - "https://docs.anthropic.com/en/docs/claude-code/hooks"
  - "https://swe-agent.com/latest/background/aci/"
  - "https://openai.github.io/openai-agents-python"
  - "https://platform.openai.com/docs/guides/agent-evals"
  - "https://docs.letta.com/guides/agents/memory/"
  - "https://www.gitworktree.org/ai-tools/parallel-agents"
  - "./semi-autonomous-agents-cross-reference-2026-05.md"
  - "./cross-reference-ampcode-sliders-to-adr-2026-04.md"
  - "../adr/ADR-2-llm-tiering.md"
  - "../adr/ADR-6-tool-sandbox-allow-list.md"
compiled: "2026-05-03"
chain_of_custody: "Primary-source claims are quoted or paraphrased from the URLs in `source`; First-Agent mapping is derived from ADR-2, ADR-6, and existing cross-reference notes listed above."
claims_requiring_verification:
  - "Provider documentation and product names evolve quickly; re-check tool, hook, eval, and memory APIs before writing an implementation ADR."
  - "The recommendations here are a radar/backlog, not accepted architecture decisions."
superseded_by: ""
---

# Cutting-edge agent research radar — May 2026

> **Status:** research radar / backlog input. Not an ADR.
>
> Purpose: capture current external patterns that look relevant to
> First-Agent v0.1/v0.2, without starting implementation or touching
> the chunker workstream.

## 0. Executive summary

First-Agent already made several good bets:

- filesystem-canonical memory instead of opaque vector-first state;
- static role routing instead of unpredictable auto-escalation;
- MCP-shaped tool calls without taking an MCP dependency in v0.1;
- path allow-list sandbox before any filesystem-touching Coder loop;
- small PRs and explicit agent coordination.

The radar below updates that direction against current agent-tooling
signals:

1. **MCP/tool registry** is still the right compatibility target, but
   v0.1 should pin only the in-process shape: stable tool names,
   JSON Schema inputs, structured results, structured errors, and
   audit metadata. Avoid real MCP server/process complexity until v0.2.
2. **ACI design matters more than raw shell access.** SWE-agent-style
   findings point toward windowed file views, search-first tools,
   syntactic checks around edits, and explicit empty-output messages.
3. **Hooks are the cleanest way to compose sandbox, validation, audit,
   and future HITL.** The v0.1 shape should stay tiny:
   `pre_tool` + `post_tool`.
4. **Agent memory should remain filesystem-first.** Letta/MemGPT-style
   stateful agents validate the importance of persistent state, but
   First-Agent should not import a memory server before the Mechanical
   Wiki baseline is working.
5. **Eval should start from traces and tiny datasets.** Do not wait for
   a large benchmark. Each Phase M module should emit enough trace-like
   data to later grade tool choice, sandbox denials, and edit success.
6. **Sandbox/tool audit is not optional.** MCP and modern agent SDK docs
   converge on input validation, access controls, timeouts, logs, and
   user confirmation for sensitive operations.
7. **Multi-agent coordination is mostly workspace isolation + file
   ownership.** Forks/worktrees are implementation details; the rule is
   one agent, one branch/workspace, one PR target.

Recommended next PRs:

1. ADR-7 prep note: inner-loop + tool contract.
2. Glossary update: MCP, Hook, ACI, Trace, Guardrail, Handoff.
3. Small eval-fixture note: what first 5 trace/eval cases should be.
4. Optional playbook: "launch two non-overlapping First-Agent agents".

---

## 1. Radar table

| Area | External signal | First-Agent fit | Action |
|---|---|---|---|
| MCP/tool registry | MCP tools expose `name`, `description`, `inputSchema`; calls use `tools/call`; results can set `isError`. | Matches ADR-2 MCP-shaped convention. | Feed into ADR-7 contract. |
| ACI | SWE-agent emphasizes custom file viewer, concise search, edit-time lint, and explicit empty-output messages. | Strong fit for UC1 and UC3 with mid-tier Coder. | Use as tool-surface constraints. |
| Hooks | Claude Code exposes lifecycle hooks, including `PreToolUse` / `PostToolUse`, with JSON input/output and deny decisions. | Maps to ADR-6 sandbox and future audit log. | Use only `pre_tool` / `post_tool` in v0.1. |
| Memory | Letta stores state, messages, tool calls, and editable memory blocks. | Confirms persistent state need; too heavy for v0.1. | Keep Mechanical Wiki; revisit v0.2 volatile store. |
| Eval | OpenAI agent evals emphasize traces first, then datasets/eval runs. | Fits small Phase M modules. | Add trace schema before large eval suite. |
| Sandbox/audit | MCP security notes require validation, access control, rate limits, output sanitization, timeouts, audit logs. | Reinforces ADR-6. | ADR-7 must include structured errors and audit rows. |
| Multi-agent | Worktree/fork guidance says isolated workspace per agent prevents write/read/build interference. | Matches current two-fork workflow. | Keep ownership declarations before work starts. |

---

## 2. MCP / tool registry

### 2.1 What the sources say

The MCP tools spec defines tools as model-invoked capabilities with:

- unique `name`;
- human-readable `description`;
- `inputSchema` using JSON Schema;
- `tools/list` for discovery;
- `tools/call` for invocation;
- structured result content;
- `isError` for tool execution errors;
- JSON-RPC protocol errors for invalid tools / invalid arguments.

It also says tools are model-controlled, but applications should keep
human-in-the-loop affordances for safety-sensitive invocations.

MCP security guidance in the same spec says servers must validate tool
inputs, implement access controls, rate-limit invocations, and sanitize
outputs; clients should show inputs for sensitive operations, validate
tool results, implement timeouts, and log usage for audit.

### 2.2 First-Agent mapping

ADR-2 already made the right zero-cost move:

```text
request:  { name: str, params: dict[str, Any] }
response: { result: Any | None, error: { code: int, message: str } | None }
```

The radar update is: ADR-7 should align more closely with MCP's public
tool vocabulary:

- `name` — stable tool string, e.g. `repo.read`, `repo.search`;
- `description` — short model-facing description;
- `input_schema` — JSON Schema;
- `annotations` — optional v0.2 field, not needed in v0.1;
- `result.content` — text-first result payload;
- `is_error` or `error` — one canonical error surface.

### 2.3 Recommendation

For v0.1, avoid a real MCP dependency. Implement an in-process
`ToolRegistry` with MCP-compatible shape:

```text
ToolSpec
  name: str
  description: str
  input_schema: dict
  handler: Callable

ToolRequest
  name: str
  params: dict

ToolResult
  result: object | None
  error: ToolError | None
  summary: str
```

ADR-7 should explicitly say:

- transport is out of scope;
- external MCP servers are out of scope;
- internal shape is MCP-compatible to avoid redesign later.

### 2.4 Open question for ADR-7

MCP distinguishes protocol errors from tool execution errors. First-Agent
needs one local convention:

- unknown tool / schema validation failure = protocol-ish error;
- sandbox denial / grep no match / command failed = execution error.

The exact Python exception hierarchy can be small, but it should be
specified before implementation.

---

## 3. Agent-Computer Interface (ACI)

### 3.1 What the sources say

SWE-agent frames ACI as the tool/interface layer an agent uses to work
with a computer environment. Its docs highlight five practical findings:

1. run a linter when an edit command is issued and block syntactically
   invalid edits;
2. use a purpose-built file viewer instead of raw `cat`;
3. show small file windows, around 100 lines per turn;
4. make repository search concise, often listing files with matches
   instead of dumping every matching line;
5. say explicitly when a command succeeds with empty output.

The key lesson is not "copy SWE-agent"; it is that LLM agents are a new
kind of user and need interfaces shaped for their failure modes.

### 3.2 First-Agent mapping

First-Agent's UC1/UC3 constraints make ACI more important than for
Claude-only systems:

- Coder is mid-tier OSS by design;
- full-file reads waste the context window;
- the Mechanical Wiki chunker can provide stable windows;
- ADR-6 treats reads as network egress when remote LLMs are used;
- sandbox denials must be understandable enough for the model to recover.

### 3.3 Recommendation

ADR-7 should define tools as ACI primitives, not thin shell wrappers:

```text
repo.search(pattern, path, limit) -> list[SearchHitSummary]
repo.read(path, start_line, end_line) -> FileWindow
repo.write_patch(path, patch) -> EditResult
repo.status() -> RepoStatus
```

Avoid for v0.1:

- unrestricted `run_command`;
- full-file `read_file` as the default path;
- raw `cat` / `grep` dumps as model-facing results;
- editing commands that do not run syntax / formatting checks.

### 3.4 Open question for ADR-7

Should `repo.read` always require a window, or allow `path`-only for
small files?

Suggested v0.1 compromise:

- files below a small byte threshold can be returned whole;
- larger files require `start_line` / `end_line` or chunk anchor;
- every result includes `truncated: true | false`.

---

## 4. Inner-loop hooks

### 4.1 What the sources say

Claude Code's hook reference exposes lifecycle events such as
`SessionStart`, `PreToolUse`, `PostToolUse`, `Stop`, `PreCompact`, and
others. Hooks receive JSON context and can return decisions. A
`PreToolUse` hook can deny a dangerous shell command and surface a
reason back to the agent.

This matches the earlier First-Agent cross-reference note: hooks are
valuable when they separate cross-cutting concerns from tool bodies.

### 4.2 First-Agent mapping

ADR-6 sandbox is already a pre-tool hook conceptually:

```text
ToolRequest -> sandbox check -> allow or structured error
```

Audit logging is a post-tool hook:

```text
ToolRequest + ToolResult + duration -> JSONL audit row
```

HITL confirmation is a future pre-tool hook:

```text
dangerous write / push -> ask user -> allow or deny
```

### 4.3 Recommendation

v0.1 should implement only:

- `pre_tool(request, context) -> HookDecision`;
- `post_tool(request, result, context) -> None`.

Do not implement:

- `pre_run`;
- `post_run`;
- `on_event`;
- async hooks;
- HTTP hooks;
- plugin systems.

The minimal hook chain should be ordered:

1. schema validation;
2. sandbox/path policy;
3. optional one-shot bypass policy;
4. tool execution;
5. audit log;
6. result summarization.

### 4.4 Open question for ADR-7

Should schema validation be a hook or part of the dispatcher?

Suggested answer: dispatcher core. Hooks should receive typed/validated
requests. That keeps hook code simple and prevents every hook from
re-implementing JSON Schema checks.

---

## 5. Agent memory

### 5.1 What the sources say

Letta's memory docs describe stateful agents where system prompts,
memory blocks, messages, reasoning, and tool calls persist in a
database. Important core memories are injected into context, and agents
can modify their own memories via tools.

This is close to the MemGPT idea: context window as active memory, with
larger persistent tiers outside context.

### 5.2 First-Agent mapping

First-Agent already has the safer v0.1 baseline:

- Markdown files are source of truth;
- ADR-3 rejects vector/graph complexity for v0.1;
- ADR-4 uses SQLite FTS5 for search;
- `hot.md` / session archives are intended as audit trail;
- future volatile-store hooks are reserved, not implemented.

The radar update is: do not import a memory platform yet. The patterns
are relevant, but the implementation would fight the filesystem-canon
principle if adopted too early.

### 5.3 Recommendation

For v0.1:

- keep memory writes explicit and reviewable;
- write durable notes as Markdown;
- store tool/session traces as JSONL under `~/.fa/state/`;
- let the Mechanical Wiki index read those files later.

For v0.2:

- consider "memory blocks" as frontmatter-addressable Markdown sections;
- consider shared blocks only after multi-user scope returns;
- consider a volatile store only for low-stakes, evictable facts.

### 5.4 Open question for v0.2

Should an agent be allowed to edit its own durable memory without PR
review?

Suggested answer for now: no. It may propose memory edits, but durable
project memory still lands through PR review.

---

## 6. Eval harness and traces

### 6.1 What the sources say

OpenAI's agent-evals docs recommend starting with traces while debugging
behavior. A trace captures model calls, tool calls, guardrails, and
handoffs for a run. Trace grading answers questions such as:

- Did the agent pick the right tool?
- Did a handoff happen when it should?
- Did the workflow violate an instruction or safety policy?
- Did a prompt or routing change improve end-to-end behavior?

Then, when behavior is better understood, move to repeatable datasets
and eval runs.

The OpenAI Agents SDK also treats tracing as a first-class feature,
alongside tools, handoffs, guardrails, sessions, and MCP tool calling.

### 6.2 First-Agent mapping

First-Agent does not need a hosted eval product to copy the useful
shape. It needs a local trace schema early enough that Phase M modules
can emit data before the eval suite exists.

### 6.3 Recommendation

Define a local trace/audit row family:

```json
{
  "session_id": "2026-05-03T...",
  "step": 12,
  "role": "coder",
  "event": "tool_call",
  "tool": "repo.read",
  "args_hash": "sha256:...",
  "result_summary": "returned 80 lines",
  "is_error": false,
  "duration_ms": 42
}
```

Then add tiny eval fixtures:

1. tool selection: search before read on large file;
2. sandbox: denied read returns recoverable structured error;
3. edit: patch changes exactly one intended file;
4. chunker: markdown and code chunks preserve provenance;
5. role routing: Coder failure does not auto-escalate to Debug.

### 6.4 Open question for ADR-7

Does trace logging belong in ADR-7 or a separate Eval ADR?

Suggested answer: ADR-7 owns the raw event schema because it is emitted
by the loop. A later Eval ADR owns graders/datasets.

---

## 7. Sandbox and tool audit

### 7.1 What the sources say

MCP tool security guidance says:

- validate all tool inputs;
- implement access controls;
- rate-limit tool invocations;
- sanitize tool outputs;
- prompt for confirmation on sensitive operations;
- validate tool results before passing them back to the LLM;
- implement timeouts;
- log tool usage for audit.

OpenAI Agents SDK docs list guardrails, sandbox agents, human-in-the-loop
mechanisms, and tracing as core runtime primitives.

Claude Code hooks demonstrate deterministic pre-tool blocking with a
structured denial reason.

### 7.2 First-Agent mapping

ADR-6 covers path access control well, but ADR-7 must connect the policy
to the actual loop:

- where schema validation runs;
- what a sandbox denial looks like;
- how denials are visible to the model;
- what gets logged;
- whether logs include raw paths or hashes;
- which operations require human confirmation.

### 7.3 Recommendation

ADR-7 should define:

- structured `ToolError`;
- audit JSONL path;
- path redaction/hash policy;
- max tool duration / timeout policy;
- recoverable vs fatal errors;
- one-shot bypass flow from ADR-6.

Suggested `ToolError` shape:

```json
{
  "code": "sandbox.denied",
  "message": "Read denied by sandbox policy",
  "recoverable": true,
  "hint": "Use repo.search within an allowed repository"
}
```

### 7.4 Open question for ADR-7

Should audit logs store raw paths?

Suggested v0.1 answer:

- store raw paths for allowed project files;
- redact/omit denied sensitive-looking paths;
- always store `args_hash` for correlation.

---

## 8. Multi-agent coordination

### 8.1 What the sources say

Git worktree guidance for parallel agents emphasizes isolated workspaces:
multiple agents in the same directory can overwrite each other's files,
read half-written changes, or collide on builds and ports. Worktrees give
each agent a separate directory and branch while sharing the git object
store.

The same logic applies to separate forks: isolation matters more than
the UI's fork-chain display.

### 8.2 First-Agent mapping

Current project workflow:

```text
GITcrassuskey-shop/First-Agent:main
├─ GrasshopperBoy/First-Agent-fork:<branch>
└─ MondayInRussian/First-Agent-fork2:<branch>
```

The important invariants:

- each agent starts from upstream `main`;
- each agent owns a disjoint file set;
- each agent opens a direct PR to the main repo target;
- dependent PRs document merge order;
- no agent merges through another agent's fork as routine workflow.

### 8.3 Recommendation

Keep this as the project-level default:

1. Assign ownership before agent launch.
2. Give each agent a standalone prompt.
3. Require each agent to list intended files before editing.
4. Use separate branches/forks/worktrees.
5. Use direct PRs to `GITcrassuskey-shop/First-Agent:main`.
6. If stacked, write `Recommended merge order: PR A → PR B`.

### 8.4 Open question for v0.2

Should First-Agent eventually include a "dispatcher" that launches
parallel child agents?

Suggested answer: defer. Current scope explicitly defers UC5. The useful
part today is the human-readable coordination protocol.

---

## 9. Candidate backlog

### 9.1 v0.1 hard requirements

1. **ADR-7 prep note / ADR draft**
   - Tool registry.
   - Inner-loop pseudocode.
   - MCP-shaped request/result/error.
   - `native` vs `prompt-only` handling.
   - `pre_tool` / `post_tool`.
   - audit trace schema.
   - cancellation / hot.md flush.

2. **Tool trace schema**
   - Minimal JSONL rows.
   - Required event types.
   - Redaction policy.
   - Test fixture for one denied sandbox read.

3. **ACI tool surface**
   - `repo.search`.
   - `repo.read` window.
   - `repo.write_patch` or equivalent.
   - no unrestricted shell in v0.1.

### 9.2 v0.1 nice-to-have

1. **Glossary update**
   - MCP.
   - ACI.
   - Hook.
   - Trace.
   - Guardrail.
   - Handoff.

2. **Multi-agent playbook**
   - Copy-paste prompt.
   - File ownership checklist.
   - Direct upstream PR rule.

3. **Five-case eval seed**
   - small enough to run manually;
   - covers tool choice, sandbox, edit, chunk, role routing.

### 9.3 v0.2 candidates

1. Real MCP server mode for selected tools.
2. Rich hook lifecycle beyond `pre_tool` / `post_tool`.
3. Volatile memory blocks or self-managed memory.
4. Parallel child-agent dispatcher.
5. Hosted/remote trace viewer.
6. Graph/vector layer after Mechanical Wiki proves insufficient.

---

## 10. Explicit non-goals

This note does **not** recommend:

- adding a real MCP dependency in v0.1;
- adopting OpenAI Agents SDK, Letta, Claude Code hooks, or SWE-agent
  wholesale;
- implementing multi-agent execution in v0.1;
- replacing Markdown source-of-truth with a memory database;
- adding unrestricted shell access;
- changing ADR-1 scope;
- changing the chunker implementation plan owned by the parallel agent.

---

## 11. Follow-up prompt for ADR-7 prep

Use this after the radar PR lands:

```text
Write an ADR-7 prep note for First-Agent: inner-loop + tool contract.

Inputs:
- knowledge/research/cutting-edge-agent-research-radar-2026-05.md
- knowledge/research/semi-autonomous-agents-cross-reference-2026-05.md
- knowledge/research/cross-reference-ampcode-sliders-to-adr-2026-04.md §10 R-1
- knowledge/adr/ADR-2-llm-tiering.md amendments 2026-04-29 and 2026-05-01
- knowledge/adr/ADR-6-tool-sandbox-allow-list.md

Output:
- A focused research/pre-ADR note, not the final ADR.
- Must specify candidate decisions, open questions, and minimal v0.1 scope.
- Do not touch chunker implementation files.
```

---

## 12. Bottom line

The cutting-edge signal is not "use a bigger framework". It is:

- keep the loop simple;
- make tools schema-first;
- make file access windowed and searchable;
- place sandbox/validation before tool execution;
- log every tool decision;
- evaluate from traces;
- isolate parallel agents by workspace and file ownership.

That is strongly aligned with the existing First-Agent ADR direction.
