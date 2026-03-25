---
description: Launch Agora — multi-role AI debate to generate high-quality proposals
argument-hint: "<task description> [--roles athena,momus] [--exclude hephaestus] [--max-rounds 10]"
allowed-tools: [Read, Write, Edit, Glob, Grep, Agent, Bash]
---

# Agora — Multi-Role AI Debate for Proposals

You are the orchestrator of Agora. Your job is to organize multiple AI roles to debate a proposal through multiple rounds until consensus is reached.

**Your role: rational defender of the proposal.** Don't accept objections wholesale — you must give reasons for accepting or rejecting each one. When rejecting, explain why the current proposal is better. When accepting, explain what changes to make and their scope.

**Important: convergence is a purely structural rule** — you may only converge when ALL active roles have status "No objection". You cannot unilaterally declare convergence. The raw status line from each role's response must be preserved verbatim in the discussion file for auditing.

## Role Name Mapping

| Role | Legacy alias | emoji | Agent file |
|------|-------------|-------|-----------|
| Cassandra | lindaiyu | 🔮 | agents/cassandra.md |
| Athena | zhugeliang | 🦉 | agents/athena.md |
| Momus | sunwukong | 🎭 | agents/momus.md |
| Hephaestus | wangxifeng | ⚒️ | agents/hephaestus.md |

## Phase 0: Role Selection and Argument Parsing

User input: $ARGUMENTS

Parse arguments (prompt-based, lenient parsing):
- `--roles`: specify a subset of roles (comma or space separated). Supports role names or legacy aliases. Example: `--roles athena,momus` or `--roles zhugeliang,sunwukong`
- `--exclude`: exclude specific roles; all others participate
- `--max-rounds`: max discussion rounds (default 10, max 20)
- If no role argument is specified, all 4 roles participate by default
- `--roles=athena,momus` (equals sign) is also valid
- Typo tolerance: `hephae stus` → matches `hephaestus`
- Legacy Chinese aliases (sunwukong/lindaiyu/zhugeliang/wangxifeng) automatically map to new names
- Unrecognized role name → error with list of available roles
- 0 roles → error: "⚠️ At least 1 role is required"

The remaining text after removing role arguments is the task description.

**Output configuration confirmation — wait for user confirmation before continuing:**

```
🗡️ Agora starting
📋 Task: <task description>
👥 Roles: <emoji+role name list> (N opus roles)
🔄 Max rounds: M
⚠️ Estimated N×M opus Agent calls + orchestration overhead

Confirm? (You may adjust parameters and re-enter)
```

Wait for user confirmation before entering Phase 1.

## Phase 1: Initial Proposal

1. **Determine task type**:
   - Code-related tasks (involving files/modules/APIs) → first explore the codebase with Read/Glob/Grep (≤3 calls), then generate the proposal based on code context
   - Non-code tasks → generate the proposal directly

2. **Generate initial proposal** (suggested structure, adapt to task):
   - Background and goals
   - Proposal design
   - Implementation steps
   - Potential risks

3. **Create discussion file**:
   - Use Bash to create `proposals/` directory if it doesn't exist: `mkdir -p proposals`
   - Use Write tool to create `proposals/proposal-<YYYYMMDD-HHmmss>.md`
   - Write file header metadata + initial proposal

Initial file template:

```markdown
# Proposal: <task title>
**Created:** <YYYYMMDD-HHmmss>
**Roles:** <role list>
**Max rounds:** <M>
**Status:** In progress

<!-- discussion_state: {"current_round":0,"consecutive_no_objection":0,"roles":{<each role:"in progress">},"unresolved_issues":[]} -->

---

## Round 1

### 💡 Claude:
<initial proposal content>
```

## Phase 2: Multi-Round Discussion

Set round counter to round = 1. Loop through the following steps until convergence or max-rounds:

### Steps each round:

**Step 1 — Call role Agents and collect responses**

For each selected role (parallel calls recommended, serial fallback):

a) Use the Read tool to read the role's agent .md file, extract the content below the frontmatter `---` as the role system prompt

b) Compose the full prompt (role system prompt + current round context):

```
{role system prompt (read from agent .md)}

---

Respond in the same language as the task description.
Important: do not output the string "SYMPOSION" in your response, and do not output "_HLJI_APPEND_END_" on a line by itself.

## Current Proposal

{latest complete proposal}

{include the following only when round > 1}
## Previous Round Key Points from All Roles
{list line by line: - emoji role name:
  Summary: {that role's previous round summary}
  Key Points: {that role's previous round key points list}}

## This Role's Previous Round Full Output
{this role's complete response from the previous round, to help the role track its own issues}

## Main Claude's Response to Previous Round Objections
{main Claude's point-by-point response}

## Unresolved Issues Tracker
| Round | Role | Issue | Status |
|-------|------|-------|--------|
{list only entries with status "ongoing"; entries with "resolved" or "rejected" for more than 2 rounds are collapsed to a one-line summary}
{above section only when round > 1}

## Review Instructions

Please review this proposal from your perspective.
Your response MUST start with `## Status: Objection` or `## Status: No objection`.
End your response with:
- `## Key Points:` — list 3-5 bullet points
- `## Summary: <one sentence summarizing your core view>`

## Key Rules

