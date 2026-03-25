# Agora Anti-Patterns Reference

| Anti-pattern | Correct approach |
|-------------|-----------------|
| Role repeats the same objection | Mark "Already addressed in Round N", reference prior response |
| Accept all objections wholesale | Evaluate each independently, only accept with good reason |
| Dismissive responses | Give specific reasoning for each objection — never just "noted" |
| Blind continuation after parse failure | Default to "Objection" + flag warning in terminal |
| Missing summary/key points | Use first 100 chars / objection list as fallback |
| Give up after Agent call failure | Retry once, then skip with terminal warning |
| Accepted changes not applied to code | Code file changes must be immediately written |
| Removing role without notification | Output warning in terminal + record in file |
| Main Claude unilaterally converging | Convergence is a structural rule — 2 consecutive rounds of all-no-objection |
| Biased status parsing | Preserve raw status lines verbatim in file for auditing |
| Tracker growing without bound | Collapse resolved/rejected entries older than 2 rounds |
