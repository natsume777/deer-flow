---
name: systematic-literature-review
description: Use this skill whenever the user wants to survey, synthesize, or do a systematic literature review (SLR) across multiple academic papers on a topic. Triggers on queries like "review the literature on X", "survey recent papers about Y", "do an SLR on Z", "what does the literature say about W", "summarize recent research in A", "compare findings across papers on B", "annotated bibliography on C", or "what are the research trends in D". Make sure to use this skill even when the user does not say the word "systematic" — the defining signal is that they want a synthesis across MANY papers rather than a deep read of a single one. Also trigger when the user asks for a "state of the art" summary or a "research landscape" overview. Distinct from `academic-paper-review`, which does single-paper peer review. This skill searches arXiv, extracts structured metadata from each paper in parallel via subagents, synthesizes themes across the set, and emits a report in APA, IEEE, or BibTeX citation format.
---

# Systematic Literature Review Skill

## Overview

This skill produces a structured **systematic literature review (SLR)** across multiple academic papers on a research topic. Given a topic query, it searches arXiv, extracts structured metadata (research question, methodology, key findings, limitations) from each paper in parallel, synthesizes themes across the full set, and emits a final report with consistent citations.

**Distinct from `academic-paper-review`:** that skill does deep peer review of a single paper. This skill does breadth-first synthesis across many papers. If the user hands you one paper URL and asks "review this paper", route to `academic-paper-review` instead.

**Distinct from `deep-research`:** that skill does general web research across diverse sources. This skill is narrowly scoped to academic literature on arXiv with formal citation formatting. If the user wants a non-academic overview of a topic, route to `deep-research` instead.

## When to Use This Skill

Use this skill when the user wants any of the following:

- A literature survey on a topic ("survey transformer attention variants", "review the literature on diffusion models")
- A synthesis across multiple papers ("what do recent papers say about X", "compare methodologies across papers on Y")
- A systematic review with consistent citation format ("do an SLR on Z in APA format")
- An annotated bibliography on a topic
- An overview of research trends in a field over a time window

Do **not** use this skill when:

- The user provides exactly one paper and asks to review it (use `academic-paper-review`)
- The user asks a factual question that does not require synthesizing multiple sources (answer directly)
- The user wants general web research without academic rigor (use standard web search)

## Workflow

The workflow has five phases. Follow them in order.

### Phase 1: Plan

Before doing any retrieval, confirm the following with the user. If any of these are unclear, ask **one** clarifying question that covers the missing pieces. Do not ask one question at a time.

- **Topic**: the research area in plain English (e.g. "transformer attention variants").
- **Scope**: how many papers (default 20, hard upper bound 50), optional time window (e.g. "last 2 years"), optional arXiv category (e.g. `cs.CL`, `cs.CV`). For common category mappings, see the reference table below.
- **Citation format**: APA, IEEE, or BibTeX (default APA if the user does not specify and does not seem to be writing for a specific venue).
- **Output location**: where to save the final report (default `/mnt/user-data/outputs/`).

If the user says "50+ papers", politely cap it at 50 and explain that synthesis quality degrades quickly past that — for larger surveys they should split by sub-topic.

**Common arXiv category mappings** (use these to translate natural-language field descriptions into `--category` flags):

| User's field description | arXiv category |
|---|---|
| NLP / natural language processing | `cs.CL` |
| Computer vision / image recognition | `cs.CV` |
| Machine learning (general) | `cs.LG` |
| AI / artificial intelligence | `cs.AI` |
| Robotics | `cs.RO` |
| Information retrieval / search | `cs.IR` |
| Statistics / statistical learning | `stat.ML` |
| Physics (general) | `physics` |
| Mathematics | `math` |

When the user's topic spans multiple categories (e.g. "multimodal learning" which touches `cs.CV` and `cs.CL`), omit the `--category` flag and let the query alone do the filtering — adding a single category would exclude relevant cross-disciplinary papers.

### Phase 2: Search arXiv

