Arguments: $ARGUMENTS

If the arguments contain "help":
- Read all agent files from ~/.claude/agents/
- For each agent file, extract the name, description, and focus areas
- Display them in a table with columns: Agent | Repository | Focus Area
- Format as a markdown table similar to:

| Agent | Repository | Focus Area |
|-------|------------|------------|
| src1 | ~/code/source | trading folder |
| ... | ... | ... |

Otherwise:
Create an agent team for a collaborative task.

Which agents should be on the team? (e.g., "src1 and src2" or "tf, dbt, and comet")

What should the team work on?
