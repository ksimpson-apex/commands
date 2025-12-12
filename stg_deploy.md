---
allowed-tools: ["Bash(gh pr list *)", "Bash(gh pr view *)", "Bash(gh pr diff *)", "Bash(gh pr review *)", "Bash(gh api *)", "Bash(jq *)", "Bash(grep *)", "Bash(echo *)", "Bash(date *)", "Read", "WebFetch"]
description: Check the staging gitops PR and approve the PRs if they're current
---

## Context

Deployment PRs are in the https://github.com/apex-fintech-solutions/gitops/ repo.

Note that most users will be on MacOS, so make sure to use the appropriate syntax for the command line tools.

Use the gh cli tool for github interactions.  Do not use python or command line loops.

## Your Task

* Find PRs opened in the last 3 days that contain "to stg" and one of these strings in the title:
  * asset-trading-cfg
  * fractional-liquidations
  * open-orders
  * orchestrator (and limit this to those that contain `orch-trading` OR `orch-assettradingconfig`)
  * orders-service
  * svc-config-admin
  * trade-simulator
  I think a command like `gh pr list --repo apex-fintech-solutions/gitops --state open --search "\"to stg\" created:>=YYYY-MM-DD" --json title,number,url` (with the appropriate values for YYYY-MM-DD) will be the best command for this
* Make sure that the orchestrator PRs contain `orch-trading` OR `orch-assettradingconfig`.  If they do not, do not include them in further steps.
* Note any fn-config.yaml diffs and keep track of them, but do not consider while deciding if a PR should be approved or not.
* Find the version string(s) in each of the manfifests and compare them to the dev version of the manifests.  The version should be in the  metadata.labels.[tags.datadoghq.com/version]
* If the version string(s) are the same, approve the PRs
* Print out links to the PRs that were approved, as well as a list of those that were not, highlighting the differences that caused them not to be approved.  Also include details about any fn-config.yaml differences previously tracked.
