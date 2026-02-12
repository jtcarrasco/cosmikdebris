---
title: "Building a Jekyll Site with AI: My First Experiment"
date: 2026-02-08 10:00:00 -0800
categories: [Web Development, AI]
tags: [jekyll, ai, claude, web-dev, workflow, learning]
author: jason
pin: false
image:
  path: /assets/img/posts/robots.jpg
  alt: Robots & Cosmik Debris logo
---

## A Different Kind of Build Process

This site you're reading right now represents a first for me: building a website by collaborating with AI. Not just using AI to generate a snippet here or fix a bug there, but actively working with Claude to architect the documentation structure, plan the workflow, and establish the foundation of this Jekyll blog.

I'll be honest—I wasn't sure what to expect. Would it feel like I was just delegating work? Would I learn anything? Would the result feel authentic to what I wanted? Spoiler: it turned out to be more collaborative and educational than I anticipated.

## Why This Approach?

I've built hundreds of websites before, but this time I wanted to try something different. I had a clear vision for what I wanted—a personal blog using Jekyll and the Chirpy theme, hosted on GitHub Pages—but I was curious whether working with AI could streamline the setup process and help me establish good documentation practices from the start.

The traditional approach would have been: dive into the Chirpy documentation, set up the site, start writing posts, and maybe (probably not) create some documentation files later. Instead, I flipped it: start with the documentation structure, work with AI to organize my thinking, and build a system that would make future work easier.

## The Documentation-First Approach

One of the first things Claude and I worked on was creating a documentation system. Rather than just having a Jekyll site with posts, I wanted a separate documentation folder that would serve as my "mission control" for the project.

We created several key files:

**claude.md** - The main reference document with project overview, technical stack, and common tasks. Think of it as the README for how I work with Claude on this project.

**workflow.md** - Git workflow, branch naming conventions, and pre-publish checklists. This one has already saved me from forgetting steps.

**content.md** - Blog post planning, drafts, content calendar, and writing guidelines. My ideas backlog lives here.

**tags.md** - Tag taxonomy and management. Sounds boring, but having consistent tagging from the start matters.

**design.md** - Design customization specs, including my Cobalt2 color scheme choices and typography system.

This might seem like overkill for a personal blog, but here's what I learned: having these files means I don't have to remember everything. Future me (and Claude, in future sessions) can reference these documents to understand decisions, maintain consistency, and pick up where we left off.

## The Collaboration Workflow

Working with AI on this project wasn't about just asking for code and copying it. It was more of a conversation. Here's how it typically went:

**Me:** "I want to use the Chirpy theme and customize it with Cobalt2 colors."

**Claude:** "Here's how Chirpy's structure works, here are the files you'll need to modify, and here's the CSS for Cobalt2 colors. Should we create a design.md file to document this?"

**Me:** "Yes, and let's add a checklist for testing the changes."

**Claude:** *Creates design.md with implementation plan, code snippets, and testing checklist*

The AI wasn't just generating content—it was helping me think through the structure, asking good questions, and suggesting best practices I might not have considered. When I wanted to add something, Claude would often suggest how it fit into the existing documentation structure.

## What Worked Really Well

**Structured thinking**: Having to explain what I wanted to an AI made me clarify my own thinking. Vague ideas became concrete requirements.

**Documentation discipline**: Normally I'd skip documentation until later (read: never). Having Claude help create it made the barrier to entry lower.

**Pattern recognition**: Claude picked up on the conventions I was establishing and maintained consistency across files. Branch naming, tag structure, front matter format—once we established a pattern, it stuck.

**Learning opportunity**: Because Claude explained why certain approaches made sense for Jekyll, GitHub Pages, and the Chirpy theme, I learned the reasoning behind the structure, not just the structure itself.

## What Was Challenging

**Context retention**: In longer sessions, I sometimes had to remind Claude of decisions we'd made earlier. This is where having documentation files helped—we could reference what we'd written.

**Knowing what to ask**: The quality of the output depended heavily on how well I articulated what I wanted. This got easier with practice.

**Trust but verify**: I still needed to understand what was being created. AI can be confidently wrong, so I reviewed everything and made sure it matched my understanding of Jekyll and the Chirpy theme.

## The Markdown Structure

The heart of this system is simple: organized markdown files for different concerns. Each file serves a specific purpose:

- Reference files (claude.md) for "what is this project"
- Workflow files (workflow.md) for "how do I do things"
- Planning files (content.md) for "what am I working on"
- Specification files (design.md) for "how should this look/work"

This separation means I can find information quickly, and Claude can reference the relevant file for context. It's not revolutionary—it's just good organization—but having AI help set it up meant I actually did it instead of procrastinating.

## Lessons for Others

If you're thinking about using AI to help build a Jekyll site (or any static site), here's what I'd suggest:

**Start with structure**: Before generating content or code, establish your documentation system. It pays dividends immediately.

**Be specific**: "Make me a blog" gets you generic results. "Create a Jekyll blog with Chirpy theme, Cobalt2 colors, focused on maker projects and Linux" gets you something useful.

**Iterate openly**: If something isn't right, say so. AI responds well to "that's close, but let's adjust X" or "I like option 2 better than option 1."

**Document decisions**: As you make choices, capture them. Future you will thank present you.

**Understand the output**: Don't just copy-paste. Make sure you understand why the AI suggested what it did. Ask questions.

## What's Next

Now that the foundation is in place, I'm excited to fill this site with actual content about the projects I'm working on. The documentation system is there to support that work, not get in the way of it.

And who knows? Maybe in a few months I'll write a follow-up about what I learned after actually using this system for a while. For now, I'm calling this experiment a success.

If you're curious about using AI in your own projects, I'd encourage you to try it. Start small, be clear about what you want, and don't be afraid to iterate. You might be surprised at what you can build together.

— Jason
