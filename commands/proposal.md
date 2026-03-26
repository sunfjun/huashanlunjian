---
description: Launch Agora — multi-role AI debate to generate high-quality proposals
argument-hint: "<task description> [--roles athena,momus] [--exclude hephaestus] [--max-rounds 10] [--interactive]"
allowed-tools: [Read, Write, Edit, Glob, Grep, Agent, Bash, AskUserQuestion]
---

# Agora — Multi-Role AI Debate for Proposals

You are the orchestrator of Agora. Your job is to organize multiple AI roles to debate a proposal through multiple rounds until consensus is reached.

**Your role: rational steward of the proposal.** You defend what works, accept what improves, and reject what weakens — with explicit reasoning for every decision.

**Important: convergence is a purely structural rule** — you may only converge when ALL active roles have status "No objection". You cannot unilaterally declare convergence. The raw status line from each role's response must be preserved verbatim in the discussion file for auditing.

## Role Name Mapping

| Role | emoji | Agent file |
|------|-------|-----------|
| Cassandra | 🔮 | agents/cassandra.md |
| Athena | 🦉 | agents/athena.md |
| Momus | 🎭 | agents/momus.md |
| Hephaestus | ⚒️ | agents/hephaestus.md |

## Setup: Role Selection and Argument Parsing

User input: $ARGUMENTS

Parse arguments (prompt-based, lenient parsing):
- `--roles`: specify a subset of roles (comma or space separated). Example: `--roles athena,momus`
- `--exclude`: exclude specific roles; all others participate. Mutually exclusive with `--roles` — if both are specified, error: "⚠️ --roles and --exclude cannot be used together"
- `--max-rounds`: max discussion rounds (default 10, max 20)
- `--interactive` (or `-i`): enable interactive mode — pause after each round of agent feedback to let the user provide input before the orchestrator responds
- `--resume <file>`: resume a previously interrupted discussion (see Resume section below)
- `--roles=athena,momus` (equals sign) is also valid
- Typo tolerance: `hephae stus` → matches `hephaestus`
- Unrecognized role name → error with list of available roles
- 0 roles → error: "⚠️ At least 1 role is required"

The remaining text after removing role arguments is the task description.

**If `--resume` is specified, skip the rest of Setup and Initial Proposal, and go directly to Resume.**

**Auto role selection (when no `--roles` or `--exclude` is specified):**

If the user did not explicitly specify roles, assess task complexity and auto-select:

| Complexity | Criteria | Roles | Max rounds |
|-----------|----------|-------|------------|
| Light (default) | Most tasks: bug fix, feature addition, refactor, config change, analysis | 🎭Momus + ⚒️Hephaestus (2) | 5 |
| Medium | Multi-module refactor, API design, moderate architecture decision | 🔮Cassandra + 🦉Athena + 🎭Momus (3) | 8 |
| Heavy | System architecture, security-critical change, cross-team impact, large-scale migration | All 4 roles | 10 |

Selection heuristics (assess from the task description):
- **Default to Light** unless the task clearly matches Medium or Heavy criteria
- Involves security, auth, or data privacy → must include 🎭Momus
- Involves architecture, scalability, or long-term design → must include 🦉Athena
- Involves implementation details, edge cases, or data handling → must include 🔮Cassandra
- Involves feasibility, timeline, or resource constraints → must include ⚒️Hephaestus
- When in doubt, prefer Light over Medium

Show auto-selection reasoning in the confirmation prompt so the user can override.

**Output configuration confirmation — wait for user confirmation before continuing:**

```
🗡️ Agora starting
📋 Task: <task description>
👥 Roles: <emoji+role name list> (N opus roles) {if auto-selected: "[auto: <complexity level>]"}
🔄 Max rounds: M
{if interactive: "🤝 Interactive mode: ON — you can provide input after each round"}
⚠️ Estimated N×M opus Agent calls + orchestration overhead
{if auto-selected Medium: "💡 如需可行性评估，可添加 --roles 包含 hephaestus"}

Confirm? (You may adjust parameters and re-enter)
```

Wait for user confirmation before entering Initial Proposal.

## Resume (only when `--resume` is specified)

When `--resume <file>` is provided (file can be either the discussion `.md` or the `.state.json`):

1. **Locate both files**: derive the timestamp from the filename, then find `proposal-<timestamp>.md` and `proposal-<timestamp>.state.json` in the same directory.

