# Graphify Command Log — complete session record

**Corpus:** `./graphify-8/` (a clone of the graphify v8 repo itself — 540 files, ~844K words)
**Environment:** Windows 11 · Python 3.13 host / 3.10 tool venv · graphifyy **0.8.32** via `uv tool install`
**Dates:** 2026-06-05 → 2026-06-06

Legend: 🟢 ran clean · 🟡 partial / caveat · 🔴 not runnable · ⭐ **SURPRISING result**

---

## Phase 1 — Install

### 1. `uv tool install graphifyy` 🟢
```
Installed 32 packages: graphifyy==0.8.32, networkx, scipy, tree-sitter + 24 grammar
packages (python, ts, go, rust, java, c, cpp, swift, fortran, zig, verilog, …)
Installed 1 executable: graphify
```

### 2. `graphify install --platform windows` 🟢
```
references       ->  C:\Users\simon\.claude\skills\graphify\references
skill installed  ->  C:\Users\simon\.claude\skills\graphify\SKILL.md
CLAUDE.md        ->  skill registered in C:\Users\simon\.claude\CLAUDE.md
```
The skill became invocable as `/graphify` after a skill reload.

---

## Phase 2 — Full build pipeline (`/graphify ./graphify-8/`)

### 3. Step 1 — interpreter detection 🟢
Resolved `C:\Users\simon\AppData\Roaming\uv\tools\graphifyy\Scripts\python.exe`, saved to
`graphify-out/.graphify_python`. (Two Windows gotchas hit and fixed: PowerShell 5.1
`-Encoding utf8` writes a BOM; Python stdout redirects default to cp1252 → fixed with
BOM-free writes + `PYTHONUTF8=1`.)

### 4. Step 2 — `graphify.detect.detect("./graphify-8/")` 🟢
```
Corpus: 540 files · ~844,497 words
  code:   196 files   docs: 341   papers: 1   images: 2 (.svg)   sensitive skipped: 0
```
540 > 500 triggered the corpus-size gate. Top subfolders: tools/ 162, graphify/ 161,
tests/ 140, worked/ 36, docs/ 33. User chose **whole repo anyway**.

### 5. Step 3A — AST structural extraction (local, $0) 🟢
```
AST extraction: 196/196 files (100%) [12 workers]
AST: 5,565 nodes, 11,220 edges
```

### 6. Step B0 — `check_semantic_cache` 🟢
```
Cache: 0 files hit, 540 files need extraction   (fresh corpus)
```

### 7. Steps B1–B3 — semantic extraction via 18 parallel subagents 🟢
344 non-code files → 16 doc chunks + 2 image chunks. All 18 chunk files landed on disk.
```
Merged 18 chunks: 1,533,913 subagent tokens, 738 raw nodes, 955 edges
After dedup: 652 unique semantic nodes
```
🟡 Caveat: `save_semantic_cache` reported `Cached 0 files` — relative `source_file`
paths didn't key into the cache, so a re-run would re-extract everything.

⭐ **Surprise (chunk agents):** several agents independently MD5-hashed the per-platform
reference docs and found **all 22-file platform copies byte-identical** except kiro/pi
(compact extraction-spec variant) — confirming the corpus is ~60% generated/duplicated
content (`graphify/skills/*/references/`, `tools/skillgen/expected/`).

### 8. Part C — merge AST + semantic 🟢
```
Merged: 6,190 nodes, 12,175 edges (5,565 AST + 652 semantic)
```

### 9. Step 4 — build + Leiden cluster + analyze 🟢
```
Graph: 6,188 nodes, 11,460 edges, 501 communities
```

### 10. Step 5 — community labeling 🟢
25 hand-written labels for the largest communities (Installer Tests, AST Extraction Core,
Skillgen Golden Snapshots, Always-On Platform Fragments, …) + 476 auto-derived from
dominant file stem. 194 communities have ≥10 nodes; the rest are isolates.

### 11. Step 6 — `graphify export html` 🟡
```
Graph has 6188 nodes (above 5000 limit). Building aggregated community view...
graph.html written (aggregated: 501 community nodes, 791 cross-community edges)
```
Per the honesty rules: full node-level HTML was not generated — too many nodes.

