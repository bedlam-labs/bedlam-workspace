# Graphify — Knowledge Graph Workflow

Graphify turns this codebase into a queryable knowledge graph. Use it to explore architecture, trace data flows, and ask questions about the code without reading every file.

## Quick Start

```bash
# Build the graph (first time or after major changes)
/graphify packages/agent packages/mineflayer-pathfinder packages/mineflayer-test-suite

# Ask a question
/graphify query "how does StartCombat communicate with the RL server"

# Open the interactive graph
open graphify-out/graph.html
```

## Outputs

| File | Purpose |
|------|---------|
| `graphify-out/graph.html` | Interactive graph — open in browser, no server needed |
| `graphify-out/graph.json` | Raw graph data (nodes + edges) |
| `graphify-out/GRAPH_REPORT.md` | God nodes, surprising connections, suggested questions |
| `graphify-out/cost.json` | Cumulative token usage across runs |

## What to Scope

**Do not run on the full repo root.** `packages/prismarine-viewer/public/` contains ~31k PNG textures and versioned Minecraft block JSON data that bloat the graph with noise.

Run on specific packages instead:

```bash
/graphify packages/agent
/graphify packages/mineflayer-pathfinder
/graphify packages/mineflayer-test-suite
# or all three at once:
/graphify packages/agent packages/mineflayer-pathfinder packages/mineflayer-test-suite
```

For `prismarine-viewer`, point at source subdirs only — skip `public/` entirely:

```bash
/graphify packages/prismarine-viewer/viewer packages/prismarine-viewer/lib
```

This avoids `public/` which contains ~31k PNGs, versioned block JSON, and texture atlases.

## Keeping the Graph Fresh

After adding or modifying files, use incremental update (no LLM cost for code-only changes):

```bash
/graphify packages/agent --update
```

For new docs or major structural changes, rebuild:

```bash
/graphify packages/agent packages/mineflayer-pathfinder packages/mineflayer-test-suite
```

## Querying

```bash
# Broad context (BFS — what connects to X)
/graphify query "how does the AgentLoop work"

# Trace a specific path (DFS)
/graphify query "how does MoveTo reach the pathfinder" --dfs

# Shortest path between two concepts
/graphify path "AgentLoop" "StartCombat"

# Explain a single node
/graphify explain "Coordinator"
```

## Pruning Noise

If the graph contains unwanted data (e.g. block state JSON files), prune in-place without rebuilding:

```python
# Remove nodes matching a pattern from graphify-out/graph.json, then:
graphify export html
```

See the pruning script in session history or re-run `/graphify` on a tighter scope.

## Token Cost

Semantic extraction uses LLM subagents — one per ~22-file chunk. Code-only changes after `--update` skip LLM entirely. Set `GEMINI_API_KEY` to use Gemini Flash instead of Claude subagents (cheaper for large corpora).

Current run costs tracked in `graphify-out/cost.json`.