2. **Read the state file** (`proposal-<timestamp>.state.json`).

3. **Check phase** — branch based on `state.phase`:
   - `"init"`: The discussion barely started. Discard and start fresh (re-run from Setup).
   - `"assessment"`: Interrupted during Independent Assessment. Grep the discussion file for `### {emoji} {角色名} 的开发计划：` markers to detect which roles completed. Compare with `assessment_completed_roles`; if inconsistent, trust the discussion file. Re-invoke only incomplete roles, then synthesize and proceed to Debate.
   - `"debate"`: Interrupted during Debate. Proceed to step 4 below.
   - `"final"`: Interrupted during Finalize. Re-read the latest proposal and re-run Finalize.

4. **Consistency check (debate phase only)** — compare `state.current_round` with the last `## Round N` number found in the discussion file (use Grep):
   - **Match**: state is consistent, proceed normally.
   - **Scenario A** (state `current_round` > last round in discussion file — state is newer): The interrupt happened after writing state but before appending discussion content. **Fix:** roll back `state.current_round` to match the discussion file's last round number.
   - **Scenario B** (last round in discussion file > state `current_round` — discussion is newer): The interrupt happened after appending discussion content but before writing state. **Fix:**
     1. **Completeness check**: count the number of `**Status:` lines in the last round of the discussion file and compare with the number of roles in `state.roles`. If they don't match (partial round write), roll back `current_round` to the previous complete round (last round number - 1).
     2. If complete: parse `current_round` from the discussion file, extract each role's status from `**Status: {status}**` lines. Set all other fields to conservative defaults: `consecutive_no_objection=0`, `sleeping=false`, `proposal_modified=true`.
   - Write the corrected state back to the state file.

5. **Restore context**: read the last 2 rounds from the discussion file (find `## Round N` markers). Extract the latest complete proposal from the most recent `**Updated proposal:**` section.

6. **Inject decision_log**: from the state file, prepare decision history for the orchestrator context:
   - Last 3 rounds: include full entries (round, role, issue, decision, summary)
   - Older entries: include round, role, issue, decision only (summary omitted), aggregated by role

7. **Resume Debate**: set `round = state.current_round + 1` and enter the Debate loop. **The first round after resume, all roles MUST be active (no hibernation allowed)**, regardless of their `consecutive_no_objection` or `sleeping` state — to ensure all roles have sufficient context before potentially entering sleep.

8. **Output resume status**:
```
🔄 Resuming discussion from Round {N}
📄 Discussion file: {file path}
👥 Roles: {role list from state}
📊 State: {current_round} rounds completed, proposal version {proposal_version}
```

## Initial Proposal

1. **Determine task type**:
   - Code-related tasks (involving files/modules/APIs) → deeply explore the codebase with Read/Glob/Grep (use as many calls as needed), understand existing patterns, conventions, dependencies, and the full context before proposing anything
   - Non-code tasks → generate the proposal directly

2. **Generate initial proposal** (suggested structure, adapt to task):
   - Background and goals
   - Code context (for code-related tasks): relevant files, current architecture, existing patterns
   - Proposal design — for code tasks, include specific files to modify, function-level changes, and code snippets
   - Implementation steps — ordered by dependency, with concrete file paths
   - Test plan (for code tasks): what tests to add or modify
   - Potential risks

3. **Create discussion file**:
   - Use the Write tool to create `proposals/proposal-<YYYYMMDD-HHmmss>.md` (the Write tool will create the directory if needed)
   - Write file header metadata + initial proposal

Initial files:

**Discussion file** (`proposals/proposal-<YYYYMMDD-HHmmss>.md`):

```markdown
# Proposal: <task title>
**Created:** <YYYYMMDD-HHmmss>
**Roles:** <role list>
**Max rounds:** <M>
**Status:** In progress

<!-- state_file: proposal-<YYYYMMDD-HHmmss>.state.json -->

---

## Round 1

### 💡 Claude:
<initial proposal content>
```

**State file** (`proposals/proposal-<YYYYMMDD-HHmmss>.state.json`) — use the Write tool to create:

```json
{
  "phase": "init",
  "interactive": false,
  "assessment_completed_roles": [],
  "current_round": 0,
  "consecutive_no_objection": 0,
  "proposal_version": 1,
  "proposal_modified": false,
  "roles": {
    "<role_name>": {
      "status": "in progress",
      "consecutive_no_objection": 0,
      "sleeping": false,
      "last_reviewed_version": 0,
      "sleep_since_round": null,
      "consecutive_failures": 0
    }
  },
  "metrics": {
    "agent_calls": 0,
    "sleeping_rounds_saved": 0
  },
  "unresolved_issues": [],
  "decision_log": []
}
```

