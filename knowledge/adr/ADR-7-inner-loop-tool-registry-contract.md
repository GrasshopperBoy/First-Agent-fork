# ADR-7 — Inner-loop and tool-registry contract for v0.1

- **Status:** accepted
- **Date:** 2026-05-12
- **Deciders:** project owner (`0oi9z7m1z8`), Devin (drafting)

## Context

ADR-1 commits v0.1 to UC1 (coding + PR write) and UC3
(local-docs-to-wiki). ADR-2 fixes static role routing, `tool_protocol:
native | prompt-only`, no in-loop Critic / Reflector, and an
MCP-shaped request / response convention. ADR-6 fixes a deny-by-default
filesystem sandbox. The missing contract is the **Coder inner-loop**:
how the model sees tools, how calls are validated and audited, and
which extension points exist before implementation starts.

The primary input is the active harness research note
[`efficient-llm-agent-harness-2026-05.md`](../research/efficient-llm-agent-harness-2026-05.md),
especially its §0 Decision Briefing, body recommendations, and §10
contract sketch. Secondary inputs are:

- `semi-autonomous-agents-cross-reference-2026-05.md` §7.1 (ACI
  signatures), §7.3 (edit formats), §8.4 (two-stage reads), and §8.5
  (hook rationale);
- `cutting-edge-agent-research-radar-2026-05.md` §0 and §1 (MCP/tool
  registry, ACI, hooks, traces, sandbox/audit).

This ADR intentionally stays within v0.1: single-agent loop,
in-process Python dispatcher, no external MCP server, no
code-execution-over-MCP, no always-on verifier loop.

## Options considered

### Option A — Minimal in-process MCP-shaped contract (chosen)

Keep tools as Python callables behind a registry. Expose compact
descriptors to the Coder, load JSON Schemas on demand, validate calls
deterministically, run ADR-6 sandbox checks as hooks, and append a raw
JSONL trace.

- Pros:
  - Reuses ADR-2's MCP-shaped convention without adding an MCP runtime.
  - Keeps routine Coder context below the 100k-token design budget.
  - Gives v0.2 a config-only path toward tool search and MCP servers.
  - Produces raw traces for future eval / self-evolution.
- Cons:
  - Requires a small registry, schema validator, dispatcher, and trace
    writer before the first Coder loop can run.

### Option B — Full MCP server topology in v0.1

Split repository, git, runner, and web tools into separate MCP servers.

- Pros: closest to the external ecosystem; clean process boundaries.
- Cons: adds process management, auth, transport, sandbox, and
  deployment surface before v0.1 has a baseline loop. Contradicts
  ADR-1's pragmatic single-user scope.

### Option C — Amp-style three tools with full-file reads

Expose only `read_file`, `list_files`, and `edit_file` with whole-file
payloads and string replacement.

- Pros: smallest first implementation.
- Cons: full-file reads create context bloat on 50k+ LoC repos; no
  line-windowed ACI; no structured trace; hard to evolve into tool
  search without changing the contract.

### Option D — Add Critic / verifier / multi-candidate loops now

Add a second model role that critiques each Coder action before tools
execute.

- Pros: familiar agent pattern.
- Cons: conflicts with ADR-2's explicit "no Critic / Reflector in
  v0.1"; adds prompt and token overhead before UC5 can measure whether
  it helps. The active research note records evidence that verifier
  loops can hurt agent-harness performance.

## Decision

We will choose **Option A**.

### Runtime loop

The v0.1 Coder loop is:

1. Assemble the static system prompt once at session start and keep it
   frozen for the session.
2. Load dynamic task / repository context through pointers in user
   messages, not by rebuilding the system prompt each turn.
3. Expose compact tool descriptors: stable `name`, one-line
   `description`, and `tags`. Full JSON Schema is loaded on demand.
4. Receive a model response.
5. If it contains a tool call, validate `params` against the tool's
   input schema.
6. Run `pre_tool` hooks: ADR-6 sandbox, tool-group allow-list, and any
   deterministic validation.
