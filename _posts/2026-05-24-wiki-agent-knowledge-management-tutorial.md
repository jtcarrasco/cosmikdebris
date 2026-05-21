---
title: "I Built an AI That Reads the Internet So I Don't Have To (And Now I Actually Remember Things)"
date: 2026-05-24 09:00:00 -0800
categories: [AI, Tutorial]
tags: [ai, claude, knowledge-management, obsidian, tutorial]
author: jason
pin: false
image:
  path: /assets/img/posts/wiki-agent-knowledge-management.jpg
  alt: AI wiki agent for knowledge management with Obsidian
---

Here's my workflow before this project: read an interesting article, think "I should remember this," immediately forget everything in it, open a new tab, repeat until my browser looks like a hoarder's garage. I had bookmarks folders with names like "Read Later" that I hadn't opened since 2019. Good stuff in there, probably. Who knows.

The problem isn't finding content. The firehose is wide open. The problem is that reading something and retaining it are two completely different activities, and I was only doing one of them.

So I built an agent that does the retention part for me.

**What you'll end up with:** Drop a URL in a folder from your phone. Within 5 minutes, your vault has a summary, extracted concepts, entity pages, and cross-references to everything else you've ever queued. The wiki gets smarter every time you add something. You get to feel like a genius without doing the work.

