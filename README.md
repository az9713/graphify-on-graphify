# graphify-on-graphify

**[graphify](https://github.com/safishamsi/graphify) pointed at its own source code** — a
full `/graphify` knowledge-graph build of the graphify v8 repo, run from Claude Code on
Windows, with every artifact, raw command output, and token receipt committed as evidence.

### ▶ [Open the interactive knowledge graph](https://az9713.github.io/graphify-on-graphify/)

> 6,188 nodes · 11,460 edges · 971 communities (aggregated view) — click nodes, search,
> filter, zoom. Served via GitHub Pages; GitHub strips scripts/iframes from READMEs, so
> the live graph runs at the link above rather than inline.
> Also live: [Mermaid call-flow architecture page](https://az9713.github.io/graphify-on-graphify/graphify-out/graphify_chase_ai-callflow.html).

- `graphify-8/` — the corpus: a snapshot of [safishamsi/graphify](https://github.com/safishamsi/graphify) (v8 branch, MIT — see `graphify-8/LICENSE`). 540 files · ~844K words · 196 code / 341 docs / 1 paper / 2 SVGs.
- `graphify-out/` — everything the run produced: graph, reports, findings, benchmark, and an `evidence/` folder of raw CLI outputs.

| Read this if you want… | File |
|---|---|
| The surprises | [`graphify-out/FINDINGS.md`](graphify-out/FINDINGS.md) |
| Every command + its output | [`graphify-out/COMMANDS_LOG.md`](graphify-out/COMMANDS_LOG.md) |
| Token-savings math | [`graphify-out/TOKEN_SAVINGS.md`](graphify-out/TOKEN_SAVINGS.md) |
| The graph itself | [live interactive view](https://az9713.github.io/graphify-on-graphify/) · [`graph.json`](graphify-out/graph.json) |
| Mermaid architecture page | [live view](https://az9713.github.io/graphify-on-graphify/graphify-out/graphify_chase_ai-callflow.html) · [`source`](graphify-out/graphify_chase_ai-callflow.html) |
| Raw proof for each claim | [`graphify-out/evidence/`](graphify-out/evidence/) |

---

## The command sequence

```bash
# install (Windows)
uv tool install graphifyy                      # graphifyy 0.8.32 → `graphify` CLI
graphify install --platform windows            # registers the /graphify skill in Claude Code

# build (inside Claude Code)
/graphify ./graphify-8/
#   Step 2  detect            → 540 files · 844,497 words (>500-file gate; chose whole repo)
#   Step 3A AST extraction    → 5,565 nodes / 11,220 edges from 196 code files, $0, local
#   Step 3B semantic          → 18 parallel subagents over 344 docs/images → 652 nodes / 955 edges
#   Step 4  build + cluster   → 6,188 nodes / 11,460 edges / 501 communities
#   Step 5  label             → 25 hand labels + 476 auto labels
#   Step 6  graphify export html   → aggregated view (graph > 5,000-node limit)
#   Step 8  graphify benchmark     → 46.5× avg token reduction
#   Step 9  manifest + cost.json   → 1,533,913 LLM tokens (semantic build, one-time)

# interrogate
graphify query "what connects the skillgen generator to the platform skill files"
graphify query "how does watch mode trigger graph rebuilds"
graphify query "what is the relationship between the extraction cache and semantic subagents"
graphify path "main()" "extract()"
graphify path "Skillgen Generator (gen.py)" "Graphify Skill (canonical /graphify skill template)"
graphify explain "ingest_scip_json()"
graphify affected "_make_id()"
graphify cluster-only . --graph graphify-out/graph.json --exclude-hubs 99 --no-label --no-viz
graphify export callflow-html
graphify diagnose multigraph --max-examples 3
graphify benchmark
```

Not runnable on this corpus: `graphify prs*` and `graphify hook install` (the snapshot has
no `.git`); `graphify add` skipped (mutates the corpus).

---

## ⭐ The surprises

### 1. `--exclude-hubs 99` re-partitions but does **not** re-rank god nodes
The README's common-commands table says the flag will *"suppress utility super-hubs from
god-node rankings."* Observed: communities went **501 → 971** (hub-exclusion from Leiden
partitioning works), but the god-node list in the regenerated report is **byte-for-byte
the same** — still topped by `Path` (89 edges), `str` (85), `bytes` (56). The
command-reference wording elsewhere in the same README ("exclude p99 degree nodes *from
partitioning*") matches reality; the headline wording doesn't.
**Evidence:** `backup-run1/GRAPH_REPORT.md` + `backup-run1/graph.json` (before) vs
`GRAPH_REPORT.md` + `graph.json` (after).
With builtin hubs filtered manually, the real top abstractions are `extract.py` (203
edges), `ingest_scip_json()` (84), `gen.py` (76), `_make_id()` (63), `main()` (61).

### 2. The file-watcher is the structural spine of the system
```
graphify path "main()" "extract()"
→ main() --calls--> _rebuild_code() --calls--> extract()    (2 hops)
```
The shortest path from the CLI entry point to the AST extractor runs through
**`watch.py`'s `_rebuild_code()`** — also the graph's #2 betweenness bridge (0.073) — and
the rebuild path is security-gated by `check_graph_file_size_cap()` from `security.py`.
Watch mode reads as a side feature in the docs; structurally it's the artery.
**Evidence:** `evidence/path_main_to_extract.txt`, `evidence/query_watch_mode.txt`.

### 3. `ingest_scip_json()` is the most defensively-tested function in the repo
Degree 84, and ~60 of those edges are individual edge-case test functions (duplicate-edge
dedup, non-dict input, missing target symbol, deterministic IDs, …). Nothing else in
6,188 nodes comes close — a scar-tissue map pointing at external SCIP-index ingestion.
**Evidence:** `evidence/explain_ingest_scip_json.txt`.

### 4. ~60% of the corpus is generated duplication — and the graph explains why
Extraction subagents MD5-verified that the 9 per-platform reference-doc sets under
`graphify-8/graphify/skills/*/references/` are **byte-identical** (except kiro/pi's
compact variant), and `tools/skillgen/expected/` holds 162 golden snapshots. One query
(`evidence/query_skillgen.txt`) surfaced the machine that justifies all of it:
`load_platforms() → _read_fragment() → render_all() → audit_coverage() /
monolith_roundtrip()` — skillgen's anti-drift guarantee for 20 platform variants.

### 5. Docs and code form weakly-joined continents
`graphify path` between the skillgen *code* cluster and the canonical-skill *document*
node: **"No path found."** Run-1 reported 2,215 weakly-connected nodes. Semantic doc
nodes and AST code nodes only join where an extraction agent emitted an explicit cross
edge. **Evidence:** `evidence/path_skillgen_to_skill.txt`.

### 6. The benchmark's own headline number is wrong — in graphify's favor
`graphify benchmark` printed **46.5×** token reduction against a corpus of "309,400
words". That number is synthetic: when not given a word count, `benchmark.py` falls back
to `nodes × 50` (6,188 × 50 = 309,400 — check `graphify-8/graphify/benchmark.py` L109-113).
The real corpus is 844,497 words, so the true naive-stuffing reduction is **~127×**.
**Evidence:** `evidence/benchmark.txt` + the source file itself.

---

## Token savings (full methodology in [`TOKEN_SAVINGS.md`](graphify-out/TOKEN_SAVINGS.md))

Measured: **1,533,913 tokens** one-time semantic build (`cost.json`; code AST was free) ·
**~8,875 tokens** per query (range 2.5K–17.5K).

Break-even = build ÷ (baseline − query cost), three baselines:

| Baseline per question | Reduction | Savings/query | Break-even |
|---|---|---|---|
| Stuff whole corpus in context (~1.13M tok) | ~127× | ~1.12M | **~2 queries** |
| Benchmark's printed baseline (~413K tok) | 46.5× | ~404K | **~4 queries** |
| Realistic agentic grep+read, 20–80K tok *(assumed)* | 2.3–9× | 11K–71K | **22–138 queries** |

Key assumptions: 0.75 words/token heuristic throughout; the agentic baseline is assumed,
not measured; the build cost was inflated ~2.5–4× by this corpus's duplicated generated
content (a 10-line `.graphifyignore` would fix that); savings should be discounted by an
answer-quality factor (~70–90% — one of five test queries missed its target vocabulary,
see `evidence/query_cache.txt`).

---

## Reproduce

```bash
git clone https://github.com/az9713/graphify-on-graphify
cd graphify-on-graphify
uv tool install graphifyy
graphify benchmark                       # deterministic — matches evidence/benchmark.txt
graphify path "main()" "extract()"       # matches evidence/path_main_to_extract.txt
graphify explain "ingest_scip_json()"    # matches evidence/explain_ingest_scip_json.txt
graphify query "anything about this codebase"
```

All query/path/explain/benchmark commands are local graph traversals over the committed
`graphify-out/graph.json` — no API key needed.

## Attribution

`graphify-8/` is a snapshot of [safishamsi/graphify](https://github.com/safishamsi/graphify)
(v8 branch), © Safi Shamsi, redistributed under its MIT license (`graphify-8/LICENSE`).
This repo adds the knowledge-graph build outputs, analysis, and evidence in
`graphify-out/` and this README. Built with [Claude Code](https://claude.com/claude-code).
