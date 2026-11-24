---
name: deep-research-plus
description: >-
  Run research as an explicit breadth/depth recursive loop instead of one free-form pass: generate
  search queries with a stated research goal, extract concise structured learnings and follow-up
  questions from each batch of results, dedupe facts and sources, then recurse with halved breadth
  and decremented depth until depth hits 0. Use when the user asks for "deep research", an
  iterative/exhaustive investigation, explicitly gives breadth/depth or a number of research
  rounds, or when a single research pass on a broad topic is visibly missing follow-up angles.
  Complements the built-in deep-research skill (which coordinates research subagents into a
  narrative report) by adding a deterministic stop condition and a structured learnings/questions
  extraction step; prefer this one when the user wants predictable, boundable iteration rather than
  an open-ended subagent swarm.
metadata:
  type: user
---

# Deep Research Plus

Adapted from the recursive breadth/depth loop in [dzhng/deep-research](https://github.com/dzhng/deep-research)
(an independent <500 LoC research-agent implementation), reworked so Claude runs the loop itself with its
own search/fetch tools instead of calling an external pipeline.

## When to use this over a plain research pass

Use this when the user wants **iteration with a visible stop condition** — not just "go research X and
summarize", but something that keeps narrowing in on follow-up questions for a bounded number of rounds.
Signals: the user says "深入研究"/"iterative"/"exhaustive", gives explicit breadth/depth numbers, or a first
research pass left obvious unanswered follow-ups worth another round.

## Parameters

Ask only if not inferable from the request; otherwise default:

- **breadth** (queries per round): default 4. Higher = wider net this round.
- **depth** (recursion levels): default 2. Higher = more rounds of narrowing.
- **output**: "report" (long, cites every learning, Sources section) or "answer" (one short, direct
  answer in whatever format the prompt implies — a sentence, a number, a choice). Default to "report"
  unless the prompt is clearly a single factual question.

## The loop

Maintain two running lists across all rounds: `learnings` (deduped strings) and `visitedUrls` (deduped
strings). Start both empty.

**Each round, given `query`, `breadth`, `depth`, and the running `learnings`:**

1. **Generate queries.** Produce up to `breadth` search queries from `query` (and the running
   `learnings`, if any, to sharpen them). For each query, also write a one-line **research goal**:
   what this specific query is meant to establish, and how to go deeper once results are in. Queries
   must be mutually distinct — don't generate near-duplicates.

2. **Search.** Run each query (WebSearch, or WebFetch on specific pages the query points at). For
   `breadth` ≤ 3 run them inline; for larger fan-out, dispatch each as a fork so they run in parallel
   and don't bloat this context with raw page text.

3. **Extract, per query's results:**
   - **learnings**: up to 3 concise, information-dense facts. Preserve every entity, number, date, and
     proper noun verbatim — these are what the next round's queries will sharpen against. No fluff,
     no "sources suggest" hedging language; state the fact.
   - **followUpQuestions**: up to 3 concrete next-step questions this result surfaces.
   Merge new learnings and visited URLs into the running lists, deduping exact repeats.

4. **Recurse or stop.** `newBreadth = ceil(breadth / 2)`, `newDepth = depth - 1`.
   - If `newDepth > 0`: build the next round's `query` from this round's research goal plus its
     follow-up questions (concatenate, don't just pick one), then repeat from step 1 with
     `breadth = newBreadth`, `depth = newDepth`.
   - If `newDepth == 0`: this branch is done — its learnings/URLs are already merged into the running
     lists; stop recursing on this branch.

Breadth halves and depth decrements on every recursive step — this is the deterministic stop condition.
Don't let the model decide ad hoc whether "enough" research has been done; the numbers decide.

## Final output

- **report**: write a long-form Markdown report from `query` using every item in the deduped
  `learnings` list — don't drop any for brevity. End with a `## Sources` section listing every deduped
  URL in `visitedUrls`.
- **answer**: write the shortest possible direct answer (a phrase, number, or sentence) matching
  whatever format the prompt implies, using `learnings` as support. No preamble, no "based on my
  research".

## Notes

- Dedup learnings and URLs at every merge point, not just at the end — later rounds reference the
  running `learnings` list to sharpen their queries, so stale duplicates degrade that sharpening.
- If a query times out or errors, drop it (empty learnings/URLs for that branch) and continue with the
  others — don't let one bad query stall the whole round.
- This produces a wide, shallow-then-narrow tree, not a deep chain: breadth shrinks fast (4→2→1), so by
  depth 2–3 you're down to a handful of highly specific follow-ups. For a topic that needs *more*
  rounds rather than a wider first pass, raise `depth` and keep `breadth` modest (3–4).