7. If any `pre_tool` hook modifies `params`, re-run the same JSON
   Schema validation before handler execution.
8. Execute the handler.
9. Run `post_tool` hooks: normalize result / error, write audit event,
   and persist large artifacts to the filesystem.
10. Return only compact summary plus artifact paths to the model.
11. Stop on final answer, max iterations, hard error, or user approval
    gate.

### Tool disclosure tiers

The registry has three disclosure tiers:

| Tier | Model sees | Use |
|---|---|---|
| 1 | Tool-group or server labels | Session-start routing and future tool search |
| 2 | `name`, one-line `description`, `tags` | Normal Coder prompt |
| 3 | Full JSON Schema | Loaded only for tools the model may call |

If the v0.1 catalog grows beyond roughly ten tools, future tool search
must reuse ADR-4 SQLite FTS5 through a `tools` virtual table. No new
BM25 or vector dependency is introduced by this ADR.

The initial tool registry is limited to coding / repo-local operations:

- `repo.list`
- `repo.search`
- `repo.read`
- `repo.edit_file`
- `repo.write_file`
- `repo.apply_patch`
- `git.status`
- `git.diff`

`repo.list` is the bounded directory-listing tool; `repo.write_file` is
the explicit new-file path. `repo.edit_file` remains the default for
localized replacements, and `repo.apply_patch` remains the atomic
multi-edit path. `git.commit` can land once commit-message trailers are
implemented. `web.*`, `pdf.*`, and `run_command` are not in the default
v0.1 Coder tool group.

### ToolSpec

Each registry entry has this shape:

```yaml
name: repo.read
description: Read a bounded line window from an allowed path.
input_schema: {}        # JSON Schema, loaded on demand
output_schema: {}       # optional JSON Schema
permission: read        # read | workspace | full (full reserved post-v0.1)
tags: [coding]          # one or more; used by tool-group pre_tool hook
handler: <callable>     # deterministic Python callable, not model-visible
defer_loading: true     # hint for future tool search / MCP export
```

`name` is a stable dotted string. Renaming a public tool after
implementation requires either a backwards-compatible alias or a new ADR
/ amendment, because traces and prompts depend on it.

`permission` is the tool-level capability class, not a replacement for
ADR-6 path policy:

| Permission | Meaning | ADR-6 relation |
|---|---|---|
| `read` | May only read sandbox-allowed paths or return metadata. | Requires `[read]` allow-list pass for filesystem reads. |
| `workspace` | May mutate sandbox-allowed workspace / repo paths. | Requires `[write]` allow-list pass for every write target. |
| `full` | Reserved for future privileged tools. | Not allowed in v0.1 unless a later ADR/amendment defines its sandbox. |

The v0.1 default is `read` or `workspace`. `full` exists only as an
explicit registry value so implementers cannot smuggle privileged tools
behind an underspecified boolean.

### Request / response shape

The dispatcher inherits ADR-2's MCP-shaped convention:

```json
{
  "request": {"name": "repo.read", "params": {"path": "README.md"}},
  "response": {"result": {}, "error": null}
}
```

Tool results use:

```yaml
result: {}              # compact structured payload or null
error: null             # {code, message} or null
summary: ""             # short model-facing text
artifacts: []           # paths to raw logs, diffs, or large payloads
```

Large outputs stay in `artifacts`; the model receives bounded summaries
and paths. Empty output is explicit, not silent. A `retryable` error
hint is deferred until ADR-2 explicitly amends the MCP-shaped error
schema.

### ACI and edit formats

Coder-facing repository tools must prefer windowed and bounded APIs:

- `repo.list(path, depth?, glob?) -> bounded directory entries`
- `repo.search(query, paths?, regex?) -> matches with line ranges`
- `repo.read(path, start_line, end_line) -> bounded text window`
- `repo.edit_file(path, old_string, new_string) -> single replacement`
- `repo.write_file(path, contents) -> create a new file`
- `repo.apply_patch(unified_diff) -> atomic multi-edit`

