# HANDOFF.md — for the next agent / session

> **Read this first if you are an LLM agent (Devin, Claude, ChatGPT,
> Cursor, etc.) starting a new session on this repository.**
>
> **Last updated:** 2026-05-12 by Devin session
> [`1f41214431bd4c888071b6598c725710`](https://app.devin.ai/sessions/1f41214431bd4c888071b6598c725710).

This file is a portable counterpart to the Devin Knowledge note
"First-Agent — current state pointer". Both contain the same
information. The Knowledge note auto-injects into Devin sessions;
this Markdown file exists for any LLM client that does not have
Devin's MCP context (Claude Code, Cursor, ChatGPT with repo
access, plain `git clone`).

If both disagree, the Devin Knowledge note is canonical (it gets
updated more often). When you finish a session that materially
changes the project state, update **both**.

## 60-second bootstrap

> The five steps below are a condensed bootstrap for agents that
> land on `HANDOFF.md` first (e.g. via plain `git clone`, no Devin
> MCP). The canonical routing surface for LLM agents is
> [`knowledge/llms.txt`](./knowledge/llms.txt) §MUST READ FIRST
> (six files, in order). If the two disagree, llms.txt is canonical
> — step 2 below reads it, which closes the gap.

1. Read [`AGENTS.md`](./AGENTS.md) — repo conventions, PR
   checklist, query routing.