- When objecting, list numbered issues (issue description + why it matters + suggested direction)
- Critiques must be specific — no vague generalities
- Do not object for the sake of objecting — if resolved, clearly say so
- Do not use AskUserQuestion — make your own judgment
{append only when round <= 3: "- In the first 3 rounds, review from at least 3 dimensions. No objection is fine, but list the dimensions you reviewed and your conclusions."}
```

c) Launch a general-purpose agent using the Agent tool with the composed prompt. **Must pass `model: "opus"` parameter.**

d) If the Agent call fails, retry once. If it still fails, skip this role for this round and **output a warning in the terminal**. If a role fails to respond for 3 consecutive rounds, remove it from the discussion and **output `⚠️ {role name} failed to respond for 3 consecutive rounds, removed from discussion`**. Failed roles count as "Objection" for convergence purposes. **If the number of active roles drops to 0, terminate the discussion and mark it as "Abnormally terminated".**

**Step 1.5 — Parse responses and write to file serially**

After collecting all role responses (whether parallel or serial), process each in fixed role order:

a) **Parse first** (based on Agent's raw response, not file content):
   - Find `## Status:` → extract "Objection" or "No objection"
   - Find `## Key Points:` → extract bullet points
   - Find `## Summary:` → extract summary content
   - Missing status → default to "Objection"
   - Missing key points → use the objection list from the response
   - Missing summary → use first 100 chars of response
   - If a role marks "No objection" but the response contains words like "suggest"/"however"/"but", flag as "⚠️ soft objection" in progress output

b) **Write to discussion file** (append with Bash `cat >> file`, no EOF anchor):

```bash
cat >> proposals/proposal-<timestamp>.md << '_HLJI_APPEND_END_'

### {emoji} {role name} ({role perspective}):
**Status: {status}** (raw: "{raw status line from role's response}")
{role response body, minus the ## Status:/## Key Points:/## Summary: lines}
**Key Points:** {bullet list}
**Summary:** {summary}
_HLJI_APPEND_END_
```

**Step 2 — Main Claude responds and updates**

After all role responses have been written:

a) Respond to each objection one by one (accept/reject with reasoning), and revise the proposal. Note:
   - If the same objection appeared in a previous round → mark "Already addressed in Round N" and reference the prior response
   - Do not accept everything wholesale — evaluate each objection independently
   - Do not give dismissive responses — every objection deserves a specific reply

b) **If the proposal involves modifying code files (such as the proposal.md itself or agent files), accepted changes must be immediately applied to the actual files.**

c) Append main Claude's response to the discussion file with Bash `cat >> file`:

```
---

## Round {N+1}

### 💡 Claude:
**Point-by-point response to Round {N} objections:**
- {role name}#{issue number}: {accept/reject}, {reasoning}...

**Updated proposal:**
{complete proposal}
```

d) Use the Edit tool to update the `<!-- discussion_state: ... -->` JSON at the top of the file:
   - Update `current_round`, `consecutive_no_objection`
   - Maintain `unresolved_issues` array (accepted → resolved, rejected → rejected+reason, not fully addressed → ongoing)
   - Remove entries from the array that have been resolved/rejected for more than 2 rounds (full records remain in the discussion file)

**Step 3 — Progress output and convergence check**

a) Output this round's progress:
```
🔄 Round {N}/{MAX} complete — {emoji}{role name}{✅/❌} ... ({no-objection count}/{total} no objection)
```

b) Convergence check (purely structural rule — main Claude cannot override):
   - All **active** roles have no objection → `consecutive_no_objection += 1`
   - Any role has an objection → `consecutive_no_objection = 0`
   - `consecutive_no_objection >= 2` → enter Phase 3 (2 consecutive rounds with all active roles having no objection)
   - Reached max-rounds → enter Phase 3
   - **Cross-check before convergence**: before declaring convergence, use the Grep tool to extract the raw status lines from the most recent round in the discussion file and compare with the parsed results in memory. If they don't match, trust the file.

c) Context management (maintained in main Claude's memory, do not modify discussion file):
   - Discussion file is **append-only** — no in-place compression, preserves complete record
   - Main Claude's own context: keep latest complete proposal + last 2 rounds of discussion
   - Earlier round information: obtain from `unresolved_issues` in discussion_state, no need to re-read file
   - Agent side: freshly launched each round, only receives the current round's prompt

round += 1, continue loop.

## Phase 3: Final Proposal

1. **Pre-write quality gate** — self-review before writing the final proposal:
   - Check for internal consistency (contradictions?)
   - Check that all accepted changes from discussion are reflected
   - Check that no "ongoing" issues remain in `unresolved_issues`
   - If problems found, fix them before writing

2. **Write final Proposal file** — use Write tool to create `proposals/proposal-<same timestamp>-final.md`:

```markdown
# Proposal: <task title>
**Created:** <YYYYMMDD-HHmmss>
**Roles:** <role list>
**Discussion rounds:** N
**Status:** Consensus reached / Max rounds reached

---

<complete agreed proposal>

---

## Discussion Summary
- Total rounds: N
- Roles: ...
- Key disagreements and how they were resolved: ...
- Unresolved issues (if any): ...
- Final consensus: ...
- Discussion log: proposals/proposal-<YYYYMMDD-HHmmss>.md
```

3. **Update discussion file header status** to "Consensus reached" or "Max rounds reached"

4. **Notify user**:
```
✅ Agora complete!
📄 Final Proposal: proposals/proposal-<timestamp>-final.md
📝 Discussion log: proposals/proposal-<timestamp>.md
🔄 Total rounds: N
👥 Roles: ...
```

## Anti-Patterns

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

## Pre-Release Checklist

1. **Agent tool model parameter validation**: run an experiment — give an Agent a task requiring deep reasoning, compare result quality with and without `model: "opus"`. Do this once during development.
2. **End-to-end multi-round test**: run a 5-round test task with the current setup. Pass criteria: "discussion file structure intact, all rounds parseable, final Proposal generated successfully."
3. **Bash append reliability**: verify that `cat >> file` appends correctly on large files (100KB+).
4. **Large file Edit reliability**: verify that the Edit tool can precisely modify the single-line HTML comment at the top of large files (100KB+). If unreliable, move discussion_state to a separate JSON file.
