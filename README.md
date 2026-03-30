# agent-swarm

Orchestrate external AI coding agents (Droid, Codex, Claude, Amp review, etc.) as background workers from Claude Code — with selective verification and multi-agent consensus.

> One agent is one perspective. Swarm gives you a team.

## Install

```bash
npx skills add me-frankan/agent-swarm
```

Or globally:

```bash
npx skills add me-frankan/agent-swarm -g
```

## What It Does

Swarm turns Claude Code into a dispatcher that fans out work to external AI coding agents, then **verifies** the results before reporting back.

- **Single-agent mode** — dispatch to one agent, selectively fact-check its key claims
- **Multi-agent mode** — dispatch the same task to multiple agents in parallel, cross-compare their outputs (consensus vs unique findings vs contradictions)
- **Debate mode** — adversarial design review where you and an external agent argue back and forth until convergence

```
Single agent:   Dispatch → Execute → Verify → Report
Multi agent:    Dispatch ×N → Execute (parallel) → Cross-Compare → Report
Debate:         Position → Challenge → Rebut → ... → Converge → Report
```

## Usage

```bash
# Default (droid + gpt-5.4)
/swarm review backend/api.py for security issues

# Route review through Amp's native review CLI
/swarm use amp to review main...HEAD
/swarm use amp to review backend/api.py for security issues

# Specify agent and model
/swarm use droid(gemini-3.1-pro) to review the auth module

# Multi-agent consensus
/swarm use droid and codex to review the CORS config

# Debate mode — adversarial design review
/swarm debate with codex about whether to use Redis or DB for heartbeat
/swarm 和 droid 辩论这个方案的可行性

# With worktree isolation
/swarm use sonnet to refactor error handling in a worktree

# Quick aliases
/swarm quick review api.py       # gemini-3-flash (fast + cheap)
/swarm deep review api.py        # gpt-5.4 (slow + thorough)
```

## Supported Agents

Built-in backends:

| Agent | CLI | Models |
|-------|-----|--------|
| **Droid** | `droid exec` | gpt-5.4, gemini-3.1-pro, claude-sonnet-4-6, glm-5, kimi-k2.5, ... |
| **Codex** | `codex exec` | gpt-5.3-codex, gpt-5.2 |
| **Claude** | `claude -p` | opus, sonnet, haiku |
| **Amp** | `amp review` | default (`amp` chooses the review agent) |

Add any CLI agent by editing `~/.swarm/config.yaml`.

Amp is intentionally treated as a review-only backend. Use it for code review tasks, not implementation/refactor work.

## How It Works

### Verification (Single-Agent Mode)

After the worker returns, Swarm scans for concrete claims ("file X has vulnerability Y at line Z") and spot-checks 2-3 of them by reading the actual code. Each claim gets tagged:

- **verified** — confirmed by reading the source
- **unverified** — couldn't confirm
- **incorrect** — code contradicts the claim

### Consensus (Multi-Agent Mode)

When multiple agents review the same code:

- **Agreement** — findings from 2+ agents → high confidence, no extra verification
- **Unique findings** — only one agent found it → spot-checked by the dispatcher
- **Contradictions** — agents disagree → dispatcher reads the code to settle it

### Autonomy Level

Automatically inferred from task type (Droid only):
- Review/audit → read-only (default)
- Implementation/refactor → `--auto low`
- Build/test → `--auto medium`
- Deploy/push → `--auto high` (only if explicitly requested)

### Debate Mode

Stress-test a design decision through adversarial back-and-forth. You take a position, the external agent attacks it. Each round appends to a shared transcript file so the opponent always has full context (external agents have no memory between calls).

Key rules:
- **Full transcript every round** — the transcript file IS the opponent's memory
- **Verify code claims** — if either side cites file:line, check it before accepting
- **Concede when wrong** — the goal is a better design, not winning
- Stops on convergence, irreducible disagreement, or 5 rounds max

### Session Continuation

If a worker gets stuck or has questions, Swarm continues its session (`droid exec -s <session-id>`) rather than starting over.

## Configuration

First run auto-detects available CLIs. Config lives at `~/.swarm/config.yaml`:

```yaml
default: droid
default_model: gpt-5.4

backends:
  droid:
    command: droid exec
    models: [gpt-5.4, gemini-3.1-pro-preview, claude-sonnet-4-6]
    flags:
      model: "-m {model}"
      worktree: "-w"
      session: "-s {session_id}"
      auto: "--auto {level}"

  codex:
    command: codex exec
    models: [gpt-5.3-codex]
    flags:
      model: "-m {model}"
      session: "-s {session_id}"

  claude:
    command: env -u CLAUDE_CODE_ENTRYPOINT claude -p --dangerously-skip-permissions
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
  security-reviewer:
    backend: droid
    model: gpt-5.4
    prompt: "You are a security-focused reviewer. Prioritize OWASP Top 10."
```

Re-detect CLIs anytime:

```bash
/swarm setup
```

## Amp Review Backend

When the selected backend is `amp`, Swarm dispatches through `amp review` instead of the normal prompt-style worker flow.

- Commit ranges stay positional: `/swarm use amp to review main...HEAD`
- File focus maps to `--files`: `/swarm use amp to review backend/api.py for security issues`
- Extra focus maps to `--instructions`: security, performance, error handling, etc.
- If no diff description is given, `amp review` defaults to uncommitted changes

## Requirements

- **Claude Code** as the dispatcher (uses `run_in_background`, `Read`, `Grep`)
- At least one external agent CLI installed (`droid`, `codex`, `claude`, or `amp` for review-only use)

## License

MIT
