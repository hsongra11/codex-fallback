# codex-fallback

Keep building when Claude hits the limit. Don't start from zero in Codex.

Works across **Claude Code CLI**, **Claude coWork (desktop/web)**, **Codex CLI**, and the **Codex desktop app**. Syncs your Claude project context into Codex's `AGENTS.md` format so when you switch tools, Codex knows what Claude knew.

---

## The problem

You're mid-session in Claude Code or coWork. You hit the usage cap. You open Codex to keep going.

Codex has no idea what you were working on. No project conventions. No prior decisions. No memory. No context. You spend the next 20 minutes re-explaining things Claude already understood.

The quality gap between Claude and GPT is real. But a huge chunk of the *experienced* gap is context, not model intelligence. Codex without context is significantly worse than Codex with context.

## What this does

**`setup.sh`** — Run this in any project directory. It reads your Claude Code context and writes an `AGENTS.md` file that both Codex CLI and the Codex desktop app automatically pick up. One command, then Codex just works with your context.

**`/codex-fallback lean`** — Claude-side skill (works in both CLI and coWork) that switches to quota-conservation mode so you hit the wall later.

**`/codex-fallback <task>`** — Claude-side skill that runs setup.sh and hands off a task to Codex directly.

---

## Install

```bash
git clone https://github.com/hsongra11/codex-fallback.git ~/.claude/skills/codex-fallback
```

### Prerequisites

