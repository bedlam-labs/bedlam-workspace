1. Modular Design: Break down large functions. Focus on SRP (Single Responsibility Principle).
2. No Spaghetti Code: Ensure code blocks are easy to follow and avoid nested, complex logic.
3. Self-Documenting Code: Favor descriptive variable and function names over comments.
4. Comments Policy: Do not add comments explaining what code does (if the code is clear), only why a non-obvious approach was taken.
5. Cleanliness: Remove dead code, unused imports, and commented-out code instantly.
6. Workspace Layout: JS packages live under packages/ as git submodules in a pnpm workspace. Cross-package deps use workspace:* protocol. Do not use filepath-based imports across packages.
7. Submodule Commits: Each package under packages/ is its own repo. Commit changes inside the submodule first, then update the submodule pointer in the parent.