### 12. Step 8 — `graphify benchmark` 🟢
```
Corpus:          309,400 words → ~412,533 tokens (naive)
Avg query cost:  ~8,875 tokens  →  46.5× fewer tokens per query
  [ 23.6×] how does authentication work
  [ 81.6×] what is the main entry point
  [165.1×] how are errors handled
  [ 39.4×] what connects the data layer to the api
  [ 46.6×] what are the core abstractions
```

### 13. Step 9 — manifest + cost tracker 🟢
```
This run: 1,533,913 input tokens, 0 output tokens (1 run, 540 files)
```

### Run-1 report highlights
**God nodes:** Path 89 · str 85 · `ingest_scip_json()` 84 · `_make_id()` 63 · `main()` 61 ·
`_labels()` 59 · bytes 56 · DataProcessor 56 · `extract()` 54
**Top suggested question:** why does `main()` bridge 20+ communities (betweenness 0.155,
28 unverified INFERRED edges).

---

## Phase 3 — Query / analysis commands against the built graph

### 14. `graphify query "what connects the skillgen generator to the platform skill files"` 🟢
BFS depth=2 from `Platform` → 38 nodes. Mapped the full anti-drift pipeline in one answer:
`load_platforms()` (platforms.toml) → `_read_fragment()` → `render()/render_all()/
render_always_on()` → `RenderedArtifact`, guarded by `audit_coverage()` +
`monolith_roundtrip()`. Explains why ~30% of the corpus is golden snapshots in
`tools/skillgen/expected/`.

### 15. `graphify query "how does watch mode trigger graph rebuilds"` 🟢
82 nodes. Full rebuild chain: `watch()` → `_rebuild_lock()` / `_check_shrink()` →
`_rebuild_code()` → `detect()` + `collect_files()` + `extract()` → `to_json()/to_html()` →
`generate()` report → `write_callflow_html()`.
⭐ **Surprise:** `check_graph_file_size_cap()` from **security.py** sits inside the watch
rebuild path — the watcher is security-gated against runaway graph files.

### 16. `graphify query "what is the relationship between the extraction cache and semantic subagents"` 🟡
22 nodes. Query expansion latched onto the worked-example review docs (rsl-siege-manager
findings) rather than `cache.py` internals — still surfaced `file_hash()`,
`save_cached()`, `cached_files()`, but the doc nodes dominated. Shows query quality
depends on label-vocabulary match.

### 17. ⭐ `graphify path "main()" "extract()"` 🟢 — **SURPRISING**
```
Shortest path (2 hops):
  main() --calls [INFERRED]--> _rebuild_code() --calls [INFERRED]--> extract()
```
The shortest route from the CLI entry point to the AST extractor runs **through the
file-watcher's `_rebuild_code()`** (watch.py) — not through a dedicated extract command
path. Combined with run-1's betweenness data (`_rebuild_code()` = #2 bridge, 0.073),
watch mode is structurally the spine of the system, not a peripheral feature.

### 18. `graphify path "Skillgen Generator (gen.py)" "Graphify Skill (canonical ...)"` 🟡
```
No path found.
```
The skillgen code cluster and the canonical-skill *document* cluster are disconnected in
the graph — semantic doc nodes and AST code nodes only join where an agent emitted an
explicit cross edge. (Both lookups also warned "match was ambiguous".)

### 19. ⭐ `graphify explain "ingest_scip_json()"` 🟢 — **SURPRISING**
```
Node: graphify_scip_ingest_ingest_scip_json   (scip_ingest.py L42)
Degree: 84
```
~60 of its 84 edges are **individual edge-case test functions** (duplicate-edge dedup,
non-dict input, missing target symbol, deterministic node IDs, boolean-flag defaults, …).
One SCIP-ingestion function is the most defensively-tested symbol in the entire repo —
a clear scar-tissue signal around external code-intelligence input.

### 20. `graphify affected "_make_id()"` 🔴
```
No unique node match for _make_id()
```
Several `_make_id` definitions exist across files; `affected` requires an unambiguous
label and offers no disambiguation picker. (Lookup limitation, not a graph gap.)