Call the bundled search script. Do **not** try to scrape arXiv by other means and do **not** write your own HTTP client — this script handles URL encoding, Atom XML parsing, and id normalization correctly.

```bash
python /mnt/skills/public/systematic-literature-review/scripts/arxiv_search.py \
  "<topic>" \
  --max-results <N> \
  [--category <cat>] \
  [--sort-by relevance] \
  [--start-date YYYY-MM-DD] \
  [--end-date YYYY-MM-DD]
```

**IMPORTANT — extract 2-3 core keywords before searching.** Do not pass the user's full topic description as the query. Before calling the script, mentally reduce the topic to its 2-3 most essential terms. Drop qualifiers like "in computer vision", "for NLP", "variants", "recent" — those belong in `--category` or `--start-date`, not in the query string.

**Query phrasing — keep it short.** The script wraps multi-word queries in double quotes for phrase matching on arXiv. This means:

- `"diffusion models"` → searches for the exact phrase → good, returns relevant papers.
- `"diffusion models in computer vision"` → searches for that exact 5-word phrase → **too specific, likely returns 0 results** because few papers contain that exact string.

Use **2-3 core keywords** as the query, and use `--category` to narrow the field instead of stuffing field names into the query. Examples:

| User says | Good query | Bad query |
|---|---|---|
| "diffusion models in computer vision" | `"diffusion models" --category cs.CV` | `"diffusion models in computer vision"` |
| "transformer attention variants" | `"transformer attention"` | `"transformer attention variants in NLP"` |
| "graph neural networks for molecules" | `"graph neural networks" --category cs.LG` | `"graph neural networks for molecular property prediction"` |

The script prints a JSON array to stdout. Each paper has: `id`, `title`, `authors`, `abstract`, `published`, `updated`, `categories`, `pdf_url`, `abs_url`.

**Sort strategy**:

- **Always use `relevance` sorting** — arXiv's BM25-style scoring ensures results are actually about the user's topic. `submittedDate` sorting returns the most recently submitted papers in the category regardless of topic relevance, which produces mostly off-topic results.
- When the user asks for "recent" papers or gives a time window, use `--sort-by relevance` **combined with `--start-date`** to constrain the time range while keeping results on-topic. For example, "recent diffusion model papers" → `--sort-by relevance --start-date 2024-01-01`, not `--sort-by submittedDate`.
- `submittedDate` sorting is only appropriate when the user explicitly asks for chronological order (e.g. "show me papers in the order they were published"). This is rare.
- `lastUpdatedDate` is rarely useful; ignore it unless the user asks.

**Run the search exactly once.** Do not retry with modified queries if the results seem imperfect — arXiv's relevance ranking is what it is. Retrying with different query phrasings wastes tool calls and risks hitting the recursion limit. If the results are genuinely empty (0 papers), tell the user and suggest they broaden their topic or remove the category filter.

**If the script returns fewer papers than requested**, that is the real size of the arXiv result set for the query. Do not pad the list — report the actual count to the user and proceed.

**If the script fails** (network error, non-200 from arXiv), tell the user which error and stop. Do not try to fabricate paper metadata.

**Do not save the search results to a file** — the JSON stays in your context for Phase 3. The only file saved during the entire workflow is the final report in Phase 5.

### Phase 3: Extract metadata in parallel

**You MUST delegate extraction to subagents via the `task` tool — do not extract metadata yourself.** This is non-negotiable. Specifically, do NOT do any of the following:

- ❌ Write `python -c "papers = [...]"` or any Python/bash script to process papers
- ❌ Extract metadata inline in your own context by reading abstracts one by one
- ❌ Use any tool other than `task` for this phase

Instead, you MUST call the `task` tool to spawn subagents. The reason: extracting 10-50 papers in your own context consumes too many tokens and degrades synthesis quality in Phase 4. Each subagent runs in an isolated context with only its batch of papers, producing cleaner extractions.

