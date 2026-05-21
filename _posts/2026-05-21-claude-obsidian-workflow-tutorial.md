---
title: "How to Use Claude Code With Obsidian as Your PKM and AI Workspace"
date: 2026-05-21 09:00:00 -0800
categories: [AI, Tutorial]
tags: [ai, claude, obsidian, pkm, tutorial]
author: jason
pin: false
image:
  path: /assets/img/posts/claude-obsidian-workflow.jpg
  alt: Claude Code and Obsidian workflow
---

A practical guide to connecting Claude Code to your Obsidian vault so that your AI assistant has persistent context, writes directly into your knowledge base, and works with agents that run automatically on a schedule.

**What you'll end up with:** Claude Code with read/write access to your vault, CLAUDE.md files that persist instructions across sessions, a memory system that carries behavioral context forward, and agent tasks triggered by editing markdown files on your phone.

**Prerequisites:** Claude Code CLI installed, Obsidian vault set up, a VPS or always-on machine (optional — for scheduled agents), Syncthing for vault sync.

---

**A note on PARA:** This tutorial uses the PARA folder structure for organizing the vault. PARA stands for **Projects, Areas, Resources, Archive** — four top-level folders that cover everything you're working on or referencing. Projects are active work with a finish line. Areas are ongoing responsibilities with no end date. Resources are reference material. Archive is everything inactive. You don't have to use PARA — but the path examples throughout this guide (`Projects/Agents/`, `Resources/`, etc.) assume it. Adapt them to whatever structure you use. If you're new to it, [ObsiBrain's PARA overview](https://www.obsibrain.com/docs/features/p.a.r.a-folder-structure) is a good primer.

---

## Step 1: Give Claude Code Access to Your Vault

Claude Code runs with access to whatever directories you allow. Point it at your vault root.

In your global Claude settings (`~/.claude/settings.json`):
```json
{
  "allowedPaths": [
    "/home/youruser/notes",
    "/home/youruser/dev/github"
  ]
}
```

Or launch Claude Code from within your vault directory:
```bash
cd ~/notes
claude
```

Claude can now read and write any file in your vault. Start simple: ask it to read a note, summarize a project file, or draft something into a specific directory.

---

## Step 2: Write Your Global CLAUDE.md

Create `~/.claude/CLAUDE.md`. Claude reads this at the start of every session. Write it like a brief for a new collaborator — tell it what it needs to know to be useful without explaining it again.

```markdown
# Claude Global Instructions

## Work Context
All projects under ~/dev/.

## Directory Structure
- ~/vault/ — Obsidian vault (PARA structure)
  - Inbox/ — unprocessed notes
  - Projects/ — active projects
  - Areas/ — ongoing responsibilities
  - Resources/ — reference material
  - Archive/ — done
- ~/dev/github/ — production code (treat with care)

## Standards
- Match existing file conventions before creating new ones
- Never commit to git without being asked
- Prefer editing existing files over creating new ones

## Communication
- Concise by default, no preamble
```

---

## Step 3: Add a Vault-Level CLAUDE.md

Create `~/vault/CLAUDE.md` (or wherever your vault root is). This gives Claude vault-specific context:

```markdown
## PKM Workflow
- PARA structure: Inbox, Projects, Areas, Resources, Archive
- New notes go in Inbox/ unless a target location is obvious
- Active projects live in Projects/ (status: active)
- Parked topics go in Areas/Research/ (status: research)
- Todoist handles tasks; Obsidian handles context/notes around tasks
```

---

## Step 4: Set Up the Memory System

Claude can maintain persistent memory files across sessions. Create a directory for them:

```bash
mkdir -p ~/.claude/projects/$(echo $HOME | tr '/' '-')/memory/
touch ~/.claude/projects/$(echo $HOME | tr '/' '-')/memory/MEMORY.md
```

Write a `MEMORY.md` index:
```markdown
# Memory

## User Preferences
- Concise responses, no preamble
- 2-space indentation

## Environment
- VPS: Ubuntu 24.04, Tailscale active
- Vault: ~/vault/
```

Add individual memory files for specific topics (feedback, projects, references). Claude will read the index at session start and fetch individual files as needed.

The key: when Claude corrects a behavior (e.g., "always dry-run before deleting"), save that to a memory file. Next session it remembers.

---

## Step 5: Create Agent Directories in Your Vault

Agents live in `Projects/Agents/`. Each agent is a folder with two required files:

```bash
mkdir -p ~/vault/Projects/Agents/my_agent/{knowledge,reports,logs}
```

**`CLAUDE.md`** — the agent's operating manual:
```markdown
# CLAUDE.md — my_agent

## Identity
[What this agent does]

## Working Directory
~/vault/Projects/Agents/my_agent/

## Schedule
<!-- schedule: day=1, hour=09, task=Weekly run -->

## Tasks

### Weekly run
[Step-by-step instructions for this task]

## Output Rules
- Write reports to reports/YYYY-MM-DD.md
- Update task-list.md when done
```

**`task-list.md`** — the trigger mechanism:
```markdown
# my_agent — Task List

## Backlog

## In Progress

## Done
```

---

## Step 6: Build the Dispatcher

The dispatcher is a bash script that runs on cron, checks all agent folders for `[trigger]` tags, and hands control to Claude Code.

Create `~/vault/Projects/Agents/dispatcher.sh`:

```bash
#!/bin/bash
AGENTS_DIR="$HOME/vault/Projects/Agents"
CLAUDE_CMD="$HOME/.local/bin/claude"
LOCK_FILE="/tmp/dispatcher.lock"

[ -f "$LOCK_FILE" ] && exit 0
touch "$LOCK_FILE"

for AGENT_DIR in "$AGENTS_DIR"/*_agent; do
  [ -d "$AGENT_DIR" ] || continue
  TASK_FILE="$AGENT_DIR/task-list.md"
  CLAUDE_FILE="$AGENT_DIR/CLAUDE.md"
  [ -f "$TASK_FILE" ] || continue
  [ -f "$CLAUDE_FILE" ] || continue
  [ -f "$AGENT_DIR/.env" ] && source "$AGENT_DIR/.env"

  grep -q "^- \[ \] \[trigger\]" "$TASK_FILE" || continue

  AGENT_LOCK="/tmp/$(basename $AGENT_DIR).lock"
  [ -f "$AGENT_LOCK" ] && continue
  touch "$AGENT_LOCK"

  TRIGGERED=$(grep "^- \[ \] \[trigger\]" "$TASK_FILE")
  sed -i "s/^- \[ \] \[trigger\]/- [ ] [in-progress] — started: $(date '+%Y-%m-%d %H:%M')/g" "$TASK_FILE"

  unset CLAUDECODE
  PROMPT="You are an agent. Read $CLAUDE_FILE first.
Your task file is at $TASK_FILE.
Triggered tasks: $TRIGGERED
Execute per your CLAUDE.md instructions.
When done, mark completed tasks [done] with timestamp in $TASK_FILE."

  echo "$PROMPT" | $CLAUDE_CMD --print --permission-mode bypassPermissions 2>&1

  rm -f "$AGENT_LOCK"
done

# Inject scheduled triggers
DAY=$(date '+%u')
HOUR=$(date '+%H')
for AGENT_DIR in "$AGENTS_DIR"/*_agent; do
  TASK_FILE="$AGENT_DIR/task-list.md"
  CLAUDE_FILE="$AGENT_DIR/CLAUDE.md"
  [ -f "$TASK_FILE" ] || continue
  while IFS= read -r line; do
    SCHED_DAY=$(echo "$line" | grep -oP 'day=\K[0-9]+')
    SCHED_HOUR=$(echo "$line" | grep -oP 'hour=\K[0-9]+')
    SCHED_TASK=$(echo "$line" | grep -oP 'task=\K[^-]+' | xargs)
    if [ "$DAY" == "$SCHED_DAY" ] && [ "$HOUR" == "$SCHED_HOUR" ]; then
      grep -q "^\- \[ \] \[trigger\] $SCHED_TASK" "$TASK_FILE" || \
        sed -i "/^## Backlog/a - [ ] [trigger] $SCHED_TASK" "$TASK_FILE"
    fi
  done < <(grep "<!-- schedule:" "$CLAUDE_FILE")
done

rm -f "$LOCK_FILE"
```

```bash
chmod +x ~/vault/Projects/Agents/dispatcher.sh
```

---

## Step 7: Set Up Cron

```bash
crontab -e
```

Add:
```
*/5 * * * * bash "/home/youruser/vault/Projects/Agents/dispatcher.sh" >> "/home/youruser/vault/Projects/Agents/dispatcher.log" 2>&1
```

---

## Step 8: Trigger an Agent From Your Phone

With Syncthing syncing your vault to your phone:

1. Open `task-list.md` in Obsidian on your phone
2. Add under Backlog: `- [ ] [trigger] Weekly run`
3. Save — Syncthing pushes to VPS in seconds
4. Dispatcher picks it up within 5 minutes
5. Report appears in your vault within ~10 minutes

---

## The Mental Model

**Obsidian is the file system. Claude is the process.** Files are state — they persist between sessions, sync across devices, and are readable by any tool. Claude reads and writes files. The vault becomes shared working memory: your notes, Claude's outputs, and agent task files all in one place.

CLAUDE.md files are how you give Claude durable instructions without repeating yourself. Memory files are how Claude accumulates context over time. Task files are how you trigger agents without logging into a server.

The whole system is just markdown files and a bash script. That's the point.

---

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