## Independent Assessment (after Initial Proposal, before Debate)

After creating the discussion file and initial proposal, proceed to Independent Assessment. This phase lets each agent independently explore the codebase and produce a development plan from their unique perspective.

**1. Update state:** Set `"phase": "assessment"` in the state file.

**2. Parallel agent calls:** Launch all participating roles in parallel. Each agent receives the following prompt (NOT Review Rules — this is a planning phase, not a review phase):

```
{role system prompt (read from agent .md)}

---

Respond in the same language as the task description.

You are participating in the **Independent Assessment** phase of a development task. Your job is to **independently explore the codebase** and produce a development plan from your role's perspective.

You may freely use markdown headings to organize your output. This phase does NOT use Review Rules (no Status/Objection format required).

**Task description:** {user's task description}

**Orchestrator's initial code analysis:**
{key files, project structure, and relevant code identified in Initial Proposal}

**Requirements:**
1. Use Read/Glob/Grep to independently explore the codebase — do not rely solely on the orchestrator's analysis
2. Evaluate this task from your role's perspective ({role description})
3. Output your development plan in the following format:

## Code Assessment ({role name} perspective)

### 1. Current Code Analysis
- Which files you read (with file paths and line numbers)
- Your judgment of existing architecture/patterns
- Key code points relevant to the task

### 2. Development Plan
- Specific files and functions to modify
- Code-level change descriptions (with code snippets)
- New files/modules (if any)

### 3. Risks and Recommendations
- Risks identified from your role's perspective
- Suggested mitigations

### 4. Implementation Priority
- Suggested implementation order with reasoning
```

**3. Collect and write to discussion file:** After all agents respond, append their plans to the discussion file in fixed role order:

```
## Independent Assessment

### {emoji} {role name} 的开发计划：
{agent's full output}

### {emoji} {role name} 的开发计划：
{agent's full output}
...
```

**4. Synthesize v2 proposal:** The orchestrator merges all role plans into a unified v2 proposal using these rules:

- **Union of changes:** Collect all file changes proposed by all roles into a complete list
- **Conflict arbitration:**
  - Architecture conflicts → Athena's judgment takes priority
  - Feasibility/effort conflicts → Hephaestus's judgment takes priority
  - Security conflicts → Momus's judgment takes priority
  - Edge cases/testing → Cassandra's judgment takes priority
  - **Cross-dimension fallback:** When a conflict spans multiple dimensions, orchestrator uses judgment and documents the conflict point and reasoning in the discussion file
- **Output:** Generate a unified v2 proposal using the standard Initial Proposal format
- **Context management:** Original role plans are preserved in the discussion file; orchestrator context only retains the synthesized v2

**5. Write v2 as Round 1:** Append the synthesized v2 proposal to the discussion file as `## Round 1 / ### 💡 Claude:`.

**6. Update state:** Set `"phase": "debate"`, `"current_round": 1`, update `assessment_completed_roles` to include all completed roles. Write state file **after** writing the discussion file.

**Error handling:**
- Agent call fails → retry once
- Still fails → skip that role, synthesize with available plans
- All agents fail → skip Independent Assessment entirely, use the Initial Proposal as-is and enter Debate

**7. Proceed to Debate.**

## Debate: Multi-Round Discussion

Set round counter to round = 1. Loop through the following steps until convergence or max-rounds:

### Steps each round:

**Step 0.5 — Role hibernation check**

Before calling agents, determine which roles are active this round:

For each role, check hibernation status:
- A role **enters sleep** when: `role.consecutive_no_objection >= 2` AND `proposal_modified == false` (from previous round)
- A role **wakes up** when any of:
  - `proposal_modified == true` (proposal changed last round)
  - Forced wake-up: `current_round - role.sleep_since_round >= 3` (slept for 3 rounds)
  - Fast convergence check requires it (see Step 3)
- Round 1: all roles are always active (skip hibernation check)
- On any state ambiguity: default to **awake**
- When a sleeping role wakes up, it receives the full current proposal + accumulated Key Points from all missed rounds (grouped by round)

Output sleeping roles: `😴 {role name} sleeping (since Round N)` for each sleeping role this round.