`edit_file` is the simple default for one localized replacement.
`apply_patch` is the default for multiple edits or cross-file diffs and
must be checked with `git apply --check` or an equivalent parser before
execution. The model-comparison fixture from `HANDOFF.md` may tune
prompting guidance, but the contract supports both shapes.

### Hooks

v0.1 defines only:

- `pre_tool(event) -> allow | deny | modify_params`
- `post_tool(event) -> ToolResult`

ADR-6 sandbox checks and JSON Schema validation run before handler
execution. If a `pre_tool` hook returns `modify_params`, the dispatcher
must validate the modified params against the same input schema before
executing the handler. Audit normalization and artifact persistence run
after handler execution. `pre_run`, `post_run`, `on_event`, human
approval UI, and long-lived policy hooks are deferred to v0.2.

### Trace separation

Each run writes:

```text
~/.fa/state/runs/<run_id>/events.jsonl
~/.fa/state/runs/<run_id>/artifacts/*
```

Every event includes at minimum `ts`, `actor`, `kind`, `content`,
`tool_name?`, `tool_call_id?`, `parent_event_id?`, and `harness_id`.
`events.jsonl` is append-only and is the canonical source for replay,
eval, and future self-evolution. Human-readable summaries such as
`hot.md` may cite trace paths and offsets, but must not replace the raw
trace.

### Explicit deferrals

- No external MCP server dependency in v0.1.
- No code-execution-over-MCP in v0.1.
- No `run_command` tool until ADR-6's OS-level sandbox trigger is
  revisited.
- No always-on Critic, Reflector, verifier, or multi-candidate loop.
- No web tool group by default.

## Consequences

- **Positive:** ADR-7 closes the contract needed by the first Coder
  loop without expanding v0.1 scope.
- **Positive:** Tool definitions stay progressively disclosed, so
  routine calls avoid full schema injection and fit the 100k-token
  design budget.
- **Positive:** Tool calls are replayable and auditable through raw
  traces, not only lossy summaries.
- **Positive:** The same shape can later export to MCP servers or tool
  search without changing model-facing names.
- **Negative:** Registry, schema validation, hook dispatch, and trace
  writing become required infrastructure before the Coder loop.
- **Negative:** Prompt-only models remain a supported ADR-2 shape, but
  this ADR mostly specifies the native-tool path. Prompt-only adapters
  must translate to the same internal `request` / `response` shape.

## Acceptance

Before the implementation PR for this contract merges:

1. `ToolSpec` loader rejects tools with missing `name`,
   `description`, `input_schema`, `permission`, `tags`, or handler.
2. JSON Schema validation rejects malformed params before handler
   execution.
3. ADR-6 path and tool-group checks run before any filesystem I/O.
4. Tool results with large payloads return artifact paths, not raw
   unbounded output.
5. `events.jsonl` records both successful and failed tool calls.
6. Repository reads are line-windowed or byte-bounded.
7. Edit tools expose both single-replacement and unified-diff shapes.
8. The implementation PR answers the minimalism-first self-audit:
   - What is in the agent's context window that does not need to be
     there?
   - Which tools does the agent rarely use over the latest traces?
   - Are verification or search loops hurting performance?
   - Is the control logic written in Python code or in text, and which
     would be cheaper to change?

## References

- [ADR-1 — v0.1 use-case scope](./ADR-1-v01-use-case-scope.md)
- [ADR-2 — LLM tiering & access](./ADR-2-llm-tiering.md)
- [ADR-4 — Storage backend for v0.1](./ADR-4-storage-backend.md)
- [ADR-6 — Tool sandbox & path allow-list policy](./ADR-6-tool-sandbox-allow-list.md)
- [Efficient LLM agent harness — ADR-7 prep](../research/efficient-llm-agent-harness-2026-05.md)
- [Semi-autonomous LLM agents cross-reference](../research/semi-autonomous-agents-cross-reference-2026-05.md)
- [Cutting-edge agent research radar](../research/cutting-edge-agent-research-radar-2026-05.md)
