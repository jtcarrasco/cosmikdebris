---
title: "I Let an AI Help Me Build This Blog, and Now I Have Opinions About It"
date: 2026-02-08 10:00:00 -0800
categories: [Web Development, AI]
tags: [jekyll, ai, claude, web-dev, workflow, learning]
author: jason
pin: false
image:
  path: /assets/img/posts/building-a-jekyll-site.jpg
  alt: Robots & Cosmik Debris logo
---

## The site you're reading right now shouldn't exist yet.

Not because it's bad (I hope it isn't) but because I've started approximately seven personal blogs in my life and abandoned all of them somewhere between "configure the theme" and "write the first real post." The setup always ate the momentum.

This time I tried something different: I built it in collaboration with [Claude Code](https://claude.ai/code). Not just "ask AI to fix a bug" but the whole thing: architecture decisions, documentation structure, the color scheme, the workflow I'll actually use to keep writing. What I wanted to figure out was how to actually use AI as a collaborator, not as a shortcut, but as a thinking partner integrated into a real workflow. This site was the test case.

Reader, I published this post. So something worked.

## The setup I actually wanted

The goal was clear enough: personal blog, [Jekyll](https://jekyllrb.com/) with the [Chirpy theme](https://github.com/cotes2046/jekyll-theme-chirpy), hosted on [GitHub Pages](https://pages.github.com/), my "Cosmik Pulp" color scheme. I've built hundreds of WordPress sites for clients. Static site generators are different: less infrastructure to maintain, more configuration to understand. I wanted to understand it, not just copy-paste until it worked.

The traditional path: dive into Chirpy's docs, configure the site, figure out the weird parts as you hit them, write a post someday. Maybe document things later. (Narrator: he did not document things later.)

Instead I flipped it. Start with the documentation system, build the workflow first, then let the content follow.

## Why document anything before you have content?

Because future me is a different person with no memory of why past me made any particular decision. This is not a metaphor. I genuinely cannot tell you why I made half the configuration choices in my last three projects without digging through git history like an archaeologist.

Working with Claude to build documentation files *while* making the decisions meant I captured the reasoning at the moment I had it. We created five files before I wrote a single post:

**claude.md**: Project overview, technical stack, common tasks. The briefing document for every future session where I say "remind me how this works."

**workflow.md**: Git conventions, branch naming, pre-publish checklist. Already saved me from forgetting steps twice.

**content.md**: Post backlog, ideas, writing guidelines. The holding tank where half-formed thoughts live until they become real posts.

**tags.md**: Taxonomy. Sounds tedious; turns out consistent tagging from the start is much easier than fixing it after 50 posts.

**design.md**: The Cosmik Pulp color scheme documentation, font choices, override file locations. So when I want to tweak something in six months, I know which file to touch.

This might seem like overkill for a blog that had zero posts. But here's the thing: I actually did it, because Claude helped me do it immediately rather than later. And "actually did it" beats "planned to do it" every time.

## What the collaboration actually felt like

Not what I expected. I expected to feel like I was delegating work to a very fast intern. It was more like thinking out loud with someone who retained everything and had read the Chirpy documentation more carefully than I was going to.

The pattern that emerged:

**Me:** "I want to customize the colors with the Cosmik Pulp scheme."

**Claude:** "Here's how Chirpy's sass override structure works, these are the files you'll need, here's the CSS. Should we document this in design.md?"

**Me:** "Yes, and add a checklist for testing the changes so I don't break the dark theme."

And then it existed. Implementation plan, code snippets, testing checklist, all in one place. The friction cost of doing it right was lower than the friction cost of skipping it.

What actually worked:

**Structured thinking.** Having to explain what I wanted made me clarify it to myself. Vague intentions became concrete requirements. "I want it to feel technical but not cold" became actual color values and font pairings.

**Documentation discipline.** The barrier to writing it down dropped to nearly zero when I could think through it out loud and have it captured automatically. I am not naturally a documenter. I am now, accidentally.

**Pattern consistency.** Once we established conventions (front matter format, tag structure, branch naming) they stuck. Claude picked them up from context and maintained them without reminders. This is a small thing that matters a lot over time.

**Actual learning.** Because Claude explained the *why* behind Jekyll's structure and Chirpy's override system, I understood the thing I'd built. When something breaks later, I'll have a model for debugging it instead of just poking at files until it works.

What was harder than expected:

**Context across sessions.** In longer conversations, I sometimes had to re-establish decisions from earlier. The documentation files helped with this: we'd reference what we'd written instead of reconstructing it from memory. But it's still friction. I'm getting better at front-loading context at the start of each session.

**Knowing what to ask.** The output quality tracked directly with how clearly I articulated the goal. "Make me a blog" produces something generic. "A blog with this aesthetic, this audience, this workflow constraint" produces something useful. This got easier with practice, but there's a skill to it.

**Staying skeptical.** AI can be confidently wrong. Jekyll and Chirpy have specific opinions about file locations and override behavior, and not every suggestion was right on the first pass. I needed to verify against the actual Chirpy docs when something didn't behave as described. Trust but verify is not optional.

## The part that surprised me most

After a few hours of this, I had a site that worked the way I wanted, documentation I could actually use, and a workflow I understood end to end. More importantly, I had enough momentum to keep going.

That last part is the thing I'd underestimated. Previous blogs died during setup because setup is boring and momentum is fragile. Having a collaborator who never got bored of the infrastructure details kept the energy on the parts that mattered (what I actually wanted to build) instead of burning it on configuration minutiae.

It's not magic and it's not autopilot. It's closer to pair programming with someone who has infinite patience for tedious setup questions but needs you to remain the person who knows what you're actually trying to make.

## If you want to try this

A few things I'd do the same way:

**Start with the docs.** Before writing any content or pushing any code, spend 30 minutes on the documentation structure. What decisions will you make? Where will you capture them? Do this while the project is small enough that it takes 30 minutes instead of three hours.

**Be specific.** "Jekyll blog with Chirpy theme, Cosmik Pulp color scheme, audience is homelab builders and makers, posts about Linux and maker projects." That's useful. "Personal blog" is not useful.

**Iterate openly.** "That's close but wrong, let's adjust X" works better than accepting the first version and working around it. Say what's off. The second or third attempt is usually right.

**Understand what you're building.** Don't just accept generated code. Ask why. Make sure the output matches your mental model of the tool. This protects you when things break later, and things will break later.

---

The foundation is in place. Now comes the part I actually wanted to get to: writing about the projects I'm working on.

Which, given that you're reading this, seems to be going okay.

— Jason