- [Claude Code](https://claude.ai/code) — CLI, desktop app (coWork), or web app
- [OpenAI Codex](https://github.com/openai/codex) — CLI (`npm install -g @openai/codex && codex login`) or [desktop app](https://github.com/openai/codex) (`codex app`)

---

## Usage by tool

### Claude Code CLI + Codex CLI

The terminal-to-terminal flow. You're in Claude Code CLI, you hit the limit, you switch to Codex CLI.

```bash
# In your project directory, sync context
~/.claude/skills/codex-fallback/setup.sh

# Now use Codex CLI — it has your context
codex
```

`setup.sh` writes `AGENTS.md` in your project root. Codex CLI reads it automatically as system instructions.

---

### Claude coWork (desktop/web app) + Codex desktop app

The app-to-app flow. You're in Claude coWork, you hit the limit, you switch to the Codex desktop app.

**Step 1: Run setup.sh from coWork**

In your Claude coWork session, just ask:

```
Run ~/.claude/skills/codex-fallback/setup.sh
```

Claude coWork has terminal access. It will run the script and sync your context into `AGENTS.md`.

Or if you prefer, open a terminal yourself and run it manually:

```bash
cd /path/to/your/project
~/.claude/skills/codex-fallback/setup.sh
```

**Step 2: Open the Codex desktop app**

```bash
codex app
```

Or launch it from your dock/applications. Open your project folder. The Codex app reads the `AGENTS.md` file from the project root automatically, just like the CLI does.

That's it. Same context, different app.

**Step 3 (optional): Use lean mode before you hit the wall**

In Claude coWork, type:

```
/codex-fallback lean
```

This switches Claude to conservation mode (haiku for simple ops, terse output, aggressive batching) so your session lasts longer before capping out.

---

### Claude coWork + Codex CLI

You're in the Claude desktop app but prefer Codex in the terminal.

```
# In Claude coWork, ask it to run the sync
Run ~/.claude/skills/codex-fallback/setup.sh
```

Then open a terminal:

```bash
cd /path/to/your/project
codex
```

---

### Claude Code CLI + Codex desktop app

You're in Claude Code terminal but prefer the Codex GUI.

```bash
~/.claude/skills/codex-fallback/setup.sh
codex app
```

Open your project folder in the Codex app. It picks up the `AGENTS.md`.

---

## What gets synced

| Claude source | What it contains | Where it goes |
|---|---|---|
| `CLAUDE.md` (project) | Build instructions, conventions, code style | `AGENTS.md` (project root) |
| `CLAUDE.md` (global) | Cross-project preferences | `AGENTS.md` (project root) |
| Claude memory files | Prior decisions, user context, project notes | `AGENTS.md` (project root) |
| `git status` + `git log` | Current branch, recent commits, working tree | `AGENTS.md` (project root) |
| `package.json` / `pyproject.toml` | Dependencies, project type | `AGENTS.md` (project root) |
| Directory structure | File layout (top 2 levels) | `AGENTS.md` (project root) |

The script also auto-trusts the project in Codex's `~/.codex/config.toml` so you don't get permission prompts in either the CLI or the desktop app.

---

## Examples

### Picking up where Claude left off

You were building a Next.js app in Claude coWork. You hit the limit mid-feature.

In coWork, type:
```
Run ~/.claude/skills/codex-fallback/setup.sh
```

Output:
```
codex-fallback: syncing Claude context → Codex
project: /Users/harsh/projects/dashboard-app

  found: CLAUDE.md (project)
  found: memory files (3 files)
  found: git context (branch: feat/billing)

  wrote: AGENTS.md (67 lines)
  exists: project already trusted in codex config

done. run 'codex' in this directory and it'll have your Claude context.
```

Open Codex (CLI or app) and continue:
```
Add the remaining API routes for the billing module.
Follow the same pattern as the existing routes in src/routes/users.ts.
```

Codex reads the `AGENTS.md`, sees your project conventions, understands the existing patterns, and implements accordingly.

### Continuing a coding task

```bash
~/.claude/skills/codex-fallback/setup.sh
```

In Codex (CLI or app):
```
Finish the webhook handler in src/webhooks/stripe.ts — 
event parsing is done, need subscription lifecycle handlers 
(created, updated, deleted) and the idempotency check.
Follow the same pattern as src/webhooks/github.ts
```

Codex reads your AGENTS.md, sees the project conventions, understands the existing patterns, and implements accordingly.

### Non-interactive (scripts or CI)

```bash
~/.claude/skills/codex-fallback/setup.sh && codex exec "add input validation to all API routes in src/routes/" -s full-auto
```

---

## The Claude-side skill

Works in both Claude Code CLI and Claude coWork. Use *before* hitting the wall.

### Lean mode — stretch your quota

```
/codex-fallback lean
```

Switches Claude to conservation mode for the rest of the session:
- Haiku for simple operations (reads, basic edits, search, file creation)
- One-line confirmations, no explanations
- Aggressive batching of parallel tool calls
- No re-reading files already in context

```
You:    /codex-fallback lean
Claude: Lean mode active. Conserving quota.

You:    Add input validation to all the form components in src/components/forms/
Claude: Done. Added validation to 5 form components.
```

Exit with `/codex-fallback unlean`.

### Direct handoff — Claude runs Codex for you

```
/codex-fallback finish the billing integration tests
```

Claude gathers your context, writes AGENTS.md, and runs `codex exec` with the task. You see the Codex output inline. Works in CLI. In coWork, Claude runs the terminal command through its shell access.

---

## Auto-suggestion

Add this to `~/.claude/CLAUDE.md` so Claude proactively suggests the fallback when you're running low. This works across both CLI and coWork since they read the same config:

```markdown
## Usage Limit Awareness

When you encounter rate limiting, usage cap warnings, or the user mentions
hitting limits, suggest:

> You're near your usage limit. Two options:
> - `/codex-fallback lean` — conservation mode
> - Run `~/.claude/skills/codex-fallback/setup.sh` then switch to Codex (CLI or app)
```

---

## How it works

```
┌──────────────────────────────────┐
│  Claude Code CLI / coWork app    │
│                                  │
│  CLAUDE.md    Memory files       │
│  Git state    Project structure  │     setup.sh
│  Package config                  │  ─────────────▶  AGENTS.md
│                                  │                  (project root)
└──────────────────────────────────┘

┌──────────────────────────────────┐
│  Codex CLI / Codex desktop app   │
│                                  │
│  Reads AGENTS.md automatically   │  ◀── both CLI and app read this
│  as system instructions          │      no flags needed
│                                  │
└──────────────────────────────────┘
```

The bridge is `AGENTS.md`. Claude's world gets serialized into it. Both Codex CLI and the Codex desktop app read it natively from the project root. No custom integration needed.

---

## Re-syncing

Run `setup.sh` again whenever you want to refresh the context:

- Starting a new Codex session after a Claude session
- Made significant decisions in Claude that Codex should know about
- Switched branches
- Coming back to a project after a break

The script overwrites `AGENTS.md` each time.

---

## Limitations

- **The model gap is real.** Context injection helps a lot, but Claude is better than GPT for most building tasks right now. You will review Codex output more carefully.
- **Memory is a snapshot, not live sync.** `setup.sh` captures context at the moment you run it. It doesn't update as Codex works. Re-run it when you need fresh context.
- **AGENTS.md is in your project root.** Add it to `.gitignore` if you don't want it committed. It may contain memory content you consider private.
- **Can't bypass Claude's usage limit.** This is a workaround for the interruption, not a hack.
- **Codex CLI must be installed separately.** This skill wraps it, doesn't bundle it.

---

## Uninstall

```bash
rm -rf ~/.claude/skills/codex-fallback
```

Remove `AGENTS.md` from any project directories where you ran setup.sh.

---

## Why this exists

I hit my Claude Code limit mid-session. Switched to Codex to keep going. The quality drop was instant and obvious. Not a benchmark thing. A "sitting in the terminal doing real work" thing.

Half of that drop was the model difference. The other half was Codex having zero context about what I was building, what conventions I was following, and what decisions I'd already made.

This tool fixes the second half.

---

## License

MIT
