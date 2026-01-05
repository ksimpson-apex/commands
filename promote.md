---
description: "Promote PR changes from one environment to other environments"
argument-hint: [pr-url-or-number] [target-envs (optional)]
---

Promote changes from a GitHub Pull Request to other environments.

**Arguments:**
- `$1`: PR identifier - either a full GitHub PR URL or just the PR number
- `$2`: (Optional) Comma or space-delimited list of target environments (e.g., "stg,prd" or "stg prd rcn")

**Current Known Environments:**
- `dev`, `stg`, `sbx`, `uat`, `rcn`, `prd`
- Note: There may be additional environments in the future, so always look for common environment indicators in file paths

**Workflow:**

## Phase 1: Fetch PR Information

1. Parse the PR identifier from `$1`:
   - If it's a URL: Extract the org, repo, and PR number from the URL
   - If it's a number: Get the GitHub remote URL from current git config using `git config --get remote.origin.url`, parse org/repo, and construct PR URL

2. Fetch PR details using GitHub CLI:
   ```bash
   gh pr view <PR_NUMBER> --repo <ORG>/<REPO> --json number,title,headRefName,files
   ```

3. Get the list of changed files from the PR:
   ```bash
   gh pr view <PR_NUMBER> --repo <ORG>/<REPO> --json files --jq '.files[].path'
   ```

## Phase 2: Identify Source Environment

4. Analyze the changed file paths to identify the source environment:
   - Look for known environments in paths: `dev`, `stg`, `sbx`, `uat`, `rcn`, `prd`
   - Also check for common variants: `staging`, `sandbox`, `prod`, `production`
   - Check directory names in the paths (e.g., `config/dev/`, `deployment/stg/`, `k8s/prd/`)
   - The source environment is the one that appears in the changed file paths

5. If "dev" or a clear environment indicator is found in the paths:
   - Set that as the source environment
   - Continue to Phase 3

6. If no clear environment is found in the paths:
   - Use the AskUserQuestion tool to prompt: "Which environment do these changes belong to?"
   - Options: dev, stg, sbx, uat, rcn, prd
   - Note: The user may select "Other" to specify a custom environment name not in the list
   - Use the user's answer as the source environment
   - The user's answer may also include multiple environments delimited by comma or space

## Phase 3: Discover Available Environments

7. For each changed file path, identify the parent directory containing the environment:
   - Example: `config/dev/service.yaml` → parent is `config/`
   - Example: `k8s/overlays/dev/deployment.yaml` → parent is `k8s/overlays/`

8. List sibling directories to discover other environments:
   ```bash
   ls -1 <parent_directory>
   ```

9. Filter the results to identify environment directories:
   - Look for known environment names: dev, stg, sbx, uat, rcn, prd
   - Also check for common variants: staging, sandbox, prod, production, qa
   - Build a list of discovered environments
   - Exclude the source environment from this list

## Phase 4: Determine Target Environments

10. If `$2` is provided:
    - Parse the comma or space-delimited list
    - Validate each environment exists in the discovered environments
    - If any environment is not found, warn the user but continue with valid ones
    - Use this as the target environments list

11. If `$2` is NOT provided:
    - Use ALL discovered environments (excluding the source environment) as targets

## Phase 5: Determine Environment Order

**Environment Ranking:** `dev` → `stg` → `sbx` → `uat` → `rcn` → `prd`

12. Determine the order in which to create PRs:
    - If ALL target environments are in the Current Known Environments list (`dev`, `stg`, `sbx`, `uat`, `rcn`, `prd`):
      - Use the ranking order above, filtering to only the target environments
      - Example: If targets are `[stg, prd, uat]`, order becomes `[stg, uat, prd]`
    - If ANY target environment is NOT in the Current Known Environments list:
      - Identify the new environments (those not in the known list)
      - Use AskUserQuestion to prompt: "Where should '<new_env>' be ranked in the environment order?"
      - Options: Show the current known environments and ask where to insert the new one
      - Example: "Before dev", "Between stg and sbx", "Between uat and rcn", "After prd"
      - Repeat for each new environment
      - Build the complete ranking including both known and new environments
      - Use AskUserQuestion to ask: "Should this ranking be temporary or permanent?"
      - Options: "Temporary (for this session only)", "Permanent (update the command file)"
      - If "Permanent":
        - Use the Edit tool to update this file's "Current Known Environments" section
        - Add the new environment(s) to the ranked list in step 82
        - Update the environment list in step 48 to include the new environment(s)

## Phase 6: Fetch Source PR Body

13. Fetch the source PR body and title:
    ```bash
    gh pr view <PR_NUMBER> --repo <ORG>/<REPO> --json body,title
    ```

14. Parse the source PR body to identify sections that reference other environment PRs:
    - Look for patterns like "Lower environment PRs:", "Related PRs:", "Dependencies:", etc.
    - These sections may contain links to PRs in other environments
    - We'll update these sections when creating new PRs

## Phase 7: Create PRs for Target Environments

15. Prompt for branch naming using AskUserQuestion:
    - First question: "What JIRA ticket should be used as the prefix for the branch names?"
      - No default provided - user must specify (e.g., "trex-4968")
    - Second question: "What description should be used in the branch names?"
      - No default provided - user must specify (e.g., "event-contract", "new-feature")
    - Store these as JIRA_PREFIX and DESCRIPTION
    - Always convert both to lowercase when constructing branch names