### 21. ⭐ `graphify cluster-only . --graph graphify-out/graph.json --exclude-hubs 99 --no-label --no-viz` 🟢 — **MOST SURPRISING**
```
Re-clustering...
[graphify] backed up curated graph (5 files) -> 2026-06-06/
Done - 971 communities. GRAPH_REPORT.md and graph.json updated.
```
Two-part result:
- **Partitioning changed hugely**: 501 → **971 communities**. Excluding p99-degree hubs
  from Leiden splits the hub-glued mega-clusters — this works as documented.
- **God-node ranking did NOT change**: `Path` (89), `str` (85), `bytes` (56) still top
  the regenerated report. The README's common-commands table claims the flag will
  *"suppress utility super-hubs from god-node rankings"* — **observed behavior
  contradicts that wording** (the command-reference wording, "exclude p99 degree nodes
  from partitioning", matches reality). Candidate upstream bug/doc report. Ironically the
  repo's own `worked/` reviews document this exact degree-centrality artifact.

**Manually re-ranked god nodes** (builtins filtered): `test_languages.py` 228 ·
`extract.py` 203 · `manifest.json` (worked example) 167 · `test_detect.py` 111 ·
`ingest_scip_json()` 84 · `gen.py` 76 · `callflow_html.py` 72 · `_make_id()` 63 ·
`main()` 61 — the real centers are the AST extractor, SCIP ingestion, node-ID minting,
and the skillgen generator.

### 22. `graphify export callflow-html` 🟢
```
Call-flow HTML written: graphify-out\graphify_chase_ai-callflow.html
Sections: 18 | Mermaid diagrams: 16 | Call tables: 15  (zoom/pan controls)
```
The most readable artifact of the session.

### 23. `graphify diagnose multigraph --max-examples 3` 🟢
```
source_location_variant_groups: 0
context_variant_groups: 0
post_build_graph_type: Graph · post_build_edges: 11,460
producer_suppression_sites: 54   (seen_ids, seen_call_pairs, seen_swift_base, …)
```
No same-endpoint edge collapse in the built graph, but 54 intentional duplicate-
suppression sites inside `extract.py` — the tool itself warns raw producer loss must be
measured before graph.json.

---

## Not runnable on this corpus

| Command | Why |
|---|---|
| `graphify prs` / `prs 42` / `prs --triage` / `prs --conflicts` 🔴 | `graphify-8/` has **no `.git`** — no repo, no GitHub remote, no PRs |
| `graphify hook install` / `hook status` 🔴 | same — git hooks need a git repo |
| `graphify add <url>` ⏭️ | skipped deliberately — mutates the corpus (downloads into `./raw/`) |
| `/graphify --update` ⏭️ | skipped — no files changed since the build |
| Gemini backend (`extract_corpus_parallel`) ⏭️ | no `GEMINI_API_KEY`/`GOOGLE_API_KEY` set → Claude subagent path used instead |

---

## ⭐ The surprising results, ranked

1. **`--exclude-hubs 99` doc-vs-behavior mismatch** (#21) — re-partitions (501→971
   communities) but does **not** re-rank god nodes, contradicting the README's
   common-commands description. Best candidate for an upstream issue.
2. **The watcher is the spine** (#17) — `main() → _rebuild_code() → extract()` is the
   shortest CLI→extractor path; `_rebuild_code()` is the graph's #2 betweenness bridge
   and is security-gated (#15).
3. **`ingest_scip_json()` is the most edge-case-hardened function in the repo** (#19) —
   ~60 dedicated tests on a single function.
4. **~60% of the corpus is generated duplication** (#7) — byte-identical per-platform
   reference docs + 162 golden snapshots, all existing to serve skillgen's anti-drift
   guarantee (#14).
5. **Doc-nodes and code-nodes form weakly-joined continents** (#18) — "No path found"
   between skillgen code and the canonical skill document; run-1 reported 2,215
   weakly-connected nodes.

## Session cost & artifacts

- **Tokens:** ~1.53M subagent tokens (semantic extraction) + queries at ~9K each;
  AST extraction free/local. Benchmark: **46.5× avg reduction** per query.
- **Artifacts:** `graph.json`, `graph.html` (aggregated), `GRAPH_REPORT.md` (971-community
  version; 501-community run-1 in `backup-run1/`), `graphify_chase_ai-callflow.html`,
  `FINDINGS.md`, `COMMANDS_LOG.md` (this file), `cost.json`, `manifest.json`,
  auto-backup `2026-06-06/`.
