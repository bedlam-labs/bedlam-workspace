## Workspace Structure

| Package | Description |
|---------|-------------|
| `packages/agent` | Bedwars LLM agent — Coordinator, tools, AgentLoop, LLM client |
| `packages/mineflayer-pathfinder` | A* pathfinding plugin for Mineflayer bots |
| `packages/mineflayer-test-suite` | Dev server + scenario runner for testing agent behavior in-world |
| `packages/prismarine-viewer` | 3D Minecraft world viewer (Three.js, observation/visualization) |

## Coding Guidelines

1. Modular Design: Break down large functions. Focus on SRP (Single Responsibility Principle).
2. No Spaghetti Code: Ensure code blocks are easy to follow and avoid nested, complex logic.
3. Self-Documenting Code: Favor descriptive variable and function names over comments.
4. Comments Policy: Do not add comments explaining what code does (if the code is clear), only why a non-obvious approach was taken.
5. Cleanliness: Remove dead code, unused imports, and commented-out code instantly.
6. Workspace Layout: JS packages live under packages/ as git submodules in a pnpm workspace. Cross-package deps use workspace:* protocol. Do not use filepath-based imports across packages.
7. Submodule Commits: Each package under packages/ is its own repo. Commit changes inside the submodule first, then update the submodule pointer in the parent.

## Karpathy-Inspired Principles

Derived from [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.

**Think Before Coding** — Don't assume. State assumptions explicitly. Present multiple interpretations when ambiguity exists. Push back if a simpler approach exists. Stop and ask when confused.

**Simplicity First** — Minimum code that solves the problem. No speculative features, no abstractions for single-use code, no error handling for impossible scenarios. If 200 lines could be 50, rewrite it. Test: would a senior engineer call this overcomplicated?

**Surgical Changes** — Touch only what the task requires. Don't improve adjacent code, comments, or formatting. Match existing style. If unrelated dead code is noticed, mention it — don't delete it. Every changed line should trace directly to the user's request.

**Goal-Driven Execution** — Define success criteria, then loop until verified. Transform imperative tasks into declarable goals:

- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

These principles bias toward caution over speed. Use judgment for trivial one-liners; apply full rigor for non-trivial work.

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