For each target environment **in the determined order**:

16. Create a new branch:
    - Construct branch name: `<JIRA_PREFIX>-<DESCRIPTION>-<TARGET_ENV>` (all lowercase)
    - Example: `trex-4968-event-contract-stg`
    ```bash
    git checkout main
    git checkout -b <lowercase_branch_name>
    ```

17. For each changed file from the source PR:
    - Determine the equivalent file path in the target environment by replacing the source environment name with the target environment name
    - Example: `config/dev/service.yaml` → `config/stg/service.yaml`

18. Fetch the diff from the source PR to understand what changed:
    ```bash
    gh pr diff <PR_NUMBER> --repo <ORG>/<REPO>
    ```

19. Apply changes to the target environment files using ONLY the Edit tool:
    - **CRITICAL:** NEVER copy files directly from one environment to another
    - **CRITICAL:** NEVER use sed commands or bash scripts to apply changes
    - **CRITICAL:** ALWAYS use the Read tool first to read the existing target environment file
    - **CRITICAL:** ALWAYS use the Edit tool to apply specific changes from the source PR diff to the target file
    - The Edit tool allows precise string replacements that preserve environment-specific differences
    - If a line needs to be added, use Edit to insert it at the correct location
    - If a line needs to be removed, use Edit to remove it
    - Read the target environment file first, then apply each change individually using Edit
    - Repeat for each file that needs to be updated

20. Commit the changes:
    ```bash
    git add .
    git commit -m "Promote changes from PR #<PR_NUMBER> to <target_env>

    Original PR: <PR_URL>
    Source environment: <source_env>
    Target environment: <target_env>

    🤖 Generated with [Claude Code](https://claude.com/claude-code)

    Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
    ```

21. Push the branch:
    ```bash
    git push -u origin <lowercase_branch_name>
    ```

22. Prepare the PR title:
    - Take the source PR title
    - Replace any mention of the source environment with the target environment
    - Common patterns to replace:
      - "to dev environment" → "to <target_env> environment"
      - "in dev" → "in <target_env>"
      - "dev environment" → "<target_env> environment"
      - "for dev" → "for <target_env>"
    - Example: "TREX-4908: Add market-simulator-v2 project to dev environment" → "TREX-4908: Add market-simulator-v2 project to stg environment"

23. Prepare the PR body:
    - Start with the source PR body as a template
    - Look for sections that reference other environment PRs (e.g., "Lower environment PRs:", "Related PRs:", "Dependencies:")
    - If found, update these sections to include a bullet list of all lower environment PRs with:
      - **FIRST:** Always list the original source PR (e.g., `dev (#8792)`)
      - **THEN:** List all intermediate environment PRs in order (e.g., `stg (#8889)`, `uat (#8891)`)
    - If no such sections exist in the source PR, append a new section at the end with the exact header:
      ```
      Link to merged PR in lower environment (if not dev)
      ```
    - Under this header, add a bullet list (hyphen-prefixed) of all lower environment PRs:
      - First bullet: original source PR
      - Subsequent bullets: intermediate environment PRs in order
    - Example for a `prd` PR with original `dev` (#8792) and intermediate `stg` (#8889), `uat` (#8891):
      ```
      Link to merged PR in lower environment (if not dev)
      - dev (#8792)
      - stg (#8889)
      - uat (#8891)
      ```

24. Create the PR in draft mode:
    ```bash
    gh pr create \
      --draft \
      --title "<PREPARED_PR_TITLE>" \
      --body "<PREPARED_PR_BODY>"
    ```

25. Store the created PR URL and number for use in subsequent iterations (for updating the lower environment PRs bullet list)

26. Return to the original branch:
    ```bash
    git checkout main
    ```

## Phase 8: Summary

27. Display a summary of created PRs in a format that can be pasted into Slack:
    - Use checkbox format: `[ ]` followed by environment name and clickable PR URL
    - Order PRs by environment rank (stg → sbx → uat → rcn → prd)
    - Format example:
      ```
      [ ] stg: https://github.com/org/repo/pull/8889
      [ ] sbx: https://github.com/org/repo/pull/8890
      [ ] uat: https://github.com/org/repo/pull/8891
      [ ] rcn: https://github.com/org/repo/pull/8892
      [ ] prd: https://github.com/org/repo/pull/8893
      ```
    - This format allows users to check off PRs as they are reviewed/merged in Slack

---

**Important Notes:**
- **CRITICAL:** NEVER copy files from one environment to another - always use Edit tool to apply changes
- **CRITICAL:** NEVER use sed commands or bash scripts to apply changes - only use Edit tool
- **CRITICAL:** Always prompt user for JIRA ticket prefix and description before creating branches
- **CRITICAL:** Always convert branch names to lowercase
- **CRITICAL:** Always update the PR title to replace the source environment with the target environment
- **CRITICAL:** Always list the original source PR first in the lower environment PRs bullet list
- The command should handle cases where target environment files don't exist yet
- Preserve any environment-specific differences (e.g., don't blindly copy URLs or secrets)
- Use TodoWrite to track progress through the phases
- Handle errors gracefully (e.g., if a target directory doesn't exist, skip that environment with a warning)

Execute this workflow for PR `$1` with target environments `$2`.
