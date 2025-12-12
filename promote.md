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

For each target environment **in the determined order**:

15. Create a new branch:
    - Use the branch from source PR as the foundation of new branch name
    - The source branch will likely start with a Jira ticket, for example "trex-1234". This is the JIRA_PREFIX.
    - What follows will be the remainder of the new branch name with the target environment appended following a hyphen. This is the REMAINDER. If REMAINDER contains the source environment, the new branch name will replace source environment with target environment *instead of* appending.
    ```bash
    git checkout -b <JIRA_PREFIX>-<REMAINDER>
    ```

16. For each changed file from the source PR:
    - Determine the equivalent file path in the target environment by replacing the source environment name with the target environment name
    - Example: `config/dev/service.yaml` → `config/stg/service.yaml`

17. Fetch the actual file content from the source PR:
    ```bash
    gh pr view <PR_NUMBER> --repo <ORG>/<REPO> --json files --jq '.files[] | select(.path=="<file_path>")'
    ```

    Then fetch the actual content from the PR branch:
    ```bash
    git fetch origin <PR_HEAD_REF>
    git show origin/<PR_HEAD_REF>:<file_path>
    ```

18. Apply the changes to the target environment files:
    - Use the Edit or Write tool to update the target file with the content from the source file
    - If the file doesn't exist in the target environment, create it

19. Commit the changes:
    ```bash
    git add .
    git commit -m "Promote changes from PR #<PR_NUMBER> to <target_env>

    Original PR: <PR_URL>
    Source environment: <source_env>
    Target environment: <target_env>

    🤖 Generated with [Claude Code](https://claude.com/claude-code)

    Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
    ```

20. Push the branch:
    ```bash
    git push -u origin <JIRA_PREFIX>-<REMAINDER>
    ```

21. Prepare the PR body:
    - Start with the source PR body as a template
    - Look for sections that reference other environment PRs (e.g., "Lower environment PRs:", "Related PRs:", "Dependencies:")
    - If found, update these sections to include links to PRs created in previous iterations (lower-ranked environments)
    - Example: If creating a PR for `prd` and already created PRs for `stg` and `uat`:
      ```
      Lower environment PRs:
      - stg: #<STG_PR_NUMBER>
      - uat: #<UAT_PR_NUMBER>
      ```
    - If no such sections exist in the source PR, append a new section at the end:
      ```
      ---
      **Promoted from:** #<SOURCE_PR_NUMBER> (<source_env>)
      **Lower environment PRs:**
      <list of PRs created for environments ranked lower than current target>
      ```

22. Create the PR in draft mode:
    ```bash
    gh pr create \
      --draft \
      --title "<SOURCE_PR_TITLE>" \
      --body "<PREPARED_PR_BODY>"
    ```

23. Store the created PR URL and number for use in subsequent iterations (for updating "Lower environment PRs" sections)

24. Return to the original branch:
    ```bash
    git checkout -
    ```

## Phase 8: Summary

25. Display a summary of created PRs with their URLs for easy access, ordered by environment rank.

---

**Important Notes:**
- The command should handle cases where target environment files don't exist yet
- Preserve any environment-specific differences (e.g., don't blindly copy URLs or secrets)
- Use TodoWrite to track progress through the phases
- Handle errors gracefully (e.g., if a target directory doesn't exist, skip that environment with a warning)

Execute this workflow for PR `$1` with target environments `$2`.
