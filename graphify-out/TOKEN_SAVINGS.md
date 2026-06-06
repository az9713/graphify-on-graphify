# Token-Savings Estimate — graphify on `./graphify-8/`

**Date:** 2026-06-06 · graphifyy 0.8.32 · graph: 6,188 nodes / 11,460 edges

## Measured inputs (this session, not estimates)

| Quantity | Value | Source |
|---|---|---|
| Corpus size | 844,497 words (540 files) | `graphify.detect.detect()` output |
| One-time semantic build cost | **1,533,913 tokens** | sum of 18 subagent usage figures, recorded in `cost.json` |
| AST (code) build cost | **0 LLM tokens** | tree-sitter, local |
| Avg query subgraph cost | **~8,875 tokens** | `graphify benchmark` (BFS depth-3 over 5 canned questions) |
| Per-question query range | ~2,500 – ~17,500 tokens | implied by benchmark per-question reductions |

## Correction to graphify's own "46.5×" figure

`benchmark.py` was invoked without `corpus_words`, so it **synthesized** the corpus size
as `nodes × 50` = 309,400 words (verified in `graphify/benchmark.py` L109-111). The real
detect count is 844,497 words. Using the benchmark's own words→tokens ratio (100 words ≈
133 tokens):

- Real naive corpus ≈ **1,126,000 tokens**
- True naive-stuffing reduction ≈ **126.9×** per query (not 46.5×)

The shipped figure *understates* savings against its own baseline by ~2.7× on this corpus.

## Methodology

Savings are computed as a **break-even analysis**:

```
break_even_queries = build_cost / (baseline_tokens_per_query − graph_tokens_per_query)
```

with three baselines, because "what you'd spend without graphify" depends on workflow:

| Baseline (per codebase question) | Per-query cost | Reduction | Savings/query | Break-even |
|---|---|---|---|---|
| **A. Full-context stuffing** (what graphify's benchmark models) | ~1,126,000 tok | 126.9× | ~1,117,000 | **~2 queries** |
| **B. Benchmark's printed figure** (synthetic 309K words) | ~412,500 tok | 46.5× | ~404,000 | **~4 queries** |
| **C. Realistic agentic search** (grep/glob + reading 5–20 files) | 20K / 40K / 80K tok | 2.3× / 4.5× / 9.0× | 11K / 31K / 71K | **138 / 49 / 22 queries** |

Baseline C is the honest comparator for a Claude Code user: an agent never stuffs the
whole corpus; it greps and reads selectively. The 20–80K range is an **assumption** based
on typical multi-file exploration in this 844K-word repo (5–20 file reads plus search
results per question); it was not measured.

### Bottom-line estimate

- Vs. context-stuffing: graphify pays for itself in **~2–4 questions** and saves ~0.4–1.1M
  tokens per question thereafter.
- Vs. realistic agentic search: graphify saves roughly **11K–71K tokens per question
  (2–9×)** and pays back its 1.53M-token build after **~22–138 questions** about this repo.
- For a repo you query a handful of times, the build doesn't pay back vs. agentic search.
  For a repo a team queries daily — or where the graph is committed and shared
  (`graphify-out/` is designed to be committed) — the build cost amortizes across every
  user and session, and the per-query economics dominate.

## Assumptions & preconditions

1. **Token arithmetic uses the 0.75 words/token heuristic** (benchmark's own ratio), not a
   real tokenizer. Code tokenizes denser than prose; true naive-corpus tokens are likely
   higher → savings vs. baselines A/B are conservative.
2. **Build cost is corpus-pathological.** ~60% of graphify-8's doc corpus is duplicated
   generated content (30 translated READMEs, 162 golden snapshots, 9 near-identical
   per-platform reference sets — subagents MD5-verified this). A 10-line
   `.graphifyignore` excluding `docs/translations/` and `tools/skillgen/expected/` would
   plausibly cut the semantic build to roughly 400–600K tokens (assumption, not measured),
   improving every break-even ~2.5–4×. Code extraction is free regardless.
3. **Subagent token figures are combined totals** from the Agent-tool usage field
   (input+output not separated); orchestration overhead in the main session (chunk
   planning, merging, labeling) is additional and not included (~low hundreds of K).
4. **Cache caveat:** `save_semantic_cache` recorded 0 files in this run (relative-path
   keying), so an `--update` after doc changes would re-pay semantic extraction for
   changed files rather than reusing this run's results. Code-only updates stay free.
5. **Query cost assumes** BFS depth-3 with default ~2K output budget; `--budget` caps it
   lower; pathological questions hitting hub nodes cost more (max observed ~17.5K).
6. **Quality precondition:** savings only count if the subgraph actually answers the
   question. Two of my five real test queries answered well; one (cache internals) latched
   onto the wrong vocabulary and would have needed a follow-up read of `cache.py` —
   real-world savings should be discounted by some answer-quality factor (call it
   70–90% effectiveness, per this session's small sample).
7. **Units are tokens, not dollars.** Build tokens were spent at interactive-session
   pricing; query tokens may be spent anywhere. Dollar savings depend on model mix.
