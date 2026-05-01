# ADR-2 — LLM tiering & access

- **Status:** accepted
- **Date:** 2026-04-27
- **Deciders:** project owner (`0oi9z7m1z8`), Devin (drafting)

## Context

FA's value proposition (see
[`project-overview.md`](../project-overview.md) §1) includes
**routing different agent roles to different LLM tiers** instead of
defaulting one model to everything. The user has stated a budget
mix of approximately:

- 60 % top-tier OSS (GLM 5.1 / Kimi 2.6 / Xiaomi Mimo 2.5)
- 30 % mid-tier OSS (Nemotron 3 Super / Qwen 3.6 27B)
- 10 % elite (Anthropic Claude latest available)

We need to decide **how roles map to tiers**, **how access is
configured**, and **what fallback behaviour** is acceptable.

User answers in PR-#17 follow-up (Q2 + Q3):

- Q2 (role routing): "Multi-LLM static routing: Planner=top-tier OSS,
  Coder=mid-tier OSS, Debug/elite=Claude — config-driven, всегда так."
- Q3 (access): "Mix: per-model в config (некоторые local, некоторые
  через OpenRouter, elite через Anthropic)."

## Options considered

### Option A — Single-LLM, role by prompt

- Pros: simplest; one provider; predictable cost.
- Cons: cannot leverage tier mix; loses FA's stated value.

### Option B — Static role routing (chosen)

Each role pinned to a tier in config; never auto-escalates.

- Pros:
  - Predictable cost: a Coder turn never silently calls Anthropic.
  - Predictable behaviour: same role + same input → same provider.
  - Simple to debug — rerun a turn against its known tier.
- Cons:
  - No graceful degradation when the pinned model is down (must be
    handled by per-role fallback chain in config).
  - Hard tasks routed to Coder fail loudly instead of escalating.
    User-stated preference accepts this.

### Option C — Hybrid dynamic routing with hard-task detector

Mid-tier by default, escalate to top-tier or elite on a detector
signal (e.g. complexity heuristic, stuck-loop detector).

- Pros: cost-optimised; auto-recovery for hard cases.
- Cons: detector reliability is a research problem of its own; not
  appropriate for v0.1; costs become unpredictable.

## Decision

We will choose **Option B (static role routing)** with the following
concrete mapping for v0.1:

| Role | Tier | Default model | Provider |
|---|---|---|---|
| **Planner** | top-tier OSS | GLM 5.1 (or Kimi 2.6 / Mimo 2.5 — config-pickable) | AnyProvider API key / OpenRouter |
| **Coder** | mid-tier OSS | Nemotron 3 Super (or Qwen 3.6 27B) | AnyProvider API key / OpenRouter |
| **Debug / elite** | top tier | DIFFERENT top-tier OSS / top tier from AnyProvider API key | AnyProvider API key / OpenRouter |
| **Eval (LLM-as-judge)** | top-tier OSS | DIFFERENT model ; isolated config slot so judge can be version-pinned | AnyProvider API key / OpenRouter |

Configuration lives in a single YAML/TOML file (e.g.
`~/.fa/models.yaml`) with one block per role:

```yaml
planner:
  primary:   { provider: openrouter, model: "z-ai/glm-5.1" }
  fallback:  { provider: AnyProvider, model: "GLM-5.1-Air" }
coder:
  primary:   { provider: AnyProvider, model: "Nemotron-3-Super-49B" }
  fallback:  { provider: openrouter, model: "qwen/qwen3-coder-27b" }
debug:
  primary:  { provider: any, model: "claude-opus-4-7-20260301" }
  fallback: { provider: any, model: "claude-sonnet-4-7-20260301" }
judge:
  primary: { provider: openrouter, model: "z-ai/kimi 2.6", pinned: true }
```

> **Note on model slugs.** The strings above (`z-ai/glm-5.1`,
> `claude-opus-4-7-20260301`, etc.) are illustrative of the
> *shape* of the config, not authoritative slugs at any given
> date. Provider catalogs change; pick the actual current slug
> from OpenRouter / Anthropic / vLLM at config time. The
> *decision* is the table above (which tier each role lives in
> and how `primary → fallback` chains); the *implementation*
> resolves slugs to whatever is current.

- "primary → fallback" chain per role; **no cross-tier escalation**
  on failure.