2. Read [`knowledge/llms.txt`](./knowledge/llms.txt) — one-fetch
   index of every documentation file in this repo
   ([llmstxt.org](https://llmstxt.org/) convention).
3. Skim [`knowledge/project-overview.md`](./knowledge/project-overview.md)
   — what the project is, what v0.1 ships, what is non-goal.
4. Skim the ADR index at
   [`knowledge/adr/README.md`](./knowledge/adr/README.md) — the
   accepted decisions that shape v0.1 (ADR-1..7).
5. Check the **Current state** section below for what is in
   flight right now.

You should now have everything you need. Do not crawl the repo
manually beyond this point.

## Current state (as of 2026-05-12)

- **Project stage:** **Stage 1** of the three-stage evolution
  (documentation + agent development через Devin). See
  [`knowledge/project-overview.md` §1.3](./knowledge/project-overview.md#13-three-stage-project-evolution)
  for the full ladder (Stage 2 — first-agent 0.1 локально + iteration
  через Devin; Stage 3 — first-agent self-improves, Devin as external
  advisor).
- **Inner-stage milestone:** Phase S scaffolding complete; ADR-7
  closes the inner-loop / tool-registry contract before the first
  feature-module PR (Phase M). `src/fa/chunker/` exists per ADR-5
  scaffolding, not yet end-to-end tested. No other feature module is
  implemented.
- **Working repos:** canonical
  [`GITcrassuskey-shop/First-Agent`](https://github.com/GITcrassuskey-shop/First-Agent);
  forks
  [`GrasshopperBoy/First-Agent-fork`](https://github.com/GrasshopperBoy/First-Agent-fork)
  and
  [`Bupitsa-ai/First-Agent-debloat`](https://github.com/Bupitsa-ai/First-Agent-debloat)
  are parallel work-trees — different agents (Devin sessions) run in
  different forks. PRs land upstream via cross-fork PR
  (`Contribute → Open pull request` on a fork page); the lead keeps
  `main` in sync across all three.
- **Architecture decisions (accepted, on `main` of both fork and upstream):**
  - [ADR-1](./knowledge/adr/ADR-1-v01-use-case-scope.md) — UC1
    (coding + PR write) and UC3 (local-docs-to-wiki) in;
    UC4 deferred; UC2 best-effort. **Amendment 2026-05-01:** UC5
    (semi-autonomous multi-LLM research/experiment) explicitly
    deferred.
  - [ADR-2](./knowledge/adr/ADR-2-llm-tiering.md) — static role
    routing (Planner / Coder / Debug / Eval) via
    `~/.fa/models.yaml`. **Amendment 2026-04-29:**
    `tool_protocol: native | prompt-only` field per role; v0.1
    inner-loop without Critic. **Amendment 2026-05-01:** MCP
    forward-compat tool-shape convention (JSON-RPC-shaped
    `name`/`params`/`result`/`error` for all tool dispatch).
  - [ADR-3](./knowledge/adr/ADR-3-memory-architecture-variant.md) —
    Variant A (mechanical wiki, no embeddings, no graph,
    no Mem0).
  - [ADR-4](./knowledge/adr/ADR-4-storage-backend.md) — SQLite
    FTS5 (external-content mode); zero extra deps. **Amendment
    2026-04-29:** chunks schema extended with provenance fields
    (`parent_title`, `breadcrumb`, `byte_start/end`, `topic`).
  - [ADR-5](./knowledge/adr/ADR-5-chunker-tool.md) —
    universal-ctags + markdown-it-py. **Amendment 2026-04-29:**
    `Chunk` dataclass extended to match ADR-4 provenance fields.
  - [ADR-6](./knowledge/adr/ADR-6-tool-sandbox-allow-list.md) —
    Tool sandbox + path allow-list policy (deny-by-default,
    `~/.fa/sandbox.toml`, gitignore-style globs, audit log at
    `~/.fa/state/sandbox.jsonl`, one-shot CLI bypass).
  - [ADR-7](./knowledge/adr/ADR-7-inner-loop-tool-registry-contract.md) —
    v0.1 Coder inner-loop and tool-registry contract: in-process
    MCP-shaped dispatcher, progressive tool disclosure, JSON Schema
    validation, `pre_tool` / `post_tool` hooks, bounded ACI tools,
    dual edit formats, and raw `events.jsonl` traces.
- **Scaffolding:** `pyproject.toml`, Ruff, mypy, pytest,
  pre-commit, GitHub Actions CI, `Makefile`, `markdown-it-py`,
  and system dependency documentation for `universal-ctags`
  are in place. CI in fork is limited to Devin Review (GitHub
  Actions are configured upstream; in the fork they are
  effectively no-op).
- **Open PRs in fork.** Check
  <https://github.com/GrasshopperBoy/First-Agent-fork/pulls>.
- **Research notes that close design questions:**
  - [`research/memory-architecture-design-2026-04-26.md`](./knowledge/research/memory-architecture-design-2026-04-26.md)
    — three variants for memory (input to ADR-3).
  - [`research/chunker-design.md`](./knowledge/research/chunker-design.md)
    — five tool classes, coverage matrix, ten open questions
    (input to ADR-5).
- **Research notes added 2026-04-29 (no ADR yet, inputs for v0.1+
  implementation and v0.2 roadmap):**
  - [`research/how-to-build-an-agent-ampcode-2026-04.md`](./knowledge/research/how-to-build-an-agent-ampcode-2026-04.md)
    — inner-loop micro-architecture from Thorsten Ball / Amp;
    three-tool baseline (`read_file` / `list_files` / `edit_file`);
    mapping to ADR-1 / ADR-2 and UC1.
  - [`research/sliders-structured-reasoning-2026-04.md`](./knowledge/research/sliders-structured-reasoning-2026-04.md)
    — SLIDERS framework (Stanford OV AL, arXiv:2604.22294)
    for QA over long document sets; mapping to ADR-3 / ADR-4 /
    ADR-5 and v0.2 extraction-layer roadmap.
  - [`research/cross-reference-ampcode-sliders-to-adr-2026-04.md`](./knowledge/research/cross-reference-ampcode-sliders-to-adr-2026-04.md)
    — cross-reference review of the two notes above against
    ADR-1..5: gaps, tensions, 10 numbered recommendations
    (R-1..R-10) and 10 open questions answered by lead in §11.
    **Q-1 / Q-2 marked superseded 2026-05-01** (ADR-6 became
    sandbox; inner-loop later closed by ADR-7).
- **Research note added 2026-05-01:**
  - [`research/semi-autonomous-agents-cross-reference-2026-05.md`](./knowledge/research/semi-autonomous-agents-cross-reference-2026-05.md)
    — critical analysis of three sources on semi-autonomous LLM
    agents (deep-research-report, semi-autonomous-llm-agents
    research, `nextlevelbuilder/goclaw` repo) against ADR-1..6.
    Accept / defer / reject filtering with explicit reasoning.
    Source for ADR-2 §Amendment 2026-05-01 (MCP forward-compat)
    and ADR-1 §Amendment 2026-05-01 (UC5 deferred). Input for
    ADR-7 inner-loop (ACI principle, hooks primitive).
- **Research notes added 2026-05-03:**
  - [`research/cutting-edge-agent-research-radar-2026-05.md`](./knowledge/research/cutting-edge-agent-research-radar-2026-05.md)
    — radar/backlog for First-Agent v0.1/v0.2 covering MCP/tool
    registry, ACI, hooks, memory, eval traces, sandbox/audit, and
    multi-agent coordination. Input for ADR-7 and future module PRs.
  - [`research/agent-ui-research-radar-v0-2-2026-05.md`](./knowledge/research/agent-ui-research-radar-v0-2-2026-05.md)
    — v0.2 UI radar covering Hermes Agent UI implementations,
    Pi surfaces/packages, OpenClaw gateway/UI forks, and
    Magentic-UI / DuetUI / AXIS research. Input for future
    UI/control-plane pre-ADR work.
- **Research note added 2026-05-07:**
  - [`research/efficient-llm-agent-harness-2026-05.md`](./knowledge/research/efficient-llm-agent-harness-2026-05.md)
    — consolidated research note for ADR-7 prep combining two
    upstream drafts (PR #37 + PR #38) into single source of
    truth. Nine resolved recommendations (R-1..R-9; 8 TAKE +
    1 DEFER, no surviving UNCERTAIN-ASK). Ships ADR-7 contract
    sketch (§10) — ToolSpec / ToolResult / Trace pseudo-schema +
    static layered prompt-assembly invariant + subtraction-first
    self-audit acceptance-block. Both upstream PR #37 and PR #38
    close without merge at cross-fork sync (lead action).

## Next steps (intended order)

1. **Implementation PR — chunker.** Implement `src/fa/chunker/`
   with the `Chunk` dataclass and `Chunker` Protocol from
   [ADR-5 §Decision](./knowledge/adr/ADR-5-chunker-tool.md#decision)
   (now including provenance fields per 2026-04-29 amendment).
   **Hard gate:** the four sample-tests in
   [`research/chunker-design.md` §8](./knowledge/research/chunker-design.md#8-sample-test-plan-pre-implementation)
   must pass — including the PowerShell sanity-check on the
   project lead's real 1500-line `.ps1` (not synthetic). The
   project lead should provide the real `.ps1` and a
   representative Go sample before this PR is considered
   mergeable.
2. **Chunker CLI surface.** Add `fa chunk <path>` for manual
   inspection of produced chunks as part of the chunker PR.
3. **R-3 edit-format fixture.** Run a 5-10 string-replace +
   5-10 unified-diff `apply_patch` test set on each
   tool-using model from ADR-2 (Qwen 3.6, Kimi 2.6, GLM 5.1,
   Claude latest, Nemotron 3 Super). Empirically verify that
   each model handles both edit-shapes; the result can tune
   prompting guidance for ADR-7 implementation. Optional; can be
   parallel.
4. **Glossary** (cross-reference §10 R-8 + semi-autonomous
   note §7.8): add `MCP`, `Hook`, `ACI`,
   `Reflexion / Critic / Reflector`, `Self-evolving` terms
   to [`docs/glossary.md`](./docs/glossary.md). Optional;
   not blocking ADR-7.
5. **v0.2 UI/control-plane pre-ADR** (optional if project lead
   prioritizes UI): use
   [`research/agent-ui-research-radar-v0-2-2026-05.md`](./knowledge/research/agent-ui-research-radar-v0-2-2026-05.md)
   to decide trace-viewer-first vs live-dashboard-first, local BFF
   shape, event schema, approval UI, and non-goals.

Phase-S item #7 (auto-generated `llms.txt`) is recorded in
[`docs/workflow.md`](./docs/workflow.md) as future work; not
blocking.

## Conventions worth knowing in 5 lines

- **Branch:** `devin/<unix-timestamp>-<slug>` from `main`.
- **Commit trailer:** every LLM-driven commit ends with
  `AI-Session: <session-id>` plus
  `Co-Authored-By: <human> <email>` (codedna pattern).
  Trailers must be in the squash-merge commit body — see PRs #19,
  #21, #23 for examples.
- **PR description:** every changed/new file listed as a clickable
  blob URL on the head branch (rule #6).
- **`knowledge/llms.txt`:** must be updated whenever `docs/` or
  `knowledge/` changes structure (rule #7).
- **Research note language:** notes are read by both humans and
  agents. Prefer Russian analytical prose and recommendations unless
  the project lead asks otherwise; keep protocol names, API fields,
  code, frontmatter keys, and direct quotes in their original language
  where precision matters.
- **Code fences:** always have a language tag (`python`, `yaml`,
  `text`, …); never a bare ` ``` ` opening.

## When to update this file

- After any PR merge that materially changes the project state
  (new ADR, new phase, new module, new big research note).
- Bump the **Last updated** date at the top.
- Also update the matching Devin Knowledge note (or replace it
  entirely with the body of this file — they are meant to be
  identical).

## Why this file exists

Pattern lifted from the educational angle of this project: every
convention should be discoverable via either Devin-specific
infrastructure (Knowledge note) **or** plain repo browsing
(this file). Forks of this repo as a starter template do not
necessarily use Devin; HANDOFF.md is what they get for free.
