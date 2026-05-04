---
title: "<one-line title>"
source:
  - "<primary URL or repo path>"
compiled: "<YYYY-MM-DD>"
chain_of_custody: |
  <where to find the primary source for specific facts; see
  knowledge/README.md §Provenance-frontmatter for the conventions>
# goal_lens: one-sentence research goal elicited at session start;
# required for notes produced via knowledge/prompts/research-briefing.md,
# optional otherwise. See knowledge/README.md §Frontmatter v2.
goal_lens: "<one-sentence research goal>"
# v2 optional fields below — additive, leave out the ones that do not apply
tier: stable
links: []
mentions: []
confidence: extracted
claims_requiring_verification:
  - "<claim 1>"
---

> **Status:** active. Note produced via
> [`knowledge/prompts/research-briefing.md`](../prompts/research-briefing.md).
>
> §0 below is the Decision Briefing intended for the project lead and
> for future LLM agents reading the note from the top. It mirrors the
> chat-handover the agent posted at session end. §1.. are deep-dive
> sections; load them only when §0 is insufficient.

## 0. Decision Briefing

### R-N — <short title>

- **What:** 1–2 sentences, plain language, no jargon.
- **Project-axis fit (stable across notes):**
  - (A) reduces session-start noise: YES (~X tokens saved) | NO | UNKNOWN
  - (B) helps LLM find context when needed: YES (pointer-shape) | NO | UNKNOWN
- **Goal-lens fit (per session, dynamic):**
  - (C) advances chosen goal_lens "<verbatim from frontmatter>": YES
    (1-sentence reason) | PARTIAL (1-sentence caveat) | NO
- **Cost:** cheap (<1h) | medium (1–4h) | expensive (>4h)
- **Verdict:** TAKE | SKIP | DEFER | UNCERTAIN-ASK
- **If UNCERTAIN-ASK:** <one question, 3–4 concrete options>
- **Alternative-if-rejected:** <one sentence path-not-taken>
- **Concrete first step (if TAKE):** <file path / command / 1-line action>

<!-- Repeat the block above for R-2, R-3, ... R-K. -->

### Summary

| R-N | Verdict | Project-fit (A / B) | Goal-fit (C) | Cost | Alternative-if-rejected | User decision needed? |
|-----|---------|---------------------|--------------|------|--------------------------|------------------------|
| R-1 | TAKE    | YES / YES           | YES (tag)    | medium | <fallback path>        | No (TAKE)              |
| R-2 | …       | …                   | …            | …    | …                        | …                      |

<!--
  Goal-fit (C) cell carries Y / PARTIAL / N + a 2–3-word tag; the full
  1-sentence reason lives in the per-R block above.
  Use `n/a` with a one-clause reason in any cell that does not apply.
-->

## 1. TL;DR

- 5–7 bullet points summarising the deep-dive below.

## 2. Scope, метод

- What sources were read; what was deliberately excluded.
- Method: cross-reference / replication / literature review / etc.
- Goal-lens (verbatim): "<text from frontmatter>".

## 3. Key concepts (<source-language terms>)

- Bullet list with one-line definitions for the protocol / API / paper
  terms used downstream. Keep terms in source language for precision.

## 4. Mapping / analysis

<!--
  The actual cross-reference / analysis content. Use tables where the
  structure repeats (e.g. one row per ADR / per concept / per option).
  Subsections as needed (4.1, 4.2, ...).
-->

## 5. Risks and caveats

- Open caveats / unverified claims / contested numbers.

## 6. Numbered recommendations (R-1..R-K)

<!-- Long-form prose for each R; §0 references back here for details. -->

### R-1 — <name> (cost: cheap | medium | expensive)

<!-- Couple of paragraphs explaining the recommendation, the failure
     mode it prevents, and concrete first step.  -->

## 7. Open questions (Q-1..Q-M)

### Q-1 — <one-line question>

<!-- One paragraph stating the unknown, why it matters, and what would
     resolve it. -->

## 8. Files used

- <bulleted list of source URLs / repo paths actually read>

## 9. Out of scope

- <bulleted list of things the note deliberately does not cover>