- Anthropic is the only mandatory remote in v0.1 (for Debug). Coder
  is preferentially local (vLLM); Planner can be either. This matches
  the user's "remote API ≈ 99 %" tolerance while leaving headroom
  for local-only Coder if vLLM is configured.

## Consequences

- **Positive:** Cost predictability — a Coder turn cannot silently
  hit Anthropic. Token-efficiency metric in
  [`project-overview.md`](../project-overview.md) §3 becomes
  meaningful.
- **Positive:** Per-role evaluation is straightforward — swap one
  block of config to A/B-test models on a role.
- **Positive:** The `judge:` role being version-pinned mitigates
  R5 (eval baseline drift) from `project-overview.md` §7.
- **Negative:** No auto-escalation means a Coder failure on a hard
  task surfaces as a hard error; the **user must explicitly invoke
  Debug** (or rewrite the prompt). v0.2 may revisit this if the
  pattern shows real friction.
- **Negative:** Three providers (Anthropic + OpenRouter + local vLLM)
  triples auth surface area and failure modes (R3 in
  `project-overview.md` §7). Mitigated by per-role fallback chain.
- **Follow-up work this unlocks:**
  - `src/fa/llm/router.py` — minimal role-based dispatcher reading
    `~/.fa/models.yaml`.
  - Provider adapters: `provider_client.py`, `openrouter_client.py`,
    `vllm_local_client.py` (one thin wrapper per provider).
  - Secrets policy: `~/.fa/secrets.env` (chmod 600), never committed.
  - Decision deferred to a future ADR: how to express token / cost
    budgets per role in the same config.

## Amendments

### Amendment 2026-04-29 — `tool_protocol` field + native-by-default; v0.1 inner-loop without Critic

**Source.** Cross-reference review
[`research/cross-reference-ampcode-sliders-to-adr-2026-04.md`](../research/cross-reference-ampcode-sliders-to-adr-2026-04.md)
§3.3, §9.6, §10 R-1 / R-7 — the ADR's per-role `primary →
fallback` chain does not specify how the tools are wired into
each model. Native-tool models (Anthropic, OpenAI, Qwen 3.6,
Kimi 2.6, GLM-5.1) and prompt-only models accept tool calls in
**different shapes**. A silent fallback from native to
prompt-only would break the inner-loop. The user has confirmed
that the current model picks (Qwen 3.6 / Kimi 2.6 / Claude
latest) all support native tool-calling, so native is the
default.

Independently, ADR-2 §Decision lists Planner / Coder / Debug /
Eval but does not name a separate **Critic / Reflector** role.
[`research/agent-roles.md`](../research/agent-roles.md) §5.1
proposes Planner / Executor / Critic as the minimum trio.
Cross-reference §3.4 / §9.7 confirm v0.1 explicitly does
**not** include a Critic (no reflection / self-correction loop).
Eval (offline LLM-as-judge) is not a Critic — it judges
finished work, not in-loop turns. This amendment fixes the
terminology so a v0.2 implementer does not silently introduce a
Critic.

**Decision (additive to the original Decision section).**

1. **`tool_protocol` field per-role in `~/.fa/models.yaml`.**
   Allowed values: `native` | `prompt-only`. Default for **any
   role that calls tools** is `native`. Default for the `judge`
   role (LLM-as-judge, no tool calls) is irrelevant — set to
   `native` for shape consistency, the inner-loop ignores it.

   ```yaml
   coder:
     primary:   { provider: AnyProvider, model: "Nemotron-3-Super-49B", tool_protocol: native }
     fallback:  { provider: openrouter, model: "qwen/qwen3-coder-27b", tool_protocol: native }
   ```

2. **No mixing of `native` and `prompt-only` within a single
   role's `primary → fallback` chain.** The `models.yaml` loader
   enforces this at startup; mixed configurations are a hard
   error, not a warning. Rationale: silent shape changes mid-
   session corrupt the conversation accumulator (see ADR-3
   `hot.md` invariant).

3. **Loop adapts to the role's `tool_protocol`, not to the
   model.** This isolates the tool-shape decision from model
   choice — swapping `coder.primary` to a different native model
   is one-line; switching the whole role to `prompt-only` is
   one-line; mixing within a chain is forbidden.

4. **`prompt-only` is supported but not used in the v0.1
   reference config.** Kept as an option for forks that pin to
   models without native tool-calling (older OSS releases, some
   self-hosted vLLM models). Implementation must include the
   prompt-only path so swapping is config-only, never code.

5. **v0.1 inner-loop has no Critic / Reflector role.** The roles
   are exactly: Planner, Coder, Debug (manual escalation only —
   see original `## Consequences` §«No auto-escalation»), Eval
   (offline judge, out-of-band). Reflection / self-correction
   loops are **v0.2** material; design is in-flight (user note,
   Apr 2026) and will land as a separate ADR. The ADR-2
   no-auto-escalation clause means "no cross-tier escalation",
   not "no intra-role retry-loop"; an intra-role retry-loop
   (e.g. Coder retrying after a failed `edit_file` validation)
   stays allowed in v0.1 — see cross-reference §9.7.