**Graceful degradation when subagents are unavailable**: If the `task` tool is not available (because `subagent_enabled` is `false`), do **not** silently skip Phase 3 or abort. Instead:
1. Inform the user that subagents are disabled and the review will be less thorough.
2. Fall back to extracting metadata yourself in-context, but limit to at most 10 papers to avoid context exhaustion. If the user requested more than 10, explain the constraint and offer to either (a) enable subagents for the full set, or (b) proceed with a 10-paper subset.
3. Use the same JSON schema described below so Phase 4 works identically regardless of extraction method.

Split papers into batches of ~5, then for each batch, call the `task` tool with `subagent_type: "general-purpose"`. Each subagent receives the paper abstracts as text and returns structured JSON.

**Concurrency limit: at most 3 subagents per turn.** The DeerFlow runtime enforces `MAX_CONCURRENT_SUBAGENTS = 3` and will silently drop any extra dispatches in the same turn — the LLM will not be told this happened, so strictly follow the round strategy below.

**Round strategy — use this decision table, do not compute the split yourself**:

| Paper count | Batches of ~5 papers | Rounds | Per-round subagent count |
|---|---|---|---|
| 1–5 | 1 batch | 1 round | 1 subagent |
| 6–10 | 2 batches | 1 round | 2 subagents |
| 11–15 | 3 batches | 1 round | 3 subagents |
| 16–20 | 4 batches | 2 rounds | 3 + 1 |
| 21–25 | 5 batches | 2 rounds | 3 + 2 |
| 26–30 | 6 batches | 2 rounds | 3 + 3 |
| 31–35 | 7 batches | 3 rounds | 3 + 3 + 1 |
| 36–40 | 8 batches | 3 rounds | 3 + 3 + 2 |
| 41–45 | 9 batches | 3 rounds | 3 + 3 + 3 |
| 46–50 | 10 batches | 4 rounds | 3 + 3 + 3 + 1 |

**Never dispatch more than 3 subagents in the same turn.** When a row says "2 rounds (3 + 1)", that means: first turn dispatches 3 subagents in parallel, wait for all 3 to complete, then second turn dispatches 1 subagent. Rounds are strictly sequential at the main-agent level.

If the paper count lands between rows (e.g. 23 papers), round up to the next row's layout but only dispatch as many batches as you actually need — the decision table gives you the shape, not a rigid prescription.

**Do the batching at the main-agent level**: you already have every paper's abstract from Phase 2, so each subagent receives pure text input. Subagents should not need to access the network or the sandbox — their only job is to read text and return JSON. Do not ask subagents to re-run `arxiv_search.py`; that would waste tokens and risk rate-limiting.

**What each subagent receives**, as a structured prompt:

When calling the `task` tool, you must provide all three required parameters in this order:
1. `description`: a short label, e.g. "Extract metadata batch 1"
2. `prompt`: the full extraction instructions (see template below)
3. `subagent_type`: always `"general-purpose"` for this phase

```
Extract structured metadata and key findings from the following arXiv papers.

Papers:
[Paper 1]
arxiv_id: 1706.03762
title: Attention Is All You Need
authors: Ashish Vaswani, Noam Shazeer, ...
published: 2017-06-12
abstract: <full abstract text>

[Paper 2]
arxiv_id: ...
...

For each paper, return a JSON object with exactly these fields:
- arxiv_id (string) — must match the input id exactly
- title (string)
- authors (list of strings)
- published_date (string, YYYY-MM-DD)
- research_question (1 sentence, what problem the paper tackles)
- methodology (1-2 sentences, how they tackle it)
- key_findings (list of 3-5 strings, what they actually found)
- limitations (1-2 sentences, what they acknowledge or what is obviously missing)
- categories (list of strings, from the input metadata)

Return the result as a JSON array, one object per paper, in the same
order as the input. Do not include any text outside the JSON — no
preamble, no markdown fences, just the raw JSON array.
```

**Parsing subagent results**: the task tool returns strings with a fixed prefix like `Task Succeeded. Result: [...JSON...]`. Strip the `Task Succeeded. Result: ` prefix (or `Task failed.` / `Task timed out.` / `Task cancelled by user.` prefixes) before trying to parse JSON. If a batch fails or returns unparseable JSON, log it, note which papers were affected, and continue with the remaining batches — do not fail the whole synthesis on one bad batch.

