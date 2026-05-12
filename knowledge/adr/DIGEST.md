# ADR Digest — one-paragraph cheat-sheet

> **Purpose.** Cheat-sheet for agents and humans who need the gist of
> all accepted ADRs without reading the full source set. One
> paragraph per ADR + bulleted amendments. **The per-ADR file is the
> authoritative source** — this digest only paraphrases.
>
> **Maintenance rule.** When an ADR amendment lands, update the
> matching row here in the **same PR**. Per
> [AGENTS.md PR Checklist rule #9](../../AGENTS.md#pr-checklist).
> Stale rows defeat the purpose.

## ADR-1 — v0.1 use-case scope (accepted 2026-04-27)

**Decision.** Ship UC1 (coding + PR end-to-end) and UC3 (local docs
to wiki) in v0.1; UC2 (multi-source research) best-effort via
LLM-fan-out on top-k chunks; UC4 (Telegram multi-user) deferred to
v0.2 entirely. **Rationale.** User's verbatim ranking puts UC1 + UC3
first; UC1 acceptance scenario (folder → search → edit → PR) is
mechanically verifiable, UC3 is the simpler half-step (no PR-write,
no allow-lists). UC4 needs per-user namespacing that does not exist.

**Amendments.**

- **2026-05-01** — UC5 (semi-autonomous multi-LLM research / experiment)
  added to deferred list.
- **2026-05-06** — UC5 expanded to eval-driven harness iteration
  (5a benchmark suite → 5b trace schema → 5c agent self-modification
  via Skills → 5d re-benchmark → 5e leaderboard); makes Pillar 3
  efficient-harness claim measurable.

**Source:** [`ADR-1`](./ADR-1-v01-use-case-scope.md).

## ADR-2 — LLM tiering & access (accepted 2026-04-27)

**Decision.** Static role routing — Planner = top-tier OSS,
Coder = mid-tier OSS, Debug = elite (Claude), Eval = top-tier OSS
pinned. Configuration in `~/.fa/models.yaml`. **No cross-tier
auto-escalation** on failure; Coder fails loudly, user retries.
**Rationale.** Predictable cost, predictable behavior, simple to
debug; cross-tier auto-escalation is a research problem unsuited
for v0.1.

**Amendments.**

- **2026-04-29** — `tool_protocol: native | prompt-only` per role;
  v0.1 inner-loop has **no Critic / Reflector** role (kept as
  `intra-role retry` only).
- **2026-05-01** — MCP forward-compat tool-shape convention: in-process
  tool dispatcher mirrors JSON-RPC `request: {name, params}` /
  `response: {result, error}`. **No `mcp` package dependency in v0.1.**

**Source:** [`ADR-2`](./ADR-2-llm-tiering.md).

## ADR-3 — Memory architecture variant for v0.1 (accepted 2026-04-27)

**Decision.** Variant A "Mechanical Wiki" — filesystem-canonical
Markdown + YAML frontmatter; deterministic Python chunker; SQLite
FTS5 BM25 read-side. **No embeddings, no graph, no Mem0-style
volatile store in v0.1.** Volatile-store hooks (`src/fa/memory/volatile/`)
scaffolded but empty — additive in v0.2. **Rationale.** Smallest
LoC + smallest dependency surface (~600 LoC + sqlite stdlib);
aligns with ADR-1 scope; v0.2 hooks are additive, not a migration.

**Amendments.** None.

**Source:** [`ADR-3`](./ADR-3-memory-architecture-variant.md).

## ADR-4 — Storage backend for v0.1 (accepted 2026-04-27)

**Decision.** SQLite FTS5 at `~/.fa/state/index.sqlite`
(config-overridable). External-content FTS5 over `chunks` table;
tokeniser `unicode61 remove_diacritics 2` + porter stemmer. **No
vector store in v0.1** (v0.2 ADR slot reserved for `sqlite-vec` or
separate `embeddings.sqlite`). **Rationale.** Zero extra runtime
deps (sqlite3 in stdlib); persistent + incremental upserts; BM25
ranking built-in; index is disposable cache.

**Amendments.**

- **2026-04-29** — `chunks` schema gains seven columns
  (`parent_title`, `breadcrumb`, `line_start/_end`, `byte_start/_end`,
  `topic`); migration `0002_provenance_columns.sql`. Mirrors ADR-5
  `Chunk` dataclass extension same date.

**Source:** [`ADR-4`](./ADR-4-storage-backend.md).

## ADR-5 — Chunker tool selection for v0.1 (accepted 2026-04-28)

**Decision.** universal-ctags (code) + markdown-it-py (prose),
combined behind a stable `Chunker.chunk_file(path) -> list[Chunk]`
interface. Pipeline per-extension: Markdown → AST split by H1/H2;
source code → ctags JSON, slice by line-range; config files →
one-file-per-chunk; catch-all → one-file. **tree-sitter is
explicit non-goal** with named re-evaluation triggers.
**Rationale.** Smallest dependency surface (~6 MB system + 1 MB
Python wheel); covers all six target languages including
PowerShell; loud failure modes.

**Amendments.**

- **2026-04-29** — `Chunk` dataclass extended with `parent_title`,
  `breadcrumb`, `byte_start/_end`, `topic`; `CHUNKER_VERSION` bumps
  invalidate cache once on first run.

**Source:** [`ADR-5`](./ADR-5-chunker-tool.md).

## ADR-6 — Tool sandbox & path allow-list policy (accepted 2026-04-29)

**Decision.** Path allow-list at `~/.fa/sandbox.toml` with separate
`[read]` / `[write]` blocks; default-deny; deny overrides allow;
`pathlib.Path.resolve(strict=False)` collapses symlinks; gitignore-style
globs via `pathspec`. Audit log to `~/.fa/state/sandbox.jsonl`.
**Single resolution per tool invocation** prevents TOCTOU. **v0.1
ships no `run_command` tool.** **Rationale.** Mid-tier Coder
hallucinates; reads are de-facto network egress for ~99% remote-API
config; path-level guard is loud, fast, stoppable; symmetric to
`~/.fa/repos.toml` PR-write allow-list.

**Amendments.** None.

**Source:** [`ADR-6`](./ADR-6-tool-sandbox-allow-list.md).

## ADR-7 — Inner-loop and tool-registry contract for v0.1 (accepted 2026-05-12)

**Decision.** v0.1 uses a minimal in-process Coder loop with an
MCP-shaped registry, not external MCP servers. Tools are Python
callables behind `ToolSpec` entries (`name`, one-line `description`,
JSON Schema, `permission`, `tags`, handler, `defer_loading`), disclosed
progressively: group labels → compact descriptors → full schema on
demand. Tool calls validate JSON Schema, run `pre_tool` hooks (ADR-6
sandbox + `tool_groups` + deterministic checks), execute, run
`post_tool` audit / artifact persistence, and append raw events to
`~/.fa/state/runs/<run_id>/events.jsonl`. Repository APIs are
windowed/bounded (`repo.search`, `repo.read`) and expose both
`edit_file(old_string, new_string)` and `apply_patch(unified_diff)`.
No Critic / Reflector, external MCP runtime, code-execution-over-MCP,
or `run_command` ships in v0.1. **Rationale.** Reuses ADR-2 / ADR-6,
keeps routine context below the 100k-token design budget, preserves raw
traces for eval, and leaves v0.2 tool-search / MCP export config-shaped.

**Amendments.** None.

**Source:** [`ADR-7`](./ADR-7-inner-loop-tool-registry-contract.md).

## See also

- [`README.md`](./README.md) — ADR process and ordered index.
- [`../trace/exploration_log.md`](../trace/exploration_log.md) — alternatives that were rejected at decision time + lessons (per ADR).
- [`../project-overview.md` §1.1](../project-overview.md#11-четыре-столпа-цели-project-goal--four-pillars) — four-pillar project goal that all ADR decisions advance.
