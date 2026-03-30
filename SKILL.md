---
name: swarm
description: Orchestrate external AI coding agents (Droid, Codex, Claude, Amp review, etc.) as background workers from Claude Code. Use this skill whenever the user wants to dispatch tasks to other AI agents, run multi-agent reviews, get consensus from multiple models, or delegate work to external coding CLIs. Triggers on phrases like "/swarm", "let droid do X", "have codex review", "use amp to review", "send to gpt-5.4", "multi-agent review", "dispatch to", "use X agent to", or any mention of running tasks on external AI coding agents in parallel.
---

# Swarm

Orchestrate external AI coding agents as background workers. You are the dispatcher — you plan, spawn, verify, and report.

**Announce at start:** "Using swarm to dispatch this task to [agent(s)]."

## Why This Exists

A single agent gives one perspective. Different models catch different issues. Swarm lets you fan out work to external agents (Droid, Codex, Claude CLI, Amp review, or any compatible CLI), then verify or cross-compare the results — without leaving your Claude Code session.

## Core Concepts

- **Dispatcher**: You (Claude Code). You stay lean — plan the work, spawn workers, verify results.
- **Worker**: An external agent CLI running as a background process. It does the heavy lifting.
- **Single-agent mode**: One worker. You selectively verify its key claims afterward.
- **Multi-agent mode**: Multiple workers in parallel on the same task. Cross-compare their outputs — agreement = confidence, divergence = flag.

## Execution Flow

```
Single agent:   Parse → Config → Dispatch (background) → Verify → Report
Multi agent:    Parse → Config → Dispatch ×N (parallel background) → Cross-Compare → Report
```

## Step 0: Configuration

### First Run

If `~/.swarm/config.yaml` does not exist, follow the instructions in `references/first-run-setup.md` to auto-detect available CLIs and generate the config. Do this once, then proceed.

### Config Structure

Read `~/.swarm/config.yaml`. It has three sections:

```yaml
default: droid              # default backend
default_model: gpt-5.4      # default model

backends:
  droid:
    command: droid exec     # base command
    models: [gpt-5.4, gemini-3.1-pro-preview, claude-opus-4-6]
    flags:
      model: "-m {model}"
      worktree: "-w"
      output_json: "-o stream-json"
      session: "-s {session_id}"
  codex:
    command: codex exec
    models: [gpt-5.4]
    flags:
      model: "-m {model}"
      session: "-s {session_id}"
  claude:
    command: >
      env -u CLAUDE_CODE_ENTRYPOINT -u CLAUDECODE
      claude -p --dangerously-skip-permissions
    models: [opus, sonnet, haiku]
    flags:
      worktree: "-w"

  amp:
    command: amp review
    models: [default]
    task_types: [review]
    flags:
      files: "--files {files}"
      instructions: "--instructions {instructions}"
      checks_only: "--checks-only"
      summary_only: "--summary-only"

aliases:
  quick: { backend: droid, model: gemini-3-flash-preview }
  deep:  { backend: droid, model: gpt-5.4 }
```

Users can add any CLI agent by adding a new backend entry.

`amp` is a review-only backend. Do not dispatch implementation, refactor, or file-modifying tasks to it.

## Step 1: Parse the User's Request

Extract from the user's prompt:

1. **Task description** — what the worker should do
2. **Backend + model** — which agent and model to use. Scan for:
   - Explicit backend: "用 droid", "let codex", "have claude", "use amp to review"
   - Explicit model: "gpt-5.4", "opus", "gemini-3.1-pro"
   - Alias: "quick", "deep"
   - If nothing specified, use `default` backend + `default_model` from config (so `/swarm review X` just works)
3. **Multi-agent?** — look for "和...同时", "and...together", multiple agent names, "consensus", "compare"
4. **Worktree?** — look for "worktree", "隔离", "isolated"
5. **Target files/scope** — what code the worker should focus on

## Step 2: Build the Worker Command

For each worker, construct the full command:

### Command Template

For standard prompt-driven backends:

```
{backend.command} {backend.flags.model} "{prompt}"
```

For Amp review:

```
amp review [diff_description] [--files <paths...>] [--instructions <text>] [--checks-only] [--summary-only]
```

If the selected backend has `task_types: [review]`, only dispatch review/audit/analysis tasks to it. If the user explicitly chooses `amp` for non-review work, stop and tell them the Amp backend only supports code review.

### Prompt Construction

Write a focused prompt for the worker. Include:
- The task in clear, specific terms
- Target files or directories
- What output format you want (findings list, code changes, etc.)
- Constraint: "Write your complete output — findings, reasoning, and any code changes."
- **For review/analysis tasks:** ALWAYS append "Do NOT modify any files — report findings only." Even with `--auto medium`, workers will modify files unless explicitly told not to.

