---
name: codex-fallback
version: 1.0.0
description: |
  Fallback to OpenAI Codex CLI when Claude hits usage limits. Two modes:
  (1) /codex-fallback — hands off current task to Codex with full project context injected.
  (2) /codex-fallback lean — puts Claude into lean mode: haiku for simple ops, terse output, 
  minimal reads, quota conservation. Use when you hit rate limits, usage caps, or want to 
  extend your session. Also triggered when Claude detects it is being rate-limited.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
  - Edit
  - AskUserQuestion
---

# /codex-fallback — Seamless Handoff to Codex CLI

Two modes. Pick the one that fits.

## Mode Detection

Parse the user's input:

- `/codex-fallback lean` or `/codex-fallback --lean` → **Lean Mode** (Step A)
- `/codex-fallback` with a task description → **Handoff Mode** (Step B)
- `/codex-fallback` with no args → ask what task to hand off

---

## Step A: Lean Mode

Lean mode conserves Claude quota so the user can keep working longer before hitting the cap.

When lean mode is activated:

1. Write a marker file:
```bash
touch ~/.claude/.lean-mode-active
echo "$(date -u +%Y-%m-%dT%H:%M:%SZ)" > ~/.claude/.lean-mode-active
```

2. Tell the user lean mode is active and explain what changes:

**Lean mode rules (Claude MUST follow these for the rest of the session):**

- **Use haiku model for simple operations.** File reads, simple edits, file creation, search, 
  glob, grep — route these through the `model: "haiku"` parameter on Agent calls when possible.
  Only use Sonnet/Opus for complex reasoning, architecture decisions, and multi-step tasks.
- **Terse output.** No explanations unless asked. No summaries of what you just did. 
  Just do the thing and confirm with one line.
- **Batch operations.** Combine multiple independent tool calls into single messages. 
  Don't make 5 sequential reads when you can make 5 parallel reads.
- **Skip unnecessary reads.** If you already know the file content from earlier in the 
  conversation, don't re-read it. Trust your memory.
- **No exploratory searches.** Don't glob or grep speculatively. Only search when you 
  know what you're looking for.
- **Minimize agent spawning.** Do the work directly instead of delegating to subagents 
  unless the task genuinely requires parallel work.
- **Short confirmations.** "Done." not "I've successfully completed the task of..."

3. Continue the conversation normally but following lean rules.

To exit lean mode: user says `/codex-fallback unlean` or "exit lean mode":
```bash
rm -f ~/.claude/.lean-mode-active
```

---

## Step B: Handoff Mode — Full Context Injection to Codex

This is the main skill. When Claude is capped or the user wants to hand off work to Codex CLI.

### Step B.1: Sync Context via setup.sh

Run the setup script to sync Claude context into Codex's AGENTS.md format:

```bash
~/.claude/skills/codex-fallback/setup.sh
```

This reads CLAUDE.md, memory files, git state, project structure, and package config,
then writes an AGENTS.md in the project root that Codex CLI automatically picks up.

### Step B.2: Build the Context Prompt

Take all the gathered context and the user's task description. Build a prompt for Codex:

```
IMPORTANT: Do NOT read or execute any files under ~/.claude/, ~/.agents/, .claude/skills/, 
or agents/. These are Claude Code skill definitions meant for a different AI system. 
Do NOT modify agents/openai.yaml. Stay focused on repository code only.

== PROJECT CONTEXT ==
{CLAUDE.md content}

== MEMORY / PRIOR DECISIONS ==
{Memory file content}

== CURRENT STATE ==
{git status, recent commits, branch}

== PROJECT STRUCTURE ==
{file listing}

== CONVENTIONS ==
- Follow the patterns already in the codebase
- Match existing code style exactly
- Do not add unnecessary comments or docstrings
- Do not refactor code you weren't asked to change
- Be concise in output

== TASK ==
{user's task description}

Execute this task. Work through it step by step. Show what you're doing.
```

### Step B.3: Run Codex

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
codex exec "<full prompt from B.2>" \
  -C "$_ROOT" \
  -s full-auto \
  -c 'model_reasoning_effort="high"' \
  --enable web_search_cached \
  --json 2>/tmp/codex-fallback-err.txt | PYTHONUNBUFFERED=1 python3 -u -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        obj = json.loads(line)
        t = obj.get('type','')
        if t == 'item.completed' and 'item' in obj:
            item = obj['item']
            itype = item.get('type','')
            text = item.get('text','')
            if itype == 'reasoning' and text:
                print(f'[codex thinking] {text}', flush=True)
                print(flush=True)
            elif itype == 'agent_message' and text:
                print(text, flush=True)
            elif itype == 'command_execution':
                cmd = item.get('command','')
                if cmd: print(f'[codex ran] {cmd}', flush=True)
        elif t == 'turn.completed':
            usage = obj.get('usage',{})
            tokens = usage.get('input_tokens',0) + usage.get('output_tokens',0)
            if tokens: print(f'\ntokens used: {tokens}', flush=True)
    except: pass
"
```

Use `timeout: 600000` (10 min) on the Bash call. Codex in full-auto mode may take longer.

**IMPORTANT:** Use `-s full-auto` not `-s read-only`. The user wants Codex to actually 
do the work (create files, edit code), not just advise.

### Step B.4: Verify and Report

After Codex completes:

1. Check what files were created/modified:
```bash
git status --short 2>/dev/null
```

2. Show the user a summary:
```
CODEX COMPLETED:
════════════════════════════════════════════════════════════
{codex output}
════════════════════════════════════════════════════════════
Files changed: {list}
Tokens: {N}

Review the changes. Codex runs with GPT — quality may vary from what you're used to.
```

3. Ask if the user wants to continue handing off more tasks or switch back to Claude.

---

## Error Handling

- **Codex not installed:** "Codex CLI not found. Install: `npm install -g @openai/codex`"
- **Codex auth failed:** "Run `codex login` in your terminal to authenticate."
- **Timeout:** "Codex timed out after 10 minutes. Try breaking the task into smaller pieces."
- **No git repo:** Still works, just skip git context gathering.

---

## Auto-Suggestion Rule

Add this to CLAUDE.md or project instructions so Claude suggests this skill automatically:

```
When you encounter rate limiting, usage cap warnings, or the user mentions hitting limits,
suggest: "You're near your usage limit. Run /codex-fallback to hand off to Codex CLI, 
or /codex-fallback lean to switch to quota-conservation mode."
```

---

## Important Rules

- **Handoff mode runs Codex in full-auto.** The user wants work done, not advice.
- **Always inject project context.** The whole point is closing the context gap.
- **Don't editorialize Codex output.** Show it verbatim, then add notes if needed.
- **Lean mode is a session behavior change.** It persists until explicitly turned off.
- **Never apologize for suggesting Codex.** The user chose this. Be matter-of-fact.