**Notes.**

- The `tool_protocol` field is consumed by `src/fa/llm/router.py`
  + the inner-loop module specified in the planned **Inner-loop
  ADR** (cross-reference §10 R-1, not yet drafted). Until that
  ADR lands, the implementer may stub a single `native`-only
  inner-loop and mark `prompt-only` as `NotImplemented`; the
  field still goes into the schema so the config never has to
  be re-written.
- Verified model coverage (user, Apr 2026): Qwen 3.6, Kimi 2.6,
  GLM 5.1, Claude latest, Nemotron 3 Super — all native-tool.
  Mid-tier OSS prompt-only fallbacks remain possible for
  budget-constrained forks.

**Consequence.** `models.yaml` schema gains a required
`tool_protocol` field per role (with `native` as the default if
unset, to keep current configs valid). The validator added in
the implementation PR refuses `primary` and `fallback` blocks
that disagree on `tool_protocol`. Rejecting the config at startup
is a hard error: this matches the original ADR's "fails loudly"
posture for hard-task escalation.

## Amendment 2026-05-01 — MCP forward-compat tool-shape convention

**Context.** Three independent sources surveyed in
[`research/semi-autonomous-agents-cross-reference-2026-05.md`](../research/semi-autonomous-agents-cross-reference-2026-05.md)
§2.3-A and §3.3-B (deep-research-report on agent
architectures, semi-autonomous-agents research, and the
`nextlevelbuilder/goclaw` README) call the **Model Context
Protocol (MCP)** the de-facto industry standard for the
agent ↔ tools boundary as of 2026. MCP defines a
JSON-RPC-shaped contract between an MCP host (the agent)
and one or more MCP servers (which expose tools, resources,
and prompts).

The original Decision and the 2026-04-29 amendment fix the
agent ↔ LLM contract (`tool_protocol: native | prompt-only`).
They say nothing about the **agent ↔ tools** contract — i.e.
how the inner-loop calls a tool function, whether locally
in-process or eventually via an MCP server. Without an
explicit convention, the future inner-loop ADR (cross-
reference §10 R-1, now planned as ADR-7) might design a
tool-shape that is **not** JSON-RPC-shaped, which would force
re-design when v0.2 wants to expose internal tools as MCP
servers (so other MCP hosts — Claude Desktop, third-party
agents — can use them, or so heavy tools like `mcp-runner` /
`mcp-web` can be moved to separate processes).

This amendment fixes the convention now, at zero
implementation cost, because native function-calling APIs
(Anthropic, OpenAI, Qwen-native) already deliver
JSON-shaped tool inputs and outputs.

**Decision (additive to the original Decision section and
the 2026-04-29 amendment).**

1. **MCP-shaped tool signatures.** All v0.1 tools — including
   the in-process Python functions used by the inner-loop —
   expose a JSON-RPC-shaped surface:

   ```text
   request:  { name: str, params: dict[str, Any] }
   response: { result: Any | None, error: { code: int, message: str } | None }
   ```

   Tools are invoked through a single dispatcher that
   accepts and returns these shapes; tool-specific Python
   code does not appear in the LLM-facing protocol. Tool
   parameters are described by **JSON Schema** (parsed from
   Python type hints + Pydantic models or hand-written),
   which is the MCP-spec shape. Tool errors carry a numeric
   code and a string message, matching JSON-RPC `error`
   semantics.