Do NOT include:
- Tool usage instructions (the worker knows its own tools)
- Implementation details about how to explore the codebase

### Amp Review Mapping

When the backend is `amp`, do not build a free-form worker prompt. Map the user's review request onto `amp review` arguments instead:

- **Diff description**: Use commit ranges, git-ish refs, or natural-language review scopes as the positional argument. Examples: `HEAD~1`, `main...HEAD`, `uncommitted changes in backend/`.
- **Files**: If the user names specific files or directories to focus on, pass them via `--files`.
- **Instructions**: If the user specifies extra focus like security, performance, naming, or error handling, pass that via `--instructions`.
- **No diff description**: Omit the positional argument so `amp review` uses uncommitted changes.
- **Read-only behavior**: Do not append "Do NOT modify any files". `amp review` is already a review command.

### Autonomy Level (Droid only)

Infer the `--auto` level from the task type — the user shouldn't have to think about this:
- **Review / audit / analysis** → `--auto medium` (Droid's default read-only mode only allows cat/less/head; grep/find/rg need medium)
- **Implementation / refactor / fix** → `--auto medium` (file creation/modification + search tools)
- **Install deps / run tests / build** → `--auto medium` (package installs, git local ops)
- **Git push / deploy** → `--auto high` (only if user explicitly asks)

Append the flag via `{backend.flags.auto}` with the inferred level.

### Worktree Handling

If the user requested worktree isolation:
- **Droid/Claude**: Append the worktree flag from config (e.g., `-w`)
- **Codex** (no native worktree): Create a worktree yourself first, then point Codex at it:
  ```bash
  git worktree add .worktrees/swarm-<task-id> -b swarm/<task-id>
  codex exec -C .worktrees/swarm-<task-id> -m <model> "<prompt>"
  ```
- **Amp**: It has no native worktree flag. Create/select the worktree yourself, then run `amp review` with Bash `workdir` set to that worktree path.
- **Other backends without worktree support**: Same as Codex — create worktree, pass working directory

## Step 3: Dispatch

### Create Task Directory

```bash
mkdir -p ~/.swarm/tasks/<task-id>/output
```

Use a short descriptive ID like `review-api-auth` or `refactor-errors`.

### Spawn Workers

Use Bash with `run_in_background: true`. Do NOT add shell redirects (`> file 2>&1`) — the `run_in_background` mechanism automatically captures stdout/stderr to its output file. Adding a redirect steals the output, leaving the background task's output file empty.

```bash
<full-command>
```

For multi-agent mode, spawn ALL workers in a single message (parallel tool calls).

### Report to User

After spawning, immediately tell the user:
- Task ID
- Which agent(s) and model(s) are running
- Brief description of what was dispatched

Then **return control** — don't block waiting. The user can keep working.

## Step 4: Collect Results

When a background task completes (you receive a notification), read the output from the background task's output file path (provided in the notification).

### Session Continuation (if worker needs more info)

If the worker's output contains questions, unresolved blockers, or incomplete work:

1. Surface the question to the user
2. Get their answer
3. Continue the worker's session:
   ```bash
   droid exec -s <session-id> "<answer>" > ~/.swarm/tasks/<task-id>/output/<backend>-continued.md 2>&1
   ```
   (Use the session ID from the worker's output — Droid and Codex both print it)

Amp review does not have session continuation. If its output is incomplete, refine the diff description or instructions and dispatch a fresh `amp review` run.

If NO session continuation is needed, proceed to verification.

## Step 5: Verify / Cross-Compare

### Single-Agent Mode: Selective Verify

Scan the worker's output for **concrete, verifiable claims** — things like:
- "File X has vulnerability Y at line Z"
- "Function F is missing error handling"
- "This endpoint has no authentication"

Pick 2-3 of the most critical claims and verify them yourself using Read/Grep:

```
Read the file → Check if the claim is accurate → Mark result
```

Tag each verified claim:
- **verified** — confirmed by reading the code
- **unverified** — couldn't confirm (file/function not found, or claim is ambiguous)
- **incorrect** — code contradicts the claim

Include these tags in your report to the user.

### Multi-Agent Mode: Cross-Compare + Spot-Check Unique Findings

Read all workers' outputs. Compare them:

**Agreement** — findings mentioned by 2+ agents. These are high confidence. No verification needed.

**Unique findings** — only one agent mentioned it. These are the most uncertain — could be an insight the other agents missed, or a hallucination. Pick 1-2 of the most impactful unique findings and **spot-check them yourself** using Read/Grep, the same way you'd verify in single-agent mode. Tag each as verified/unverified/incorrect.

**Contradictions** — agents disagree. Read the relevant code yourself to settle it.

Present a unified summary, not N separate outputs. Lead with agreements, then unique findings (with verification tags), then contradictions.

## Step 6: Report

Present the final report to the user:

### Single-Agent Report Format

```
## Swarm Report: <task-description>
**Agent:** <backend> (<model>) | **Duration:** <time>

### Findings
<worker's key findings, organized by severity/topic>

### Verification
- verified: <claim> — confirmed at <file:line>
- incorrect: <claim> — actually <what the code shows>
- unverified: <claim>
```

### Multi-Agent Report Format

```
## Swarm Report: <task-description>
**Agents:** <list> | **Duration:** <time>

### Consensus (N/N agents agree)
<findings all agents identified>

### Unique Findings
- [droid/gpt-5.4]: <finding only this agent caught>
- [codex/gpt-5.4]: <finding only this agent caught>

### Contradictions
- <topic>: Agent A says X, Agent B says Y
```

## Debate Mode

Adversarial design review between you (Claude Code) and an external agent. Use when the user wants to stress-test a design decision, not just get a second opinion.

**Trigger words:** "debate with", "argue with", "辩论", "反驳", "让 X 反驳", "和 X 讨论方案"

**Announce at start:** "Using swarm debate mode — I'll take position A, [agent] will challenge it."

### Why Debate, Not Just Review

A review asks "is this good?" A debate asks "where does this break?" The external agent is adversarial — its job is to find holes in your reasoning, not validate it. You then rebut, and the debate continues until convergence or the user intervenes.

### Execution Flow

```
Debate:  Take position → Write transcript → Dispatch opponent → Read response →
         Rebut (append to transcript) → Dispatch again → ... → Converge → Report
```

### Step 1: Setup

Create a shared transcript file. This is the **single source of truth** — both sides read the full history.

```bash
mkdir -p ~/.swarm/tasks/<debate-id>/output
```

Write the initial transcript to `~/.swarm/tasks/<debate-id>/debate.md`:

```markdown
# <Topic> — Design Debate

Context: <brief problem description>

---

## Round 1: <Your Name>'s Position

<your argument>

---

## Your Task

You are <opponent>. Read the FULL debate above. Attack the position — find holes, not just disagree.
<specific questions to answer>

Do NOT modify any files — analysis only.
```

### Step 2: Dispatch Rounds

Each round:

1. **Append your rebuttal** to the transcript (new `## Round N` section)
2. **Update the "Your Task" section** at the bottom with new questions
3. **Dispatch the full file** as the prompt:
   ```bash
   <backend-command> "$(cat ~/.swarm/tasks/<debate-id>/debate.md)" \
     > ~/.swarm/tasks/<debate-id>/output/round-N.md 2>&1
   ```
4. **Read the response**, extract the opponent's arguments
5. **Append the opponent's response** as a new round in the transcript
6. **Evaluate**: converged? New holes found? User should weigh in?

### Step 3: Convergence

Stop when:
- **Agreement reached** — both sides converge on the same recommendation
- **Irreducible disagreement** — positions are clear, tradeoffs understood, user must decide
- **Max rounds** (5) — surface to user with summary of positions
- **User intervenes** — user picks a direction

### Step 4: Report

```
## Swarm Debate Report: <topic>
**Agents:** You (Claude) vs <opponent> (<model>) | **Rounds:** N

### Consensus
<points both sides agree on>

### Your Position (final)
<your refined argument after N rounds>

### Opponent's Position (final)
<their refined argument>

### Key Concessions
- You conceded: <what>
- Opponent conceded: <what>

### Unresolved
<remaining disagreements — user must decide>

### Recommendation
<your final recommendation incorporating debate findings>
```

### Critical Rules

- **Full transcript every round.** The opponent has no memory between dispatches. The transcript file IS the memory. Every dispatch sends the complete file.
- **Append, don't overwrite.** Each round adds to the transcript. Never delete prior rounds.
- **Steel-man the opponent.** When rebutting, acknowledge valid points before countering. Don't strawman.
- **Verify claims.** If either side cites specific code (file:line), verify it with Read/Grep before accepting. Code-grounded claims beat theoretical arguments.
- **Concede when wrong.** If the opponent finds a real hole, say so. Update your position. The goal is a better design, not winning.

## Reference Files

- `references/config-example.yaml` — Default config template with all built-in backends
- `references/first-run-setup.md` — Instructions for first-run CLI auto-detection

## Routing

When `/swarm` is invoked:

- **No arguments**: Explain what swarm does and show usage examples
- **Config request** ("configure", "setup", "add backend"): Follow `references/first-run-setup.md`
- **Debate request** ("debate", "argue", "辩论", "反驳"): Execute debate mode flow
- **Task request**: Execute the single/multi-agent dispatch flow above