**Step 1 — Call role Agents and collect responses**

For each **active** (non-sleeping) role (parallel calls recommended, serial fallback):

a) Use the Read tool to read the role's agent .md file, extract the content below the frontmatter `---` as the role system prompt

b) Compose the full prompt (role system prompt + review rules injected by orchestrator + dynamic context):

```
{role system prompt (read from agent .md)}

## Review Rules

- Every response MUST start with `## Status: Objection` or `## Status: No objection`
- When objecting, list numbered issues (issue description + why it matters + suggested direction)
- Critiques must be specific — no vague generalities
- Do not object for the sake of objecting — if resolved, clearly say so
- End every response with `## Key Points:` (3-5 bullet points) and `## Summary: <one sentence>`
- Do NOT use markdown headings (`#`/`##`/`###`) in the body — only `## Status:`, `## Key Points:`, and `## Summary:` are allowed as headings
- You are a subagent. You cannot use AskUserQuestion. You must complete the review independently and make your own judgments.

---

Respond in the same language as the task description.
Important: do not output the string "SYMPOSION" in your response, and do not output "_HLJI_APPEND_END_" on a line by itself.

**When the proposal involves code changes, you MUST use your tools (Read, Glob, Grep) to read the actual source code before reviewing.** Do not review based on proposal text alone — verify file paths, function signatures, and existing behavior against the real codebase. Your review should reference specific files and line numbers.

## Current Proposal

{latest complete proposal}

{include the following only when round > 1}
## Previous Round Key Points from All Roles
{list line by line: - emoji role name:
  Key Points: {that role's previous round key points list}}

## This Role's Previous Round Issues
{structured state format — replaces full output to save ~50-60% tokens:}
{for each objection the role raised last round:}
{N. [original objection text verbatim] → Status: accepted/rejected/ongoing}
{   Orchestrator: [1-2 sentence summary of response]}
{   (for ongoing only) Action: re-evaluate whether this issue persists in the current proposal version. Mark as resolved or still unresolved with updated reasoning.}
{Your previous Summary: [original summary text]}

## Main Claude's Response to Previous Round Objections
{main Claude's point-by-point response}

## Unresolved Issues Tracker
| Round | Role | Issue | Status |
|-------|------|-------|--------|
{list only entries with status "ongoing"; entries with "resolved" or "rejected" for more than 2 rounds are collapsed to a one-line summary}
{above section only when round > 1}

Please review this proposal from your perspective.
{append only when round <= 3: "In the first 3 rounds, review from at least 3 dimensions. No objection is fine, but list the dimensions you reviewed and your conclusions."}
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
   - If a role marks "No objection" but the response contains actionable change requests (e.g. "应该改为"/"must fix"/"should change to"/"需要修改"), flag as "⚠️ soft objection" in progress output

b) **Write to discussion file** using the Edit tool to append at the end of the file. Find the last line of the file and append after it. Content to append:

```
### {emoji} {role name} ({role perspective}):
**Status: {status}** (raw: "{raw status line from role's response}")
{role response body, minus the ## Status:/## Key Points:/## Summary: lines}
**Key Points:** {bullet list}
**Summary:** {summary}
```

**Step 1.8 — User input (interactive mode only)**

Skip this step if `--interactive` was not specified.

After all role responses have been written to the discussion file, pause to collect user input:

a) Output a brief summary of this round's agent feedback:
```
📋 Round {N} agent feedback:
{for each active role: - emoji role_name: status — summary}
```

b) Use AskUserQuestion to ask: "本轮 agent 反馈已完成，是否需要补充意见？" with options:
   - "继续" (description: "直接进入下一步，由 orchestrator 回应")
   - "我有补充" (description: "输入你的意见或约束条件，会作为额外上下文纳入讨论")

c) If user selects "继续" → proceed to Step 2 normally.

d) If user selects "我有补充" or provides custom text → treat user input as a **stakeholder directive**:
   - Append to discussion file:
     ```
     ### 👤 User:
     {user input text}
     ```
   - In Step 2, the orchestrator MUST address user input alongside agent objections. User directives take priority over agent suggestions when they conflict — the user is the ultimate decision maker.
   - Include user input in the next round's agent prompt under a new section:
     ```
     ## User Directive (Round {N})
     {user input text}
     ```

**Step 2 — Main Claude responds and updates**

After all role responses have been written (and user input collected if in interactive mode):

