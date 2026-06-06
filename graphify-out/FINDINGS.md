# Graphify CLI Findings — `./graphify-8/` corpus

**Date:** 2026-06-06 · **Graph:** 6,188 nodes · 11,460 edges · graphifyy 0.8.32
**Question asked:** do the graphify commands from the README uncover meaningful/surprising info about this codebase?

---

## Commands run and what they revealed

| Command | Ran? | Verdict |
|---|---|---|
| `graphify query "<question>"` | ✅ | **Meaningful** — answered architecture questions in ~8.9k tokens instead of ~412k |
| `graphify path "A" "B"` | ✅ | **Surprising** — revealed an unexpected 2-hop bridge (see #2) |
| `graphify explain "X"` | ✅ | **Meaningful** — exposed the most test-covered function in the repo (see #3) |
| `graphify affected "X"` | ✅ | Partial — needs exact node labels (`_make_id()` was ambiguous across files) |
| `graphify cluster-only . --exclude-hubs 99` | ✅ | **Surprising** — found a doc-vs-behavior mismatch (see #1) |
| `graphify export callflow-html` | ✅ | Useful — 18 architecture sections, 16 Mermaid call-flow diagrams |
| `graphify benchmark` | ✅ | Meaningful — 46.5× avg token reduction per query (per-question range 23.6×–165.1×) |
| `graphify diagnose multigraph` | ✅ | Useful — 0 edge-collapse variant groups, 54 producer suppression sites in extract.py |
| `graphify prs` / `prs --triage` / `prs --conflicts` | ❌ | N/A — `graphify-8/` is not a git repo (no `.git`, no GitHub remote) |
| `graphify hook install` | ❌ | N/A — same reason |
| `graphify add <url>` | ⏭️ | Skipped — would mutate the corpus (fetches into `./raw/`) |
| `--update` | ⏭️ | Skipped — nothing changed since the build |

---

## The surprising findings

### 1. `--exclude-hubs 99` does NOT re-rank god nodes (doc ↔ behavior mismatch)

The README claims `--cluster-only --exclude-hubs 99` will *"suppress utility super-hubs
from god-node rankings"*. Empirically, on this graph:

- **Partitioning changed dramatically**: 501 → **971 communities** (hubs removed from
  Leiden partitioning splits the giant clusters into finer ones — this part works).
- **God-node ranking did NOT change**: `Path` (89 edges), `str` (85), `bytes` (56) still
  top the regenerated GRAPH_REPORT.md, exactly as before.

The command-reference section of the same README describes the flag differently
("exclude p99 degree nodes *from partitioning*") — that's the description matching the
observed behavior. Ironically, the repo's own worked examples
(`worked/httpx/review.md`, `worked/rsl-siege-manager/review.md`) document this exact
failure mode: *"degree centrality surfaces non-core nodes"*. The tool knows it has this
problem; the flag that sounds like the fix only fixes clustering, not ranking.

**True god nodes** (builtin/typing hubs filtered out manually):

| # | Node | Edges | Where |
|---|---|---|---|
| 1 | `test_languages.py` | 228 | tests/ |
| 2 | `extract.py` | 203 | graphify/ |
| 3 | `manifest.json` | 167 | worked/rsl-siege-manager/ |
| 4 | `test_detect.py` | 111 | tests/ |
| 5 | `ingest_scip_json()` | 84 | graphify/scip_ingest.py |
| 6 | `gen.py` | 76 | tools/skillgen/ |
| 7 | `callflow_html.py` | 72 | graphify/ |
| 8 | `_make_id()` | 63 | graphify/extract.py |
| 9 | `main()` | 61 | graphify/__main__.py |

So the real architectural centers are **`extract.py`** (the 11k-line AST extractor),
**`ingest_scip_json()`**, **`_make_id()`** (deterministic node IDs — the contract that
keeps AST and LLM extraction mergeable), and **`gen.py`** (the skillgen generator).

### 2. `graphify path "main()" "extract()"` — the CLI reaches extraction through the *watcher*

```
main() --calls [INFERRED]--> _rebuild_code() --calls [INFERRED]--> extract()
```

The shortest path from the CLI entry point to the AST extractor is **2 hops through
`watch.py`'s `_rebuild_code()`** — not through an "extract command" module. The
file-watcher rebuild path is the most direct artery between the CLI and extraction;
the graph also shows `_rebuild_code()` is the #2 betweenness bridge in the whole graph
(0.073), connecting `Watch` to `Extract`, the test suites, security checks
(`check_graph_file_size_cap()` in security.py guards the rebuild), and `callflow_html`.
Watch mode isn't a peripheral feature — structurally it's the spine.

### 3. `graphify explain "ingest_scip_json()"` — the most defensively-tested function in the repo

Degree 84, and the connection list is dominated by **~60 dedicated test functions**
(`test_ingest_relationship_without_target_symbol_is_skipped`,
`test_ingest_duplicate_edges_are_deduplicated`, `test_ingest_node_id_is_deterministic`, …).
One function in `scip_ingest.py` (SCIP code-intelligence index ingestion) carries more
individual edge-case tests than any other symbol in the codebase — a strong signal that
external-index ingestion is where the team has been burned by malformed input.

### 4. `graphify query` — the skillgen anti-drift machine

Querying *"what connects the skillgen generator to the platform skill files"* surfaced
the full `tools/skillgen/gen.py` pipeline in one shot: `load_platforms()` (platforms.toml)
→ `_read_fragment()` → `render()` / `render_all()` / `render_always_on()` →
`RenderedArtifact`, plus `audit_coverage()` and `monolith_roundtrip()` — the functions
that enforce that 20 per-platform skill variants never drift from the shared fragments.
The golden snapshots in `tools/skillgen/expected/` (162 files, ~30% of the whole corpus)
exist purely as this machine's regression net.

### 5. `graphify diagnose multigraph` — clean bill of health, with caveat

0 source-location variant groups and 0 context variant groups (no same-endpoint edge
collapse in the built graph), but **54 producer-side suppression sites** in
`extract.py` (`seen_ids`, `seen_call_pairs`, `seen_swift_base`, …) where duplicate edges
are intentionally dropped before the graph is built — the tool itself notes raw producer
loss must be measured earlier than graph.json.

---

## Token-efficiency evidence (`graphify benchmark`)

```
Corpus:          309,400 words → ~412,533 tokens (naive)
Avg query cost:  ~8,875 tokens   →  46.5× fewer tokens per query
  [ 23.6×] how does authentication work
  [ 81.6×] what is the main entry point
  [165.1×] how are errors handled
  [ 39.4×] what connects the data layer to the api
  [ 46.6×] what are the core abstractions
```

---

## Bottom line

**Yes — three commands uncovered genuinely surprising information:**

1. **`cluster-only --exclude-hubs`** exposed a README-vs-behavior mismatch (flag fixes
   partitioning, not god-node ranking) — a candidate bug report for the repo.
2. **`path`** revealed that the watcher's `_rebuild_code()` is the structural spine
   between the CLI and the extractor, not an add-on feature.
3. **`explain`** identified SCIP ingestion as the single most edge-case-hardened
   function in the codebase.

`query` and `benchmark` are the workhorses (real answers at 23–165× fewer tokens), and
`callflow-html` produced the most readable artifact
(`graphify-out/graphify_chase_ai-callflow.html`). The `prs` family and `hook install`
are inapplicable to this corpus — `graphify-8/` is an exported folder, not a git clone.

## Artifacts produced this session

- `graphify-out/FINDINGS.md` (this file)
- `graphify-out/graphify_chase_ai-callflow.html` — Mermaid architecture/call-flow page
- `graphify-out/GRAPH_REPORT.md` — regenerated with 971 hub-excluded communities
- `graphify-out/backup-run1/` — run-1 report/graph/html (501 labeled communities)
- `graphify-out/2026-06-06/` — automatic pre-recluster backup made by graphify
