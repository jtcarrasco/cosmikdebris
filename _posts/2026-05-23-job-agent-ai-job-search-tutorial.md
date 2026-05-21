---
title: "How to Build an AI Job Search Agent With Claude Code and Markdown Task Files"
date: 2026-05-23 09:00:00 -0800
categories: [AI, Tutorial]
tags: [ai, claude, automation, job-search, homelab, tutorial]
author: jason
pin: false
image:
  path: /assets/img/posts/job-search-agent.jpg
  alt: AI job search agent with Claude Code
---

A step-by-step guide to building an AI agent that searches multiple job boards, deduplicates results, and writes formatted reports to your Obsidian vault — automatically, on a schedule, with no custom server required.

**What you'll end up with:** An agent that runs 3-5x/week, hits multiple job APIs, deduplicates results, and writes a markdown report you can read in Obsidian or any text editor.

**Prerequisites:** Claude Code CLI installed, a VPS or always-on machine, Obsidian vault set up, API keys for JSearch and Adzuna (free tiers work).

---

## Step 1: Set Up the Agent Directory

Create the agent's folder structure:

```bash
mkdir -p ~/vault/Projects/Agents/job_agent/{knowledge,reports/{w2,freelance,pm},logs}
cd ~/vault/Projects/Agents/job_agent
```

Create the required files:
```bash
touch knowledge/criteria-w2.md
touch knowledge/criteria-freelance.md
touch knowledge/criteria-pm-w2.md
touch knowledge/seen-postings.md
touch knowledge/blacklist.md
touch task-list.md
```

---

## Step 2: Write the Agent's Instructions (CLAUDE.md)

The `CLAUDE.md` file is the agent's complete operating manual. Claude reads this before doing anything. Create `CLAUDE.md`:

```markdown
# CLAUDE.md — Job Search Agent

## Identity
Search for remote roles. Run three search types: W2 web dev, W2 project management, freelance/contract.

## Working Directory
~/vault/Projects/Agents/job_agent/

## Schedule
<!-- schedule: day=2, hour=08, task=W2 weekly run -->
<!-- schedule: day=4, hour=08, task=W2 weekly run -->
<!-- schedule: day=1, hour=08, task=Freelance run -->
<!-- schedule: day=3, hour=08, task=Freelance run -->

## Tasks

### W2 weekly run
Search for remote full-time W2 web development roles posted in the last 7 days.

**Sources:**
1. We Work Remotely RSS: https://weworkremotely.com/remote-jobs.rss
2. JSearch API: https://api.openwebninja.com/jsearch/search
   - Header: x-api-key: $JSEARCH_KEY
   - Params: query=remote web developer, date_posted=week, work_from_home=true
3. Adzuna API: https://api.adzuna.com/v1/api/jobs/us/search/1
   - Params: app_id=$ADZUNA_APP_ID, app_key=$ADZUNA_APP_KEY, what=remote web developer, max_days_old=7
   - Note: do NOT use where=remote — include "remote" in what= instead

**Output:** reports/w2/YYYY-MM-DD.md

[... add Freelance and PM sections similarly ...]

## Report Format
[Define your report structure here — table of results, posting notes, summary]

## Dedup Rules
- Read knowledge/seen-postings.md at start of every run
- Remove entries older than 90 days
- Skip any URL already in the list
- Append new URLs after writing report: YYYY-MM-DD https://...

## Output Rules
- Never overwrite a previous report — append -2 suffix if file exists
- Update task-list.md after every run — mark [done] with timestamp and result count
```

---

## Step 3: Get Your API Keys

**JSearch (OpenWeb Ninja):**
- Sign up at openwebninja.com
- Get your API key
- Tests against LinkedIn, Glassdoor, Indeed, ZipRecruiter in one call

**Adzuna:**
- Register at developer.adzuna.com
- Get `app_id` and `app_key` (free tier: 1000 calls/month)

Create a `.env` file in the agent directory:
```bash
cat > .env << 'EOF'
JSEARCH_KEY=your_jsearch_key_here
ADZUNA_APP_ID=your_adzuna_app_id
ADZUNA_APP_KEY=your_adzuna_app_key
EOF
chmod 600 .env
```

---

## Step 4: Write Your Search Criteria

Edit `knowledge/criteria-w2.md`:

```markdown
# W2 Web Dev Search Criteria

## Include
- Remote or remote-friendly positions
- Web development, front-end, full-stack
- WordPress, React, JavaScript, PHP, HTML/CSS
- US-based companies or open to US applicants

## Exclude
- Pure backend / DevOps / SRE roles
- Design-only (no dev component)
- Non-English job postings
- Anything in blacklist.md

## Salary
- Prefer $80k+ but don't exclude below if role is strong
```

Edit `knowledge/blacklist.md` with any companies or terms you want excluded:
```markdown
# Blacklist
- CompanyToAvoid
- MLM schemes
- Commission-only
```

---

## Step 5: Create the Task List

Edit `task-list.md`:

```markdown
# job_agent — Task List

> Add a [trigger] tag to run manually:
> `- [ ] [trigger] W2 weekly run`

---

## Backlog

---

## In Progress

---

## Done
```

---

## Step 6: Build the Dispatcher

The dispatcher is a bash script that runs on cron, checks each agent folder for `[trigger]` tags, and runs Claude Code with the appropriate context.

Create `~/vault/Projects/Agents/dispatcher.sh`:

