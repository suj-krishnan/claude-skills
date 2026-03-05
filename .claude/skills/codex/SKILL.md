---
name: codex
description: Use OpenAI's Codex in Claude. Use this skill whenever the user
  mentions codex, asks to "run this through codex", wants a "second opinion"
  or "different perspective" from another model, asks to compare approaches
  between models, or wants to delegate a task to codex while continuing
  other work.
---

# Codex Skill

Delegate tasks, get code reviews, and ask questions using OpenAI's Codex
CLI through `roachdev codex`.

## When to Use
- The user asks for a "second opinion" or "different perspective" on code,
  design, or implementation
- The user wants to validate an approach with a different model before
  committing to it
- The user wants an independent code review from a different model
- The user explicitly mentions codex or asks to "run something through codex"
- The user wants to delegate a task to codex (e.g., "have codex refactor
  this while I work on something else")
- The user wants to compare how different models approach a problem

**Parallel execution is mandatory.** When running a codex command, always
launch it in parallel with your other work (e.g., use `run_in_background`
or invoke it alongside other tool calls). 

## Prerequisites
Verify Codex CLI is available:
```bash
codex --version  # Should display installed version
```

## Commands

### When to use `--`

Use `--` to separate `roachdev codex` arguments from flags intended for
the underlying `codex` CLI. Flags like `-m` (model) and `-c` (config)
belong to the codex CLI and need `--` before them. Subcommand flags like
`--base`, `--uncommitted`, and `--commit` belong to the `review`
subcommand and also need `--`. When passing only a subcommand name and a
prompt string, `--` is not needed.

```bash
# No -- needed: subcommand + prompt only
roachdev codex review "Focus on error handling"
roachdev codex exec "Explain this function"

# -- needed: passing codex CLI flags (-m, -c) or subcommand flags (--base, --uncommitted)
roachdev codex -- review --base main
roachdev codex -- review --uncommitted
roachdev codex -- -m o3 review --uncommitted
```

### Code Review (Non-Interactive)

Review changes against a base branch (e.g. after checking out a PR):
```bash
roachdev codex -- review --base main
```

Review uncommitted changes:
```bash
roachdev codex -- review --uncommitted
```

Review a specific commit:
```bash
roachdev codex -- review --commit <sha>
```

Review with custom instructions:
```bash
roachdev codex review "Focus on error handling and concurrency safety"
```

**Important:** `--uncommitted`, `--base`, and `--commit` cannot be combined
with a `[PROMPT]` argument. To add custom instructions when using these
flags, use as below:
```bash
{ printf '%s\n\n' "Review this diff; Focus on error handling"; git diff HEAD;
} | roachdev codex review -
git diff --staged | roachdev codex review -
git diff main...HEAD | roachdev codex -- review -c 'model_reasoning_effort=xhigh' -
```
Match the diff command to the intended scope: `git diff HEAD` for
staged+unstaged, `git diff --staged` for staged only,
`git diff <base>...HEAD` for base-branch comparison (like `--base`).
Note: `git diff HEAD` does not include untracked files, unlike
`--uncommitted`. When untracked files matter, prefer `--uncommitted`
without custom instructions over the stdin workaround.

### Ask a Question (Non-Interactive)

Use `codex exec` to ask a one-shot question and get a response:
```bash
roachdev codex exec "Critically review the architecture of package pkg/kv/kvserver"
```

### Specify model and reasoning effort

The examples below use placeholder model names. Use the default model
unless the user explicitly requests a different one — available models
change over time.
```bash
roachdev codex -- -m <model-name> review -c 'model_reasoning_effort=xhigh' --uncommitted
roachdev codex -- -m <model-name> exec -c 'model_reasoning_effort=high' "Explain this function"
```

### Sandbox modes
```bash
roachdev codex -- exec --sandbox read-only "Explain the auth flow"
roachdev codex -- exec --full-auto "Refactor this function"
```

### Resume a session
```bash
echo "your prompt here" | roachdev codex -- exec resume --last -
```

## Workflow

1. **Choose the right command**:
   - For code review: use `review`
   - For everything else: use `exec`
   - Select the sandbox mode required for the task; default to
     `--sandbox read-only` unless edits or network access are necessary.
2. Use the default provider model unless the user has explicitly requested a
   different model.
3. **Run the command**
   - Always run this command in parallel with any existing task you might be 
     performing.
   - Wait for the command to finish and capture the output.
   - If the command fails, check the troubleshooting section below and
     ask how to adjust using AskUserQuestion.
4. **Present the output**:
   - Codex returns plain text (often markdown-formatted). Summarize the
     key findings rather than dumping the full output. Offer to show the
     raw output if the user wants it.
   - If the user asked for a second opinion or comparison, critically
     evaluate the codex output alongside your own analysis — see below.

### Critically evaluating codex output

Do this only when the user has requested a comparison or second opinion.
Otherwise, present the summary and move on.

- Present your own output, if any, before evaluating codex's.
- Compare the two outputs.
- Explicitly identify:
  - Points where you agree (and why)
  - Points where you disagree (and why)
  - Any factual errors or questionable reasoning
  - Any insights it identified that you did not initially consider
- Maintain analytical rigor and objectivity.
- Do not defer to the other model's conclusions without justification.
- Avoid vague statements; provide reasoning for all judgments.
- If uncertainty exists, explicitly acknowledge it.
- Produce a structured response that includes agreements and
  disagreements with supporting evidence and recommendations.

## Troubleshooting

Common failure modes and how to handle them:

| Symptom | Likely cause | Action |
|---|---|---|
| `command not found: codex` | Codex CLI not installed | Tell the user to install it |
| `Error: authentication failed` or `401` | Missing or expired API key | Ask the user to check their codex authentication |
| `unrecognized flag` or `unknown command` | Outdated codex binary | Suggest updating: `codex --version` to check current version |
| Command hangs or times out | Large diff or high reasoning effort | Suggest reducing scope or lowering reasoning effort |

## Guidelines

- Use a Bash timeout of at least 120000ms (2 minutes) for codex commands.
  For tasks involving large diffs or high reasoning effort, use up to
  600000ms (10 minutes). If a command times out, inform the user and
  suggest reducing the scope or lowering the reasoning effort.
- If the codex binary is missing or returns an unrecognized-flag error,
  the user may have an outdated version. Suggest they update the binary
  and check `codex --version`.
