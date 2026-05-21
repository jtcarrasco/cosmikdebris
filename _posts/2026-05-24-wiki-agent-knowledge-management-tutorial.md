---
title: "How to Build an AI Wiki Agent That Turns Articles Into a Compounding Knowledge Base"
date: 2026-05-24 09:00:00 -0800
categories: [AI, Tutorial]
tags: [ai, claude, knowledge-management, obsidian, tutorial]
author: jason
pin: false
image:
  path: /assets/img/posts/wiki-agent-knowledge-manager.jpg
  alt: AI wiki agent for knowledge management
---

A step-by-step guide to building an AI agent that reads articles and YouTube videos, extracts concepts and entities, and builds a structured, interlinked wiki in your Obsidian vault — automatically.

**What you'll end up with:** A growing wiki where every new article you queue is automatically integrated — summaries, concept pages, entity pages, cross-references, and a weekly health check.

**Prerequisites:** Claude Code CLI, Obsidian vault, Syncthing or another sync tool, `yt-dlp` installed (for YouTube support).

---

## Step 1: Set Up the Directory Structure

```bash
# Agent folder
mkdir -p ~/vault/Projects/Agents/wiki_agent/{knowledge,logs}

# Wiki in your vault
mkdir -p ~/vault/Resources/Wiki/{sources,concepts,entities}
mkdir -p ~/vault/Inbox/wiki_queue/processed
```

Create the initial index and log:

```bash
cat > ~/vault/Resources/Wiki/index.md << 'EOF'
# Wiki Index

_Last updated: YYYY-MM-DD — 0 sources, 0 concepts, 0 entities_

## Sources
| Page | Summary | Created |
|------|---------|---------|

## Concepts
| Page | Summary | Updated |
|------|---------|---------|

## Entities
| Page | Type | Summary | Updated |
|------|------|---------|---------|
EOF

touch ~/vault/Resources/Wiki/log.md
```

---

## Step 2: Write the Agent Instructions (CLAUDE.md)

Create `~/vault/Projects/Agents/wiki_agent/CLAUDE.md`:

```markdown
# CLAUDE.md — wiki_agent

## Identity
Process source documents into a wiki at Resources/Wiki/. Knowledge compounds with every source added.

## Working Directory
~/vault/

## Page Types
- **sources/** — one per article/video. Summary, key concepts, key entities, notable claims.
- **concepts/** — ideas, patterns, frameworks that appear across multiple sources.
- **entities/** — specific tools, people, companies, products.

All pages use frontmatter:
---
type: source|concept|entity
title:
tags: []
sources: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

## Tasks

### process-queue
Process all URLs in Inbox/wiki_queue/ (skip processed/ subfolder).

For each file:
1. Extract URLs (bare URLs or markdown links)
2. For YouTube URLs (youtube.com/watch, youtu.be/): use yt-dlp to get transcript
   - Command: yt-dlp --skip-download --write-auto-subs --sub-langs en --convert-subs srt -o "/tmp/wiki-yt-%(id)s.%(ext)s" "URL"
   - Strip timestamp lines, use cleaned text as source
3. For all other URLs: fetch with WebFetch
4. Run ingest pipeline:
   - Write sources/<slug>.md
   - Update or create concept pages
   - Update or create entity pages
   - Update index.md
   - Append to log.md: ## [YYYY-MM-DD] ingest | <title> | N pages touched
5. Move queue file to Inbox/wiki_queue/processed/

### Wiki lint
Health check: index gaps, missing pages, orphans, missing cross-references, stale claims.
Write report to Resources/Wiki/lint-YYYY-MM-DD.md
Append summary to log.md.

## Output Rules
- Never modify files in Inbox/ — sources are immutable
- Never overwrite wiki pages — update in place, update the updated: frontmatter field
- Wikilinks: [[page-name]] for all cross-references
- Filenames: lowercase-hyphens.md
- Always update index.md and log.md on every run
```

---

## Step 3: Create the Queue File Format

The queue directory (`Inbox/wiki_queue/`) accepts `.md` or `.txt` files containing URLs. Any format works:

```
# Queue file: ai-articles.md
https://example.com/article-about-llms
[Great article on prompting](https://example.com/prompting)
https://www.youtube.com/watch?v=VIDEO_ID
```