```bash
#!/bin/bash

AGENTS_DIR="$HOME/vault/Projects/Agents"
CLAUDE_CMD="$HOME/.local/bin/claude"   # Path to your Claude Code binary
LOCK_FILE="/tmp/dispatcher.lock"

# Prevent overlapping runs
if [ -f "$LOCK_FILE" ]; then
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] Already running. Skipping."
  exit 0
fi
touch "$LOCK_FILE"

echo "[$(date '+%Y-%m-%d %H:%M:%S')] Dispatcher started."

# Loop all agent folders
for AGENT_DIR in "$AGENTS_DIR"/*_agent; do
  [ -d "$AGENT_DIR" ] || continue

  AGENT_NAME=$(basename "$AGENT_DIR")
  TASK_FILE="$AGENT_DIR/task-list.md"
  CLAUDE_FILE="$AGENT_DIR/CLAUDE.md"
  AGENT_LOCK="/tmp/$AGENT_NAME.lock"

  [ -f "$TASK_FILE" ] || continue
  [ -f "$CLAUDE_FILE" ] || continue

  # Load agent env
  [ -f "$AGENT_DIR/.env" ] && source "$AGENT_DIR/.env"

  # Check for trigger tag
  if ! grep -q "^- \[ \] \[trigger\]" "$TASK_FILE"; then
    continue
  fi

  # Skip if already running
  if [ -f "$AGENT_LOCK" ]; then
    echo "[$AGENT_NAME] Already running. Skipping."
    continue
  fi

  touch "$AGENT_LOCK"

  # Extract triggered tasks
  TRIGGERED_TASKS=$(grep "^- \[ \] \[trigger\]" "$TASK_FILE")

  # Mark as in-progress
  sed -i "s/^- \[ \] \[trigger\]/- [ ] [in-progress] — started: $(date '+%Y-%m-%d %H:%M')/g" "$TASK_FILE"

  # Run Claude
  unset CLAUDECODE
  PROMPT="You are an agent. Your instructions are in $CLAUDE_FILE.
Read that file first. Your task file is at $TASK_FILE.
Triggered tasks: $TRIGGERED_TASKS
Execute per your CLAUDE.md instructions.
When done, update $TASK_FILE — mark completed tasks [done] with timestamp."

  echo "$PROMPT" | $CLAUDE_CMD --print --permission-mode bypassPermissions 2>&1
  
  rm -f "$AGENT_LOCK"
  echo "[$AGENT_NAME] Done."
done

# Scheduled trigger injection
DAY_OF_WEEK=$(date '+%u')
CURRENT_HOUR=$(date '+%H')

for AGENT_DIR in "$AGENTS_DIR"/*_agent; do
  [ -d "$AGENT_DIR" ] || continue
  TASK_FILE="$AGENT_DIR/task-list.md"
  CLAUDE_FILE="$AGENT_DIR/CLAUDE.md"
  [ -f "$TASK_FILE" ] || continue

  while IFS= read -r line; do
    SCHED_DAY=$(echo "$line" | grep -oP 'day=\K[0-9]+')
    SCHED_HOUR=$(echo "$line" | grep -oP 'hour=\K[0-9]+')
    SCHED_TASK=$(echo "$line" | grep -oP 'task=\K[^-]+' | xargs)

    if [ "$DAY_OF_WEEK" == "$SCHED_DAY" ] && [ "$CURRENT_HOUR" == "$SCHED_HOUR" ]; then
      if ! grep -q "^\- \[ \] \[trigger\] $SCHED_TASK" "$TASK_FILE"; then
        sed -i "/^## Backlog/a - [ ] [trigger] $SCHED_TASK" "$TASK_FILE"
        echo "[$(basename $AGENT_DIR)] Injected scheduled trigger: $SCHED_TASK"
      fi
    fi
  done < <(grep "<!-- schedule:" "$CLAUDE_FILE")
done

rm -f "$LOCK_FILE"
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Dispatcher finished."
```

Make it executable:
```bash
chmod +x ~/vault/Projects/Agents/dispatcher.sh
```

---

## Step 7: Set Up the Cron Job

```bash
crontab -e
```

Add:
```
*/5 * * * * bash "/home/youruser/vault/Projects/Agents/dispatcher.sh" >> "/home/youruser/vault/Projects/Agents/dispatcher.log" 2>&1
```

This runs the dispatcher every 5 minutes. It checks for triggers and injects scheduled ones when the day/hour match.

---

## Step 8: Test a Manual Run

Add a trigger to `task-list.md`:
```markdown
## Backlog
- [ ] [trigger] W2 weekly run
```

Wait up to 5 minutes for cron to fire, or run the dispatcher manually:
```bash
bash ~/vault/Projects/Agents/dispatcher.sh
```

Check the output in `reports/w2/YYYY-MM-DD.md` and the dispatcher log.

---

## Triggering From Your Phone

If your vault syncs via Syncthing (or iCloud, Dropbox, etc.):

1. Open `task-list.md` in Obsidian on your phone
2. Change `- [ ] W2 weekly run` to `- [ ] [trigger] W2 weekly run`
3. Save — Syncthing pushes it to the VPS
4. Dispatcher picks it up within 5 minutes
5. Report appears in your vault within ~10 minutes

---

## Checking Reports

Reports land in `reports/w2/YYYY-MM-DD.md` (or `freelance/`, `pm/`). Open them in Obsidian. The table format renders cleanly and links are clickable.

---

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
