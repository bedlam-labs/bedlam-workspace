# Contributing to Bedlam Workspace

## Getting Started

1. Clone with submodules:
   ```sh
   git clone --recurse-submodules <REPO_URL>
   cd bedlam-workspace
   pnpm install
   ```

2. Or set up selectively:
   ```sh
   pnpm run setup:agent        # agent + pathfinder + viewer
   pnpm run setup:test-suite   # test-suite + viewer
   pnpm run setup:all          # everything
   ```

## Making Changes

Each package under `packages/` is its own git repository (submodule). Work inside the package directory and commit there.

```sh
cd packages/agent
# make changes
git add . && git commit -m "feat: description"
git push
```

Or use the shortcut scripts from the workspace root:
```sh
pnpm run commit:agent
pnpm run commit:pathfinder
pnpm run commit:test-suite
pnpm run commit:viewer
```

These stage, commit, and bump the parent submodule pointer in one step.

## Pull Requests

- Open PRs against the individual package repo, not the workspace repo.
- Keep PRs focused — one feature or fix per PR.
- Run tests before submitting: `pnpm --filter bedwars-agent test`
- Build to verify: `pnpm -r build`

## Code Standards

- Break down large functions. One responsibility per function.
- Favor descriptive names over comments.
- Only comment the *why*, not the *what*.
- Remove dead code, unused imports, and commented-out code.
- Do not use filepath-based imports across packages — use the `workspace:*` protocol.

## Reporting Issues

Open issues on the relevant package repository, not the workspace repo.