2. **Stable tool-name policy.** Tool names are stable strings
   (`repo.read`, `repo.search`, `git.status`, …) — not
   Python function objects. Renaming a tool is a v-bump
   event (semantic versioning of the tool catalogue). This
   matches the way MCP servers identify tools by name.

3. **No `mcp` package dependency in v0.1.** This amendment
   defines a **convention**, not a dependency. The agent does
   not `pip install mcp` in v0.1; it implements an in-process
   dispatcher whose **shape** is MCP-compatible. Adding the
   real `mcp` Python package, spawning external mcp-servers,
   exposing internal tools as remote MCP services, etc. — all
   v0.2 work, gated by a follow-up ADR.

4. **Inner-loop ADR (future ADR-7) inherits the convention.**
   When ADR-7 lands, its tool-registry contract must use this
   request/response shape. The ADR-7 author MAY add fields
   (e.g. an `id` for streaming tool-calls, a `metadata`
   block) but MUST NOT change the existing two fields
   (`name`, `params` for request; `result`, `error` for
   response) without a separate amendment to this ADR-2.

5. **`tool_protocol` field semantics extended.** The existing
   `tool_protocol: native | prompt-only` field per role
   (2026-04-29 amendment) describes only the **agent ↔ LLM**
   side. The **agent ↔ tools** side is fixed by this
   amendment as JSON-RPC-shaped regardless of `tool_protocol`
   value. A `prompt-only` role still emits JSON-shaped
   tool-calls (just embedded in text it has to parse) and
   the dispatcher still receives JSON-shaped requests.

**Notes.**

- This amendment is a **forward-compat convention**, not a
  constraint that changes any v0.1 implementation surface
  visible to the user. The tool-call cost is zero new code:
  every native function-calling provider (Anthropic, OpenAI,
  Qwen-native, Kimi, GLM, Nemotron) already produces
  JSON-shaped tool-call objects, and any prompt-only
  fallback would have to parse JSON from the LLM output
  anyway.
- The MCP spec (`https://modelcontextprotocol.io`) is still
  evolving (transport layer changes between 2024 and
  2025-2026 — added HTTP+SSE alongside STDIO). Pinning to a
  specific transport in v0.1 would be premature; we only
  pin to **shape**.
- ADR-6 sandbox check is the canonical pre-tool hook in this
  shape: it intercepts the dispatcher's request, validates
  the path argument against the allow-list, and either
  forwards the request or returns an `error` response. This
  is the same shape ADR-7 will use when it formalises
  hooks (cross-reference R-1 input from 2026-05-01 note
  §7.1).

**Consequence.** Tool registry implementation (deferred to
ADR-7 / Phase M PR) must produce a JSON-RPC-shaped dispatcher
even though tools are in-process Python functions in v0.1.
This costs no extra code (native function-calling produces
the right shape natively), buys zero v0.1 user-visible
features, and keeps v0.2 MCP-server distribution as a
config-only / wrapper-only change.

## References

- [`project-overview.md`](../project-overview.md) §6 (key constraints — LLM providers).
- [`research/memory-architecture-design-2026-04-26.md`](../research/memory-architecture-design-2026-04-26.md) §1 bullet 2 (mixed-LLM design constraint).
- [`research/agent-roles.md`](../research/agent-roles.md) §5.1 (Planner / Executor / Critic minimum-set rationale; Coder maps to Executor here; v0.1 omits Critic — see 2026-04-29 amendment).
- [`research/cross-reference-ampcode-sliders-to-adr-2026-04.md`](../research/cross-reference-ampcode-sliders-to-adr-2026-04.md) §3.3 / §9.6 / §10 R-1 / R-7 — rationale for the 2026-04-29 amendment.
- [`research/how-to-build-an-agent-ampcode-2026-04.md`](../research/how-to-build-an-agent-ampcode-2026-04.md) §3.1 / §4 — native tool-calling shape.
- [`research/semi-autonomous-agents-cross-reference-2026-05.md`](../research/semi-autonomous-agents-cross-reference-2026-05.md) §2.3-A / §3.3-B / §7.1 — rationale for the 2026-05-01 MCP forward-compat amendment.
- MCP specification: [https://modelcontextprotocol.io](https://modelcontextprotocol.io) — JSON-RPC + STDIO/HTTP+SSE transport.
- PR #17 review (`https://github.com/GITcrassuskey-shop/First-Agent/pull/17`) — Q2 + Q3 verbatim answers.
