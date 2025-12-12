# Claude Code Custom Commands

A collection of custom slash commands for [Claude Code](https://claude.ai/code).

## Table of Contents

- [Installation](#installation)
  - [For users WITHOUT existing commands](#for-users-without-existing-commands)
  - [For users WITH existing commands](#for-users-with-existing-commands)
  - [Alternative installation](#alternative-installation-for-users-with-existing-commands)
- [Available Commands](#available-commands)
- [Contributing](#contributing)
- [Forking and Syncing](#forking-and-syncing)
  - [Fork this repository](#fork-this-repository)
  - [Keep your fork updated](#keep-your-fork-updated)

## Installation

### For users WITHOUT existing `~/.claude/commands/`

```bash
cd ~/.claude
git clone https://github.com/ksimpson-apex/commands.git
```

### For users WITH existing commands

```bash
# Temporarily rename existing
mv ~/.claude/commands ~/.claude/commands-existing

# Clone fresh
cd ~/.claude
git clone https://github.com/ksimpson-apex/commands.git

# Copy back any personal commands
cp ~/.claude/commands-existing/*.md ~/.claude/commands/
```

> **Note:** If you want to maintain your own fork and sync updates, see [Forking and Syncing](#forking-and-syncing) below instead of using this installation method.

### Alternative installation (for users with existing commands)

> **Warning:** This method is mutually exclusive with [Forking and Syncing](#forking-and-syncing). Use this only if you don't plan to fork the repository.

```bash
cd ~/.claude/commands
git init
git remote add origin https://github.com/ksimpson-apex/commands.git
git fetch origin
git checkout main
```

Note: This may fail if there are conflicting filenames between your existing commands and the repository.

## Available Commands

<details>
<summary><code>/promote</code> - Promote PR changes between environments</summary>

Promote PR changes from one environment to other environments.

**Usage:**
```bash
/promote <pr-url-or-number> [target-envs]
```

**Examples:**
```bash
/promote 12345                           # Promote PR #12345 to all environments
/promote https://github.com/org/repo/pull/12345  # Using full URL
/promote 12345 stg,prd                   # Only promote to staging and production
/promote 12345 stg prd rcn               # Space-delimited also works
```

**What it does:**
1. Fetches PR details and changed files using GitHub CLI
2. Identifies the source environment from file paths
3. Discovers other environments by checking sibling directories
4. Creates PRs for each target environment in ranked order
5. Each PR references lower environment PRs for easy tracking

**Supported environments:** dev, stg, sbx, uat, rcn, prd

See [promote.md](./promote.md) for detailed workflow documentation.

</details>

<details>
<summary><code>/stg_deploy</code> - Approve staging gitops PRs</summary>

Check the staging gitops PR and approve the PRs if they're current.

**Source:** [Useful Tips and Tricks for Claude](https://apexclearing.atlassian.net/wiki/spaces/TRDX/pages/8754135186/Useful+Tips+and+Tricks+for+Claude#Approving-staged-PRs-for-staging)

**What it does:**
- Finds PRs opened in the last 3 days that contain "to stg" for specific services
- Validates orchestrator PRs contain `orch-trading` OR `orch-assettradingconfig`
- Compares version strings in manifests to dev versions
- Approves PRs if version strings match
- Reports approved PRs and any that were skipped with reasons

**Targeted services:**
- asset-trading-cfg
- fractional-liquidations
- open-orders
- orchestrator
- orders-service
- svc-config-admin
- trade-simulator

See [stg_deploy.md](./stg_deploy.md) for detailed implementation.

</details>

<details>
<summary><code>/pr_review</code> - Expert code review</summary>

Expert code review specialist for quality, security, and maintainability.

**Source:** [Useful Tips and Tricks for Claude](https://apexclearing.atlassian.net/wiki/spaces/TRDX/pages/8754135186/Useful+Tips+and+Tricks+for+Claude#PR-Review)

**Usage:**
```bash
/pr_review <github-pr-url>
```

**Example:**
```bash
/pr_review https://github.com/apex-fintech-solutions/source/pull/123456
```

**Review checklist:**
- Code is simple and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented
- Good test coverage
- Performance considerations addressed

**Focus areas:**
- Logic errors and side effect risks
- Correctness issues
- Agreement with PR description and JIRA tickets
- Security vulnerabilities

See [pr_review.md](./pr_review.md) for detailed implementation.

</details>

## Contributing

Feel free to submit issues or pull requests for new commands or improvements.

## Forking and Syncing

> **Note:** This approach allows you to maintain your own personal commands while syncing updates from this repository. Your personal commands won't be clobbered when syncing.

### Fork this repository

1. Fork on GitHub (click the "Fork" button at the top right of the repository page)

2. Back up existing commands (if you have any):
```bash
# Only if ~/.claude/commands already exists
mv ~/.claude/commands ~/.claude/commands-existing
```

3. Clone your fork:
```bash
cd ~/.claude
git clone https://github.com/YOUR_USERNAME/commands.git
```

4. Add the upstream remote:
```bash
cd ~/.claude/commands
git remote add upstream https://github.com/ksimpson-apex/commands.git
```

5. (Optional) Add your personal commands to your fork:
```bash
# Copy any existing personal commands
cp ~/.claude/commands-existing/*.md ~/.claude/commands/

# Commit them to your fork
git add *.md
git commit -m "Add personal commands"
git push origin main
```

### Keep your fork updated

Sync your fork with the latest changes from the upstream repository:

```bash
# Fetch the latest changes from upstream
git fetch upstream

# Make sure you're on your main branch
git checkout main

# Merge upstream changes into your fork
git merge upstream/main

# Push the updates to your fork on GitHub
git push origin main
```

**Alternative: Rebase instead of merge**

If you prefer a cleaner history without merge commits:

```bash
# Fetch the latest changes from upstream
git fetch upstream

# Make sure you're on your main branch
git checkout main

# Rebase your changes on top of upstream
git rebase upstream/main

# Push the updates (may require force push if you've rebased)
git push origin main --force-with-lease
```