**Handling partial failures**: After all rounds complete, check how many papers were successfully extracted vs. lost to failures. If more than 30% of papers failed extraction, warn the user in the report's methodology section. The synthesis in Phase 4 should only cover papers with successful extractions — do not speculate about papers that failed.

After all rounds complete, flatten the per-batch arrays into a single list of paper metadata objects, preserving order. Validate that each object has the required `arxiv_id` field — drop malformed entries and log which papers were affected.

### Phase 4: Synthesize and format

Now produce the final SLR report. Two things happen here: cross-paper synthesis (thematic analysis) and citation formatting.

**Cross-paper synthesis**: the report must do more than list papers. At minimum, identify:

- **Themes**: 3-6 recurring research directions, approaches, or problem framings across the set.
- **Convergences**: findings that multiple papers agree on.
- **Disagreements**: where papers reach different conclusions or use incompatible methodologies.
- **Gaps**: what the collective literature does not yet address (often stated explicitly in the "limitations" fields).
- **Temporal trends** (optional but encouraged): if the paper set spans multiple years, note how the field's focus has shifted over time — e.g. "Earlier papers (2020–2022) focused on X, while more recent work (2023–2025) has shifted toward Y."

If the paper set is too small or too heterogeneous to support thematic synthesis (e.g. 5 papers on wildly different sub-topics), say so explicitly in the report — do not force themes that are not there.

**Synthesis quality bar**: A good synthesis connects papers to each other, not just to the topic. For each theme, at least two papers should be discussed in relation to one another (agreeing, extending, contradicting). If you find yourself writing theme sections that mention only one paper each, that is a summary — not a synthesis. Restructure until each theme weaves multiple papers together.

**Citation formatting**: the exact format depends on user preference. Read **only** the template file that matches the user's requested format, not all three:

- [templates/apa.md](templates/apa.md) — APA 7th edition. Default for social sciences and most CS journals. Use when the user requests APA or does not specify a format.
- [templates/ieee.md](templates/ieee.md) — IEEE numeric citations. Use when the user targets an IEEE conference or journal, or explicitly asks for IEEE.
- [templates/bibtex.md](templates/bibtex.md) — BibTeX entries. Use when the user mentions BibTeX, LaTeX, or wants machine-readable references. **Important**: arXiv papers are cited as `@misc`, not `@article` — the BibTeX template covers this explicitly.

Each template contains both the citation rules and a full report structure (executive summary, themes, per-paper annotations, references, methodology section). Follow the template's structure verbatim for the report body, then fill in content from your Phase 3 metadata.

### Phase 5: Save and present

Save the full report to `/mnt/user-data/outputs/slr-<topic-slug>-<YYYYMMDD>.md` where `<topic-slug>` is a lowercased hyphenated version of the topic (e.g. `transformer-attention`), truncated to 50 characters max. Then call the `present_files` tool with that path so the user can download it.

**In the chat message**, show a short preview so the user immediately sees value without opening the file:

1. **Executive summary** — the 3–5 sentence paragraph from the top of the report, verbatim.
2. **Themes list** — bullet list of the themes you identified in Phase 4 synthesis (just the theme names + one-line gloss, not the full theme sections).
3. **Key statistics** — number of papers surveyed, date range covered, number of papers lost to extraction failures (if any).
4. **Paper count + a pointer to the file** — e.g. "Full report with 20 papers, per-paper annotations, and formatted references saved to `slr-transformer-attention-20260409.md`."

Do **not** dump the full 2000+ word report inline — per-paper annotations, references, and methodology belong in the file. The preview is there to let the user judge the report at a glance and decide whether to open it.

## Examples

**Example 1: Typical SLR request**

User: "Do a systematic literature review of recent transformer attention variants, 20 papers, APA format."