One URL per line. Multiple URLs per file. Drop the file in the folder and Syncthing pushes it to the VPS.

---

## Step 4: Install yt-dlp (for YouTube support)

```bash
pip install yt-dlp
# or
sudo apt install yt-dlp
```

Test it:
```bash
yt-dlp --skip-download --write-auto-subs --sub-langs en --convert-subs srt \
  -o "/tmp/wiki-test-%(id)s.%(ext)s" "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
```

If a `.en.srt` file appears in `/tmp`, it's working.

---

## Step 5: Configure the Queue Auto-Trigger in the Dispatcher

Add this block to your `dispatcher.sh` (before cleanup):

```bash
WIKI_QUEUE="$HOME/vault/Inbox/wiki_queue"
WIKI_TASK="$AGENTS_DIR/wiki_agent/task-list.md"

if [ -f "$WIKI_TASK" ] && compgen -G "$WIKI_QUEUE"/*.md "$WIKI_QUEUE"/*.txt > /dev/null 2>&1; then
  if ! grep -q "^- \[ \] \[trigger\] process-queue\|^- \[ \] \[in-progress\].*process-queue" "$WIKI_TASK"; then
    sed -i "/^## Backlog/a - [ ] [trigger] process-queue" "$WIKI_TASK"
    echo "[wiki_agent] Files in queue. Injected process-queue trigger."
  fi
fi
```

Now dropping a file in `wiki_queue/` automatically triggers the agent within 5 minutes.

---

## Step 6: Create the Task List

`~/vault/Projects/Agents/wiki_agent/task-list.md`:

```markdown
# wiki_agent — Task List

> **Queue trigger:** Drop a .md or .txt file with a URL into Inbox/wiki_queue/
> Cron detects it and triggers process-queue within 5 minutes.
>
> **Manual ingest:** Add below with [trigger] and filename:
> `- [ ] [trigger] ingest: filename.md`
>
> **Lint run:** Add:
> `- [ ] [trigger] lint`

---

## Backlog

---

## In Progress

---

## Done
```

---

## Step 7: Set Up the Wiki Page Templates

Source page template (the agent generates these, but this is the structure):

```markdown
---
type: source
title: Article Title Here
tags: []
sources: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

## Summary
[2–4 sentences on the article's main argument]

## Key Concepts
- [[concepts/concept-name]] — how it appears here

## Key Entities
- [[entities/entity-name]] — how it's mentioned here

## Notable Claims
- [specific, quotable claims worth tracking]

## Source
[Original title and URL]
```

---

## Step 8: Test the Queue

Drop a test file:

```bash
echo "https://en.wikipedia.org/wiki/Knowledge_graph" > ~/vault/Inbox/wiki_queue/test.md
```

Wait 5 minutes (or run the dispatcher manually). Check:

1. `Inbox/wiki_queue/processed/test.md` — queue file should be moved here
2. `Resources/Wiki/sources/` — a new source page should exist
3. `Resources/Wiki/index.md` — updated with the new source
4. `Resources/Wiki/log.md` — entry appended

---

## Step 9: Schedule the Weekly Lint

Add to `wiki_agent/CLAUDE.md`:

```markdown
## Schedule
<!-- schedule: day=7, hour=10, task=Wiki lint -->
```

Day 7 = Sunday. Hour 10 = 10:00 UTC. The dispatcher reads this and injects a lint trigger every Sunday at 10am UTC.

---

## Using the Wiki in Obsidian

Once populated, the wiki works best with:
- **Graph view** — see how concepts and entities connect across sources
- **Backlinks panel** — when reading any page, see what else references it
- **Search** — find all pages mentioning a specific concept or entity

The compounding effect becomes visible around 10-15 sources in. Concept pages start reflecting multiple perspectives. Entity pages accumulate context from different articles. Cross-references create paths you didn't plan for.

---

## Troubleshooting

**Queue file not triggering:** Check that the dispatcher's queue check block is in the script and the file extension is `.md` or `.txt`.

**YouTube transcript empty:** Some videos have auto-subs disabled. The agent falls back to WebFetch on the video URL if yt-dlp fails.

**Index getting stale:** Run a manual lint: add `- [ ] [trigger] Wiki lint` to the task list. The lint run rebuilds the index from the actual directory contents.

---

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
