# Agora Pre-Release Checklist

1. **Agent tool model parameter validation**: run an experiment — give an Agent a task requiring deep reasoning, compare result quality with and without `model: "opus"`. Do this once during development.
2. **End-to-end multi-round test**: run a 5-round test task with the current setup. Pass criteria: "discussion file structure intact, all rounds parseable, final Proposal generated successfully."
3. **Bash append reliability**: verify that `cat >> file` appends correctly on large files (100KB+).
4. **Large file Edit reliability**: verify that the Edit tool can precisely modify the single-line HTML comment at the top of large files (100KB+). If unreliable, move discussion_state to a separate JSON file.
