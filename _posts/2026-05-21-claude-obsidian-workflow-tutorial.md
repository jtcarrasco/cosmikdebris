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

Every time I started a [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) session, I'd spend the first five minutes re-explaining who I am, what I'm working on, and why my project structure looks the way it does. Same context, over and over, like onboarding a contractor who shows up with no memory of the last twelve times they were here.

The fix turned out to be embarrassingly simple: give Claude a file to read.

**What you'll end up with:** Claude Code with read/write access to your [Obsidian](https://obsidian.md/) vault, CLAUDE.md files that persist instructions across sessions, a memory system that carries behavioral context forward, and agent tasks triggered by editing markdown files on your phone.

**Prerequisites:** Claude Code CLI installed, Obsidian vault set up, a VPS or always-on machine (optional, for scheduled agents), [Syncthing](https://syncthing.net/) for vault sync.

---

**A note on PARA:** This tutorial uses the PARA folder structure for organizing the vault. PARA stands for **Projects, Areas, Resources, Archive**: four top-level folders that cover everything you're working on or referencing. Projects are active work with a finish line. Areas are ongoing responsibilities with no end date. Resources are reference material. Archive is everything inactive. You don't have to use PARA, but the path examples throughout (`Projects/Agents/`, `Resources/`, etc.) assume it. Adapt to whatever structure you use. If you're new to it, [ObsiBrain's PARA overview](https://www.obsibrain.com/docs/features/p.a.r.a-folder-structure) is a good primer.

---

## Step 1: Give Claude Code Access to Your Vault

Claude Code runs with access to whatever directories you allow. Point it at your vault root. For this tutorial we will use `notes` as our vault.

In your global Claude settings (`~/.claude/settings.json`):
```json
{
  "allowedPaths": [
    "/home/youruser/notes",
    "/home/youruser/github"
  ]
}
```

Or just launch Claude Code from within your vault directory:
```bash
cd ~/notes
claude
```

Claude can now read and write any file in your vault. Start simple: ask it to read a note, summarize a project file, or draft something into a specific location. Verify it can actually find things before building anything on top of it.

---

## Step 2: Keep Claude Running with tmux

If you're running Claude Code on a VPS, you'll hit a problem fast: SSH sessions die. Close your laptop, lose your connection, step away for an hour, the session is gone, and so is any Claude session running inside it.

[tmux](https://github.com/tmux/tmux) fixes this. It keeps terminal sessions alive on the server independent of your SSH connection. You connect, attach to your tmux session, do work, detach, close your laptop. The session keeps running. Come back from your phone, reattach, everything's exactly where you left it.

```bash
# Install tmux
apt install tmux -y

# Start a persistent session named "claude"
tmux new-session -s claude

# Inside the session, launch Claude Code from your vault
cd ~/vault
claude
```

Detach without killing anything: `Ctrl+B`, then `D`. You're back at a normal shell. The Claude session is still running on the server.

Reconnect from anywhere:
```bash
ssh youruser@your-machine.tail35225b.ts.net
tmux attach -t claude
```

The basics you'll use constantly:
- `Ctrl+B, D` — detach (session keeps running)
- `Ctrl+B, C` — new window within the session
- `tmux ls` — list running sessions
- `tmux attach -t claude` — reattach by name

The dispatcher from Step 7 runs via cron and doesn't need tmux. It handles its own Claude processes. tmux is for interactive sessions where you're actively working, and for agent runs you want to watch in real time.

---

## Step 3: Write Your Global CLAUDE.md

Create `~/.claude/CLAUDE.md`. Claude reads this at the start of every session. Think of it as the brief you'd hand a new collaborator: what they need to be useful without you explaining it from scratch every time. Keep it brief as you can add `CLAUDE.md` to your projects for more specific guidelines. There are a lot of examples out there with good, better, best global `CLAUDE.md` guidelines, download them, edit them, try them out.

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

This file is the reason you stop typing "by the way, I prefer concise responses" into every session like it's a magic incantation.

---

## Step 4: Add a Vault-Level CLAUDE.md

Create `~/vault/CLAUDE.md` at the vault root. This is vault-specific context that doesn't belong in global settings:

```markdown
## PKM Workflow
- PARA structure: Inbox, Projects, Areas, Resources, Archive
- New notes go in Inbox/ unless a target location is obvious
- Active projects live in Projects/ (status: active)
- Parked topics go in Areas/Research/ (status: research)
- Todoist handles tasks; Obsidian handles context/notes around tasks
```

Nested CLAUDE.md files at the subdirectory level work the same way. Claude picks them up when it navigates into those directories. You can get as granular as you want, but start with just global plus vault-root and add more only when you feel a gap.

---

## Step 5: Set Up the Memory System

A CLAUDE.md file tells Claude how you want it to behave. A memory file is where that behavior evolves over time. The distinction matters: CLAUDE.md is the standing brief, memory is the accumulated context from working together.

Create a memory directory:

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

Add individual memory files for specific topics: feedback patterns, project context, reference data Claude shouldn't have to re-derive. The MEMORY.md file is an index; Claude reads it at session start and fetches individual files as needed.

The workflow that makes this useful: when Claude does something you want it to always do (or stop doing), save that correction to a memory file. "Always dry-run before deleting" goes in `feedback_deletions.md`. Next session it remembers. You stop being the error-correction layer.

---

## Step 6: Create Agent Directories in Your Vault

This is where things get interesting. Agents live in `Projects/Agents/`. Each agent is a folder containing its operating manual (CLAUDE.md) and its task list, and that's enough to build something that runs on a schedule and writes results back into your vault.

```bash
mkdir -p ~/vault/Projects/Agents/my_agent/{knowledge,reports,logs}
```

**`CLAUDE.md`**: the agent's job description
```markdown
# CLAUDE.md — my_agent

## Identity
[What this agent does in one sentence]

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

**`task-list.md`**: the trigger mechanism
```markdown
# my_agent — Task List

## Backlog

## In Progress

## Done
```

The task list is deceptively simple. It's the file you'll edit from your phone when you want to kick something off manually, and the file the dispatcher watches to know when to run.

---

## Step 7: Build the Dispatcher

The dispatcher is the beating heart of the whole system: a bash script that runs on cron, scans all agent folders for `[trigger]` tags in their task lists, and hands control to Claude Code. It's about 50 lines of bash doing something that would have required a real orchestration system a few years ago.

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

The lock file at the top prevents the dispatcher from running twice if a previous Claude session is still in progress. The per-agent lock does the same for individual agents. These guard clauses are worth understanding. A dispatcher that piles up concurrent Claude sessions will burn through your API budget fast.

---

## Step 8: Set Up Cron

```bash
crontab -e
```

Add:
```
*/5 * * * * bash "/home/youruser/vault/Projects/Agents/dispatcher.sh" >> "/home/youruser/vault/Projects/Agents/dispatcher.log" 2>&1
```

Every 5 minutes, the dispatcher wakes up, checks for triggers, runs any that are pending, and goes back to sleep. The log file is your audit trail when something doesn't run as expected.

---

## Step 9: Trigger an Agent From Your Phone

This is the payoff for all the setup above, and it's satisfying the first time it works.

With Syncthing syncing your vault to your phone:

1. Open `task-list.md` in Obsidian on your phone
2. Add under Backlog: `- [ ] [trigger] Weekly run`
3. Save. Syncthing pushes to the VPS in seconds
4. Dispatcher picks it up within 5 minutes
5. Report appears in your vault within ~10 minutes

You triggered an AI agent running on a server, from your phone, by editing a text file. No app, no webhook, no API key to manage. Just a markdown checkbox.

---

## Step 10: Connect Discord as a Trigger Interface

The phone trigger from Step 9 works, but editing a markdown file isn't always the fastest path. Discord lets you type a message in a channel and have an agent running within seconds, from anywhere, on any device, without opening your vault.

**Create a Discord bot:**

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications) and click **New Application**
2. Name it (e.g., "homelab-agent"), then go to **Bot** in the left sidebar
3. Click **Add Bot**, then **Reset Token** and copy the token. This is your `DISCORD_BOT_TOKEN`. Store it securely, you won't see it again
4. Under **Privileged Gateway Intents**, enable **Message Content Intent**
5. Go to **OAuth2 > URL Generator**: check `bot` under Scopes, then check `Send Messages`, `Read Message History`, and `Read Messages/View Channels` under Bot Permissions
6. Copy the generated URL, open it in a browser, and invite the bot to your server

**Get your channel ID:**

In Discord, go to User Settings > Advanced and enable **Developer Mode**. Then right-click the channel you want the bot to watch and click **Copy Channel ID**.

**Install the Discord MCP plugin:**

```bash
claude mcp add discord-bot -- npx -y @anthropic/mcp-server-discord
```

Or add it manually to `~/.claude/settings.json`:
```json
{
  "mcpServers": {
    "discord": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-discord"],
      "env": {
        "DISCORD_BOT_TOKEN": "your-bot-token-here",
        "DISCORD_CHANNEL_ID": "your-channel-id-here"
      }
    }
  }
}
```

**Wire Discord to the dispatcher:**

Create an inbox directory:
```bash
mkdir -p ~/.claude/channels/discord/inbox
```

Add a watcher block to your `dispatcher.sh` (before the final `rm -f "$LOCK_FILE"`):

```bash
DISCORD_INBOX="$HOME/.claude/channels/discord/inbox"
if compgen -G "$DISCORD_INBOX"/*.txt "$DISCORD_INBOX"/*.md > /dev/null 2>&1; then
  for MSG_FILE in "$DISCORD_INBOX"/*.txt "$DISCORD_INBOX"/*.md; do
    [ -f "$MSG_FILE" ] || continue
    MSG=$(cat "$MSG_FILE")
    # Route to the right agent based on message prefix
    if echo "$MSG" | grep -q "^wiki:"; then
      URL=$(echo "$MSG" | sed 's/^wiki: *//')
      echo "$URL" > "$HOME/vault/Inbox/wiki_queue/discord-$(date +%s).md"
    fi
    rm -f "$MSG_FILE"
  done
fi
```

**The result:**

Type `wiki: https://example.com` in your Discord channel. The bot writes it to the inbox. The dispatcher picks it up within 5 minutes, runs your wiki agent against that URL, and posts the result back to Discord. No SSH, no vault editing, no context switching.

You can extend the routing block for any trigger pattern: `job brief`, `lint`, `report`, whatever your agents support. One channel, full control of your homelab agent stack.

Discord is just one option. Claude Code's MCP plugin ecosystem supports other messaging platforms including [Telegram](https://telegram.org/), [Slack](https://slack.com/), and others. The pattern is the same regardless of platform: messages land in an inbox directory, the dispatcher routes them to the right agent. Swap the MCP plugin and update the inbox path, and the rest of the system doesn't change.

---

## The Mental Model

**Obsidian is the file system. Claude is the process.** Files are state: they persist between sessions, sync across devices, and are readable by any tool. Claude reads and writes files. The vault becomes shared working memory: your notes, Claude's outputs, and agent task files all in one place.

CLAUDE.md files are how you give Claude durable instructions without repeating yourself. Memory files are how Claude accumulates context over time. Task files are how you trigger agents without logging into anything.

The whole system is markdown files and a bash script. That's not a limitation. That's the point. Plain text ages well. It works with every editor, syncs with everything, and never requires a migration.

---

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
