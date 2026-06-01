Phase 1: Create the Parent Shell
First, we are going to create the empty orchestrator repository. This repo won't hold any actual code; it just holds your configuration, CI/CD pipelines, and git pointers.

Open your terminal and create a new directory:

Bash
mkdir my-workspace
cd my-workspace
git init
Create a pnpm-workspace.yaml file at the root to tell pnpm where your projects will live:

YAML
packages:
  - 'packages/*'
Initialize a basic package.json at the root. This is where we will define your onboarding scripts later:

Bash
pnpm init
Phase 2: Attach Your Existing Repositories
Now, we link your existing agent and test-suite repositories as Git Submodules.

Add them into the packages/ directory:

Bash
git submodule add <URL_TO_AGENT_REPO> packages/agent
git submodule add <URL_TO_TEST_SUITE_REPO> packages/test-suite
(Note: This pulls down the code and creates a .gitmodules file at your root, which tracks these connections).

Make sure your agent/package.json points to the test suite using the workspace protocol:

JSON
{
  "name": "agent",
  "dependencies": {
    "test-suite": "workspace:*"
  }
}
Run a global install to generate your unified lockfile:

Bash
pnpm install
Commit the shell repository:

Bash
git add .
git commit -m "chore: initialize parent workspace and submodules"
git branch -M main
git remote add origin <URL_TO_NEW_PARENT_REPO>
git push -u origin main
Phase 3: Create the Onboarding Automation Scripts
To solve the "dependency cloning" problem, we will write explicit setup scripts in the root package.json. This ensures that if a developer wants to work on the agent, the script automatically pulls down the agent and the test-suite it relies on.

Update your root package.json:

JSON
{
  "name": "my-workspace",
  "version": "1.0.0",
  "scripts": {
    "setup:all": "git submodule update --init --recursive && pnpm install",
    "setup:agent": "git submodule update --init packages/agent packages/test-suite && pnpm install",
    "setup:test-suite": "git submodule update --init packages/test-suite && pnpm install"
  }
}
Now, your developers don't need to guess what relies on what. The scripts handle the git module initialization and dependency linking in one breath.

Phase 4: Configure the CI/CD Pipeline
Because the parent repo now controls the state of your application, your CI/CD pipeline lives here.

Create .github/workflows/ci.yml in the root of your workspace:

YAML
name: Global CI Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Codebase
        uses: actions/checkout@v4
        with:
          submodules: recursive # Instantly pulls all linked code
          # token: ${{ secrets.GH_PAT }} # Uncomment if submodules are private

      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - name: Install Dependencies
        run: pnpm install --frozen-lockfile

      - name: Run Agent Tests
        run: pnpm --filter agent test
Phase 5: The Final Developer Experience
Your setup is completely finished. Here is exactly what the workflow looks like for a brand new developer joining your team tomorrow:

If they want to work on EVERYTHING:

Bash
git clone <URL_TO_PARENT_REPO> my-workspace
cd my-workspace
pnpm run setup:all
If they ONLY want to work on the Agent (and its required dependencies):

Bash
git clone <URL_TO_PARENT_REPO> my-workspace
cd my-workspace
pnpm run setup:agent
They are immediately ready to code. If they make changes inside packages/agent, they use standard git commands (git add, git commit, git push) entirely within that folder, and those changes push directly to the independent Agent repository.