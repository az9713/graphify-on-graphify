# Evidence captures

Raw, unedited stdout of graphify CLI commands run against the committed
`graphify-out/graph.json` (the post-`--exclude-hubs` graph, 6,188 nodes / 11,460 edges /
971 communities). Captured 2026-06-06 with the version in `version.txt`.

Every file here is reproducible by anyone: clone this repo, `uv tool install graphifyy`,
`cd` to the repo root, and re-run the command named in each file. Queries/paths/explains
are pure local graph traversals — no API key, no LLM, no network.

| File | Command | Supports claim |
|---|---|---|
| `version.txt` | `graphify --version` | environment pin |
| `benchmark.txt` | `graphify benchmark` | TOKEN_SAVINGS.md — 46.5× printed figure (and the 309,400 = 6,188 × 50 synthetic-words artifact) |
| `path_main_to_extract.txt` | `graphify path "main()" "extract()"` | FINDINGS.md #2 — CLI→extractor shortest path runs through the watcher |
| `path_skillgen_to_skill.txt` | `graphify path "Skillgen Generator (gen.py)" "Graphify Skill (canonical /graphify skill template)"` | FINDINGS.md — doc/code clusters disconnected ("No path found") |
| `explain_ingest_scip_json.txt` | `graphify explain "ingest_scip_json()"` | FINDINGS.md #3 — degree 84, test-dominated neighborhood |
| `affected_make_id.txt` | `graphify affected "_make_id()"` | COMMANDS_LOG.md — ambiguous-label lookup limitation |
| `diagnose_multigraph.txt` | `graphify diagnose multigraph --max-examples 3` | COMMANDS_LOG.md #23 — 0 collapse groups, 54 suppression sites |
| `query_skillgen.txt` | `graphify query "what connects the skillgen generator to the platform skill files" --budget 1200` | FINDINGS.md #4 — skillgen anti-drift pipeline |
| `query_watch_mode.txt` | `graphify query "how does watch mode trigger graph rebuilds" --budget 1200` | FINDINGS.md #2 — watch rebuild chain incl. security gate |
| `query_cache.txt` | `graphify query "what is the relationship between the extraction cache and semantic subagents" --budget 1200` | COMMANDS_LOG.md #16 — vocabulary-miss example (honest negative) |

## Evidence for the `--exclude-hubs` finding (FINDINGS.md #1)

That claim compares two pipeline runs, so its evidence is the committed pair of artifacts:

- **Run 1** (before): `graphify-out/backup-run1/GRAPH_REPORT.md` + `backup-run1/graph.json`
  — 501 communities, god nodes led by `Path` (89) / `str` (85)
- **Run 2** (after `graphify cluster-only . --graph graphify-out/graph.json
  --exclude-hubs 99 --no-label --no-viz`): `graphify-out/GRAPH_REPORT.md` +
  `graphify-out/graph.json` — 971 communities, god-node list **unchanged**

Recompute degrees yourself from either graph.json: node degree ranking is identical in
both; only the community partition differs.

## Evidence for the build-cost figure (TOKEN_SAVINGS.md)

`graphify-out/cost.json` — written by the pipeline at build time:
1,533,913 input tokens, 540 files, one run.