a) Respond to each objection one by one (accept/reject with reasoning), and revise the proposal. Note:
   - If the same objection appeared in a previous round → mark "Already addressed in Round N" and reference the prior response
   - Do not accept everything wholesale — evaluate each objection independently
   - Do not give dismissive responses — every objection deserves a specific reply

b) **If the proposal involves modifying code files (such as the proposal.md itself or agent files), accepted changes must be immediately applied to the actual files.**

c) Append main Claude's response to the discussion file using the Edit tool (append at end of file):

```
---

## Round {N+1}

### 💡 Claude:
**Point-by-point response to Round {N} objections:**
- {role name}#{issue number}: {accept/reject}, {reasoning}...

**Updated proposal:**
{complete proposal}
```

d) **Update the state file** (`proposal-<timestamp>.state.json`) using the Write tool (overwrite with updated JSON). Write order: **always write the discussion file first (step c), then the state file (step d)** — because state can be inferred from the discussion file if needed.

   Update the following fields:
   - `current_round`, `consecutive_no_objection`
   - `proposal_modified`: `true` if any objection was accepted AND the proposal text was changed this round, `false` otherwise
   - If `proposal_modified == true`: increment `proposal_version`
   - Each role's state:
     - `status`: "objection" / "no objection" / "sleeping"
     - `consecutive_no_objection`: increment if No objection, reset to 0 if Objection, unchanged if sleeping
     - `sleeping`: set based on Step 0.5 rules for next round
     - `last_reviewed_version`: set to current `proposal_version` if the role was active this round
     - `sleep_since_round`: set to current round when entering sleep, null when awake
   - `unresolved_issues` array (accepted → resolved, rejected → rejected+reason, not fully addressed → ongoing). Remove entries resolved/rejected for more than 2 rounds.
   - `decision_log` array: for each objection responded to this round, append `{"round": N, "role": "<role>", "issue": "<issue text>", "decision": "accept/reject", "summary": "<1-2 sentence reasoning>"}`. **Growth control:** keep last 3 rounds' entries in full; for older entries, keep `round`, `role`, `issue`, and `decision` but remove `summary`.
   - `metrics.agent_calls`: increment by the number of agent calls made this round. `metrics.sleeping_rounds_saved`: increment by the number of sleeping roles this round.

**Step 3 — Progress output and convergence check**

a) Output this round's progress:
```
🔄 Round {N}/{MAX} complete — {emoji}{role name}{✅/❌} ... ({no-objection count}/{total} no objection)
```

b) Convergence check (purely structural rule — main Claude cannot override):
   - All **active** roles have no objection → `consecutive_no_objection += 1`
   - Any role has an objection → `consecutive_no_objection = 0`
   - **Standard convergence**: `consecutive_no_objection >= 2` AND ALL roles (including sleeping ones) have `last_reviewed_version >= proposal_version` → enter Finalize. If version check fails, wake up roles that haven't reviewed the current version and run one more round.
   - **Fast convergence** (all conditions must be met):
     1. `proposal_modified == false` for the last 2+ consecutive rounds
     2. All active roles said "No objection" this round
     3. ALL roles (including sleeping ones) have `last_reviewed_version >= proposal_version` (every role has seen the current proposal)
     - If condition 3 fails: wake up roles that haven't reviewed the current version, run one more round with them, then re-check
   - Reached max-rounds → enter Finalize
   - **Cross-check before convergence**: before declaring convergence, use the Grep tool to extract the raw status lines from the most recent round in the discussion file and compare with the parsed results in memory. If they don't match, trust the file.

c) Context management (maintained in main Claude's memory, do not modify discussion file):
   - Discussion file is **append-only** — no in-place compression, preserves complete record
   - Main Claude's own context: keep latest complete proposal + last 2 rounds of discussion
   - Earlier round information: obtain from `unresolved_issues` and `decision_log` in the state file, no need to re-read the full discussion file
   - Agent side: freshly launched each round, only receives the current round's prompt

round += 1, continue loop.

## Finalize: Final Proposal

1. **Pre-write quality gate** — read the state file (`proposal-<timestamp>.state.json`) and self-review before writing the final proposal:
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
📊 Agent calls: {metrics.agent_calls} ({sleeping_rounds_saved} sleeping rounds saved)
```

<!-- Anti-Patterns and Pre-Release Checklist moved to docs/anti-patterns.md and docs/pre-release-checklist.md -->