**Prerequisites:** [Claude Code CLI](https://claude.ai/code), [Obsidian](https://obsidian.md/) vault, [Syncthing](https://syncthing.net/) (or similar sync), [`yt-dlp`](https://github.com/yt-dlp/yt-dlp) for YouTube support.

---

## Step 1: Set Up the Directory Structure

First, we need some folders. Thrilling, I know, but the structure matters because the agent uses it to find things.

```bash
# Agent folder
mkdir -p ~/vault/Projects/Agents/wiki_agent/{knowledge,logs}

# Wiki in your vault
mkdir -p ~/vault/Resources/Wiki/{sources,concepts,entities}
mkdir -p ~/vault/Inbox/wiki_queue/processed
```

Three types of wiki pages live here:
- **sources/**: one page per article or video. The "I read this" receipts.
- **concepts/**: ideas that keep showing up across multiple sources. When three different articles all mention "context windows," that becomes a concept page.
- **entities/**: specific things: tools, people, companies. The nouns.

Create the index and log files that the agent will keep updated:

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

This is the agent's operating manual. Think of it as the job description you wish you could give every person who's ever tried to "help" you organize something.

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

The key lines are the output rules at the bottom. "Never overwrite wiki pages: update in place" is the difference between a wiki that accumulates knowledge and one that rewrites itself into amnesia every time it runs.

---

## Step 3: Create the Queue File Format

The queue is beautifully low-tech. It's just a folder. Drop a `.md` or `.txt` file in it with one or more URLs and the agent handles the rest.

```
# Queue file: ai-articles.md
https://example.com/article-about-llms
[Great article on prompting](https://example.com/prompting)
https://www.youtube.com/watch?v=VIDEO_ID
```

Bare URLs, markdown links, mixed: all fine. One URL per line, as many URLs per file as you want. When Syncthing pushes it to your VPS, the cron job picks it up within 5 minutes and it's gone from the queue, processed, moved to `processed/`, and living in your wiki.

No app. No browser extension. No "save for later" button that leads to a digital purgatory. Just a text file.

---

## Step 4: Install yt-dlp (for YouTube Support)

Because apparently a lot of good information is locked inside people talking at cameras for 45 minutes. The agent uses `yt-dlp` to pull the auto-generated transcript and treats it like an article.

```bash
pip install yt-dlp
# or
sudo apt install yt-dlp
```

Test it with a classic:

```bash
yt-dlp --skip-download --write-auto-subs --sub-langs en --convert-subs srt \
  -o "/tmp/wiki-test-%(id)s.%(ext)s" "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
```

If a `.en.srt` file appears in `/tmp`, it's working. (The test URL is Rick Astley, which means your knowledge base will now contain a page about `Never Gonna Give You Up`, the best possible first entry.)

**If you use the [Obsidian Web Clipper](https://obsidian.md/clipper):** Clipping a YouTube page from your browser embeds the transcript directly into the saved markdown file. When Syncthing pushes that file to your VPS, the agent already has the transcript: no yt-dlp required for those clips. This is the easiest path if you're already clipping from a browser: the clipper does the work locally, the file arrives on the VPS ready to process. yt-dlp handles the cases where you're queueing bare URLs without going through a browser first.

---

## Step 5: Configure the Queue Auto-Trigger in the Dispatcher

Add this to your `dispatcher.sh` before the cleanup section. It checks whether anything's sitting in the queue folder and, if so, injects a trigger so the agent picks it up on the next cron run:

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

The guard clause prevents double-triggering if you dump three files in at once. The agent runs once and processes all of them.

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

This file is how you communicate with the agent. It's a markdown file with checkboxes. The cron job reads it. There is something deeply satisfying about triggering an automated AI system by editing a text file on your phone.

---

## Step 7: Set Up the Wiki Page Templates

The agent generates these pages automatically, but here's what a source page looks like so you know what you're getting:

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

The wikilinks (`[[concepts/concept-name]]`) are the magic. When the agent processes a new article and finds a concept that already has a page, it links to it. When it finds a new concept, it creates the page and back-links it to the source. Over time the graph gets dense in ways you didn't plan.

---

## Step 8: Test the Queue

Let's make sure this thing actually works before you trust it with anything important.

```bash
echo "https://en.wikipedia.org/wiki/Knowledge_graph" > ~/vault/Inbox/wiki_queue/test.md
```

Wait 5 minutes (or run the dispatcher manually). You should see:

1. `Inbox/wiki_queue/processed/test.md` — moved here, out of the queue
2. `Resources/Wiki/sources/knowledge-graph.md` — new source page
3. `Resources/Wiki/index.md` — updated with the entry
4. `Resources/Wiki/log.md` — one line added

If all four are true, you have a working wiki agent. Everything after this is just feeding it URLs.

---

## Step 9: Schedule the Weekly Lint

The lint task is the agent auditing its own work. Every Sunday it checks for orphaned pages, missing cross-references, index gaps, and concept pages that only have one source (meaning they're not really concepts yet, just terms). It writes a report and leaves it in your wiki.

Add to `wiki_agent/CLAUDE.md`:

```markdown
## Schedule
<!-- schedule: day=7, hour=10, task=Wiki lint -->
```

Day 7 = Sunday. Hour 10 = 10am UTC. The dispatcher reads the schedule comment and injects a lint trigger automatically. You don't have to remember to run it, which is the whole point of all of this.

---

## What It Looks Like After a Few Weeks

The compounding effect kicks in around 10-15 sources. Before that it's mostly standalone pages. After that, the cross-references start doing work.

A concept page like `[[concepts/retrieval-augmented-generation]]` might have 8 sources linked to it from different angles: an intro explainer, a paper, a practical implementation guide, a critical take. The entity page for `[[entities/anthropic]]` has notes from 12 different articles, each adding a different facet.

Open the [Obsidian graph view](https://help.obsidian.md/Plugins/Graph+view) around that time. You'll see clusters forming that you didn't design. That's the compounding effect: the wiki knows things you didn't explicitly teach it, because patterns emerged from the sources.

It's a weird and good feeling.

**Three Obsidian features that pair well with a populated wiki:**
- **Graph view**: the "oh wow" moment when you see the connections
- **Backlinks panel**: when reading any page, see what else links to it
- **Search**: `is:linked` to find orphaned pages; concept names to see how they spread

---

## Troubleshooting

**Queue file isn't triggering:** Check that your dispatcher has the queue-check block and that the file extension is `.md` or `.txt`. The dispatcher reads for those extensions specifically.

**YouTube transcript is empty:** Some videos disable auto-subs. The agent falls back to WebFetch on the video URL, which at least gets the description and any transcript in the page itself.

**Index is stale:** Run a manual lint. Add `- [ ] [trigger] Wiki lint` to the task list. The lint rebuilds the index from actual directory contents, which fixes any gaps from interrupted runs.

**The Rick Astley page is actually in your wiki:** You did the test correctly. Congratulations.

---

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
