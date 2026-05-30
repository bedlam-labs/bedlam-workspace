# bedwars-agents

Research platform for studying emergent behavior in competitive multi-agent LLM environments, built on Minecraft Bedwars.

---

## What This Is

This project puts multiple LLM-driven agents into a competitive Minecraft Bedwars environment and studies what happens. The game provides a high-pressure, real-time, multi-objective scenario where agents must coordinate with teammates, compete against opponents, manage resources, and make strategic decisions — all simultaneously.

The goal is not to build the best Bedwars bot. The goal is to use Bedwars as a research environment to ask harder questions about how language models behave under competitive pressure.

---

## Workspace structure

pnpm workspace with git submodules under `packages/`:
- `packages/agent` — the bot
- `packages/mineflayer-test-suite` — test library
- `packages/prismarine-viewer` — renderer fork
- `packages/mineflayer-pathfinder` — pathfinding fork

Cross-package dependencies use `workspace:*` protocol — pnpm symlinks them automatically.

---

## First-time setup

```sh
git clone --recurse-submodules <WORKSPACE_REPO_URL>
cd bedlam-workspace
pnpm install
```

Or selectively:
```sh
pnpm run setup:agent        # agent + pathfinder + viewer
pnpm run setup:test-suite   # test-suite + viewer
pnpm run setup:all          # everything
```

---

## Scripts

| Task | Command |
|---|---|
| Run agent | `pnpm --filter bedwars-agent start` |
| Build all | `pnpm -r build` |
| Test agent | `pnpm --filter bedwars-agent test` |

---

## Per-package behavior

### `prismarine-viewer`
Plain JS — no build needed. Edits are live immediately via workspace symlink.

### `mineflayer-test-suite`
TypeScript source. Run `pnpm --filter mineflayer-test-suite build` after changes, or use ts-node for development.

### `mineflayer-pathfinder`
Plain JS — no build needed. Edits are live immediately via workspace symlink.

### `agent`
TypeScript. Use `pnpm --filter bedwars-agent start` to run via ts-node.

---

## Submodule workflow

Changes inside `packages/<name>/` push to that package's own repo. The parent workspace tracks which commit each submodule points to.

### Commit shortcuts

Stages submodule changes, prompts for commit message, then auto-bumps the parent pointer:

| Package | Command |
|---|---|
| agent | `pnpm run commit:agent` |
| mineflayer-pathfinder | `pnpm run commit:pathfinder` |
| mineflayer-test-suite | `pnpm run commit:test-suite` |
| prismarine-viewer | `pnpm run commit:viewer` |

### Manual workflow

```sh
cd packages/agent
git add . && git commit -m "feat: ..."
git push

cd ../..
git add packages/agent
git commit -m "chore: bump agent submodule"
```
