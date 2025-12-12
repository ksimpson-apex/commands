# Claude Code Custom Commands

A collection of custom slash commands for [Claude Code](https://claude.ai/code).

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

### Alternative installation (for users with existing commands)

```bash
cd ~/.claude/commands
git init
git remote add origin https://github.com/ksimpson-apex/commands.git
git fetch origin
git checkout main
```

Note: This may fail if there are conflicting filenames between your existing commands and the repository.

## Available Commands

### `/promote`

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

## Contributing

Feel free to submit issues or pull requests for new commands or improvements.
