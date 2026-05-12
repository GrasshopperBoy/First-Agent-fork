# Exploration log — alternatives rejected at decision time

> One block per accepted ADR. Each block lists the question, the
> chosen option (`Chosen:`), and each rejected option with
> `Reason:` (why rejected at decision time) + `Lesson:` (what new
> evidence would re-open the branch). Cross-question coupling
> noted under `Coupling:`. The ADR file is the authoritative
> source — this log is the cheap-read overlay agents use to learn
> *why* alternatives were rejected without re-reading every ADR
> end-to-end.
>
> **Maintenance rule.** When an ADR PR lands, append (or amend)
> the matching block in the same PR. Per
> [AGENTS.md PR Checklist rule #9](../../AGENTS.md#pr-checklist).
>
> Origin: research note
> [`ara-protocol-cross-reference-2026-05.md`](../research/ara-protocol-cross-reference-2026-05.md)
> §9 R-1, converted from YAML DAG to telegraphic markdown
> 2026-05-10 per Tsinghua NLAH finding (code → NL migration:
> +16.8 pp accuracy, 9× faster, 97% fewer LLM calls on
> `arXiv:2603.25723`).

## Q-1 — Which v0.1 use cases ship end-to-end? (2026-04-27)

- **Closed by:** [ADR-1](../adr/ADR-1-v01-use-case-scope.md)
- **Chosen:** Ship UC1 + UC3 in v0.1; defer UC4; UC2 best-effort.
- **Rejected:**
  - **UC1 + UC4 in v0.1.** Reason: UC4 ranked below UC1 + UC3 by
    lead; pulls Mem0-style volatile store from v0.2. Lesson:
    becomes viable once volatile-memory design (per-user
    namespacing) is independently justified — see Q-3 dead-end on
    Variant B.
  - **All four use cases in v0.1.** Reason: contradicts pragmatic
    medium-weight goal; sprawling scope. Lesson: revisit when v0.1
    ships and team / infra has grown past single-user.

### Q-1 amendment 2026-05-01 — UC5 (multi-LLM eval-harness) added to v0.1 deferred list

- **Coupling:** Q-1.
- **Rationale:** Needs multi-LLM runner + comparison template;
  closer to eval-harness than orchestration.
- **Source:** [ADR-1 §Amendment 2026-05-01](../adr/ADR-1-v01-use-case-scope.md#amendment-2026-05-01--uc5-added-to-deferred-list).

### Q-1 amendment 2026-05-06 — UC5 expanded to eval-driven harness iteration (5a-5e)

- **Coupling:** Q-1 + Q-1 amendment 2026-05-01.
- **Rationale:** UC5 expanded to benchmark suite + trace
  consumption + config-level iteration interface + leaderboard;
  base for Pillar 3 KPI verification per
  [`project-overview.md` §1.1](../project-overview.md#11-четыре-столпа-цели-project-goal--four-pillars).
- **Source:** [ADR-1 §Amendment 2026-05-06](../adr/ADR-1-v01-use-case-scope.md#amendment-2026-05-06--uc5-expanded-to-eval-driven-harness-iteration).

## Q-2 — How are agent roles routed across LLM tiers? (2026-04-27)

- **Closed by:** [ADR-2](../adr/ADR-2-llm-tiering.md)
- **Chosen:** Static role routing — each role pinned to a tier in
  config; no auto-escalation.
- **Rejected:**
  - **Single-LLM, role differentiated only by prompt.** Reason:
    loses FA's tier-mix value proposition; cannot exploit OSS /
    elite cost asymmetry. Lesson: acceptable only when the budget
    mix collapses to one tier (e.g. fully local, fully Anthropic).
  - **Hybrid dynamic routing with a hard-task detector.** Reason:
    detector reliability is its own research problem; cost becomes
    unpredictable. Lesson: revisit after a stuck-loop /
    complexity detector exists with measurable precision/recall on
    FA's own task corpus.

## Q-3 — Which memory architecture variant for v0.1? (2026-04-27)

- **Closed by:** [ADR-3](../adr/ADR-3-memory-architecture-variant.md)
- **Coupling:** depends on Q-1 chosen option.
- **Chosen:** Variant A — Mechanical Wiki (filesystem-canonical
  Markdown + grep / FTS5 BM25).
- **Rejected:**
  - **Variant B — Hybrid Brain (canon + Mem0 4-op volatile store).**
    Reason: UC4 deferred (Q-1 chosen option), so the main
    beneficiary of B's design is not exercised in v0.1; UPDATE /
    DELETE classifier needs a top-tier LLM on every memory write.
    Lesson: becomes attractive once UC4 (multi-user TG) returns
    to scope or cross-session episodic memory is measured to
    underperform `hot.md`.
  - **Variant C — Layered KG (write-time typed-edge extraction +
    graph traversal).** Reason: most LoC and schema lock-in; graph
    cold-start empty until corpus density materialises; lead
    labelled overkill. Lesson: revisit only when corpus is dense
    enough that BM25 plateaus on multi-hop UC2 queries.

## Q-4 — Which storage backend hosts the disposable v0.1 index? (2026-04-27)

- **Closed by:** [ADR-4](../adr/ADR-4-storage-backend.md)
- **Coupling:** depends on Q-3 chosen option.
- **Chosen:** SQLite FTS5 (stdlib `sqlite3`, BM25 ranking via
  `MATCH` + `bm25()`).
- **Rejected:**
  - **In-memory BM25 (rank-bm25 / bm25s) persisted as a pickle.**
    Reason: pickle is brittle across Python / library versions;
    cold-rebuild O(corpus) on cache invalidation; no transaction
    story for an inbox-watcher. Lesson: acceptable only if SQLite
    FTS5 is unavailable on a target platform — e.g. an embedded
    build without the `sqlite3` stdlib's FTS5 compile-flag.
  - **External services — Postgres+pgvector / Elasticsearch /
    OpenSearch.** Reason: operational overhead inappropriate for
    single-workstation single-user v0.1; contradicts «working
    prototype faster». Lesson: revisit when FA grows past
    single-user OR when a vector layer is committed and the
    corpus exceeds SQLite's practical scale.
  - **Files-only — no index, grep at query time.** Reason: linear
    scan per query; no BM25 ranking; forces full match dumps into
    LLM context, breaking token-efficiency metric. Lesson: only
    viable for corpora small enough that grep latency stays
    sub-second AND no ranking is needed.

## Q-5 — Which chunker tool covers the v0.1 language set? (2026-04-28)

- **Closed by:** [ADR-5](../adr/ADR-5-chunker-tool.md)
- **Coupling:** depends on Q-3 chosen option.
- **Chosen:** universal-ctags (code) + markdown-it-py (prose) via
  a CompositeChunker.
- **Rejected:**
  - **tree-sitter-language-pack — 305 grammars in one wheel,
    full CST.** Reason: ~50–100 MB wheel; CHUNKER_VERSION
    management hard (305 grammars upgrade independently);
    PowerShell grammar fragmented across three upstream repos;
    Markdown weaker than markdown-it-py on corner cases. Lesson:
    becomes the right tool when intra-symbol splitting is required
    (e.g. for code-graph Variant C) or PowerShell grammar
    consolidates upstream.
  - **Per-language regex (sparks-style).** Reason: sparks needed
    >3000 lines of pure-Python `extract.py` for 6 languages — that
    is the floor; maintenance disproportionate to value vs ctags.
    Lesson: only viable when ctags and tree-sitter are both
    unavailable for a target language AND the language is small
    enough to regex by hand.

## Q-6 — How is Coder filesystem access constrained? (2026-04-29)

- **Closed by:** [ADR-6](../adr/ADR-6-tool-sandbox-allow-list.md)
- **Chosen:** Path allow-list with explicit read/write policy in
  `~/.fa/sandbox.toml`; default deny.
- **Rejected:**
  - **No sandbox — Coder reads/writes anything under user's home.**
    Reason: one hallucinated path away from corrupting
    `~/.ssh/authorized_keys`, `~/.aws/credentials`, browser
    profiles; violates `project-overview.md` §4 spirit. Lesson:
    never; only an adversarially-isolated environment (throwaway
    VM) makes this safe.
  - **OS-level sandbox (chroot, bubblewrap, Docker, macOS
    sandbox-exec).** Reason: cross-platform cost high
    (bubblewrap=Linux, sandbox-exec=macOS+deprecated,
    Docker=~250 MB + breaks `gh auth`); friction defeats use;
    educational forks would disable it. Lesson: becomes worth the
    cost when v0.2 ships `run_command` — pure-policy guard cannot
    intercept arbitrary shell.
  - **Pure prompt-level instruction («only edit files inside FA
    repo»).** Reason: hallucinating Coder will violate prompt;
    failure is silent. Lesson: useful only as a layer on top of
    the chosen option, never as a replacement.

## Q-7 — What is the v0.1 Coder inner-loop and tool-registry contract? (2026-05-12)

- **Closed by:** [ADR-7](../adr/ADR-7-inner-loop-tool-registry-contract.md)
- **Coupling:** inherits Q-2 (static role routing / no Critic),
  Q-4 (SQLite FTS5 future tool-search reuse), and Q-6 (sandbox hooks).
- **Chosen:** Minimal in-process MCP-shaped registry and Coder loop:
  progressive tool disclosure, JSON Schema validation, `pre_tool` /
  `post_tool` hooks, bounded ACI tools, dual edit formats, and raw
  `events.jsonl` traces.
- **Rejected:**
  - **Full MCP server topology in v0.1.** Reason: adds process,
    transport, auth, and deployment surface before the baseline loop.
    Lesson: revisit when v0.2 needs tool distribution or multi-process
    isolation; ADR-7 keeps the shape compatible.
  - **Amp-style three-tool loop with full-file reads.** Reason:
    whole-file payloads create context bloat on large UC1 repos and do
    not encode windowed ACI / trace invariants. Lesson: acceptable only
    for toy repos; production v0.1 needs bounded reads and artifacts.
  - **Always-on Critic / verifier / multi-candidate loop.** Reason:
    contradicts ADR-2's no-Critic v0.1 amendment and adds unmeasured
    token overhead before UC5 eval exists. Lesson: revisit after raw
    traces show a measurable failure mode and benchmark delta.