Your flow:
1. Phase 1: confirm topic (transformer attention variants), scope (20 papers, default time window), format (APA). Ask **one** clarification only if something is missing (e.g. "Any particular time window, or should I default to the last 3 years?").
2. Phase 2: `arxiv_search.py "transformer attention" --max-results 20 --sort-by relevance --start-date 2023-01-01`.
3. Phase 3: 20 papers → round 1 = 3 subagents × 5 papers = 15 covered, round 2 = 1 subagent × 5 papers = 5 covered. Aggregate.
4. Phase 4: read `templates/apa.md`, write the report using its structure, fill in themes + per-paper annotations from Phase 3 metadata.
5. Phase 5: save to `slr-transformer-attention-20260409.md`, call `present_files`.

**Example 2: Small-set request with ambiguity**

User: "Survey a few papers on diffusion models for me."

Your flow:
1. Phase 1: "a few" is ambiguous. Ask one question: "How many papers would you like — 10, 20, or 30? And any citation format preference (APA is the default)?"
2. User responds "10, BibTeX".
3. Phase 2: `arxiv_search.py "diffusion models" --max-results 10 --category cs.CV`.
4. Phase 3: 10 papers → single round, 2 subagents × 5 papers.
5. Phase 4: read `templates/bibtex.md`, format with `@misc` entries (not `@article`).
6. Phase 5: save and present.

**Example 3: Out-of-scope request**

User: "Here's one paper (https://arxiv.org/abs/1706.03762). Can you review it?"

This is a single-paper peer review, not a literature survey. Do not use this skill. Route to `academic-paper-review` instead.

**Example 4: Subagents unavailable**

User: "Review the literature on federated learning, 25 papers."

Your flow:
1. Phase 1: confirm topic (federated learning), scope (25 papers), format (APA default).
2. Phase 2: `arxiv_search.py "federated learning" --max-results 25 --sort-by relevance`.
3. Phase 3: `task` tool is not available. Inform the user: "Subagents are disabled, so I'll do a condensed review of the top 10 papers instead of the full 25. Enable subagents for the complete set." Extract metadata for 10 papers in-context.
4. Phase 4: synthesize themes from the 10 extracted papers. Note the reduced scope in the methodology section.
5. Phase 5: save and present, noting the reduced scope in the preview.

## Notes

- **Subagents are strongly recommended but not strictly required**. Phase 3 works best with the `task` tool for parallel metadata extraction (`subagent_enabled=true` in `config.configurable`). Without subagents, the skill falls back to in-context extraction with a 10-paper cap — functional but less thorough. Always inform the user when operating in fallback mode.
- **arXiv only, by design**. This skill does not query Semantic Scholar, PubMed, or Google Scholar. arXiv covers the bulk of CS/ML/physics/math preprints, which is what DeerFlow users most often want to survey. Multi-source academic search belongs in a dedicated MCP server, not inside this skill. If the user's topic is primarily covered outside arXiv (e.g. biomedical research on PubMed, social science on SSRN), warn them that coverage will be incomplete.
- **Hard upper bound of 50 papers**. This is tied to the Phase 3 concurrency strategy (max 3 subagents per round, ~5 papers each, at most 4 rounds). Surveys larger than 50 papers degrade in synthesis quality and are better done by splitting into sub-topics.
- **Subagent results are strings, not objects**. Always strip the `Task Succeeded. Result: ` / `Task failed.` / `Task timed out.` / `Task cancelled by user.` prefixes before parsing the JSON payload.
- **The `id` field is a bare arXiv id** (e.g. `1706.03762`), not a URL and not with a version suffix. `abs_url` / `pdf_url` hold the full URLs if you need them.
- **Synthesis, not listing**. The final report must identify themes and compare findings across papers. A report that only lists papers one after another is a failure mode — if you cannot find themes, say so explicitly instead of faking them.
- **Abstract-only extraction**. This skill works from abstracts, not full paper text. Synthesis quality is bounded by what abstracts reveal. The methodology section of the report should acknowledge this limitation.
