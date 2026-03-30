# First-Run Setup

Auto-detect available agent CLIs and generate `~/.swarm/config.yaml`.

## Detection Steps

Run these checks in parallel:

```bash
which droid 2>/dev/null && droid --version 2>&1
which codex 2>/dev/null && codex --version 2>&1
which claude 2>/dev/null && claude --version 2>&1
which amp 2>/dev/null && amp --version 2>&1
which gemini 2>/dev/null && gemini --version 2>&1
```

## Generate Config

1. Read `references/config-example.yaml` as the template
2. For each detected CLI, keep its backend entry. Remove entries for CLIs not found.
3. Set `default` to the first available backend in priority order: droid > codex > claude > amp
4. If `default` ends up as `amp`, note that the default backend is review-only and non-review tasks must explicitly use another backend.
5. Write the result to `~/.swarm/config.yaml`
6. Create `~/.swarm/tasks/` directory

## Auth Verification

For each detected backend, run a quick smoke test to verify authentication:

```bash
droid exec -m gpt-5.4 "respond with only: ok" 2>&1
codex exec -m gpt-5.3-codex "respond with only: ok" 2>&1
amp usage 2>&1
```

If auth fails, note it in the output:
- Codex: "Run `codex login` to authenticate"
- Droid: "Set FACTORY_API_KEY or run `droid login`"
- Claude: "Should work if Claude Code is running (you're already in it)"
- Amp: "Run `amp login` to authenticate"

## Report

Tell the user:
- Which backends were detected and authenticated
- Which were not found or failed auth
- Where the config was saved
- How to add more backends later (edit `~/.swarm/config.yaml`)

## Re-running Setup

This same flow runs when the user says `/swarm setup` or `/swarm configure`.
When re-running, read the existing config first, preserve any custom backends/aliases
the user has added, and only update the built-in backend entries.
