---
title: "I Turned My Job Search Agent Into a Kit (So You Can Skip the Part Where I Broke It)"
date: 2026-09-29 11:00:00 -0700
categories: [AI, Tools]
tags: [ai, claude, claude-code, automation, job-search]
author: jason
pin: false
image:
  path: /assets/img/posts/job-search-agent-kit.webp
  alt: "Job Search Agent Kit: two AI agents for Claude Code"
---

Back in May I wrote up [how I built an AI agent to do my job searching](/posts/job-agent-ai-job-search-tutorial/). It worked. It still works. It has run most weeks since spring, and in that time it has found me postings I would never have dug up by hand, and it has also taught me every way a small agent can quietly fail on a Tuesday morning.

The obvious follow-up to that post isn't "how does it work." It's some version of "can I just have yours?" Until now the honest answer was no, because mine was welded to my setup: a VPS, a cron dispatcher, an Obsidian vault, my resume, my salary floor, my very specific grudges against certain staffing agencies.

So I pulled it apart and rebuilt it as something that runs on anyone's machine. It's called the [Job Search Agent Kit](https://cosmikdebris.gumroad.com/l/dzehiz), and it's two agents instead of one.

## What's in it

**The Job Search Agent** is the one from the original post, minus my infrastructure. You fill in a criteria file (target skills, salary floor, industries, dealbreakers) and it searches LinkedIn, Indeed, ZipRecruiter, Glassdoor, [Adzuna](https://developer.adzuna.com/) and [We Work Remotely](https://weworkremotely.com/) in one pass. LinkedIn, Indeed, ZipRecruiter and Glassdoor all come through a single API, [JSearch](https://www.openwebninja.com/api/jsearch), which is how you get four job boards without scraping any of them. It filters everything against your criteria, remembers what it has already shown you so reposts don't come back, and writes a dated markdown report you can read in two minutes. There are separate profiles for full-time, freelance and AI-focused searches.

**The Job Application Agent** is the one I never wrote about. You paste in a job posting and it produces a folder with:

- a tailored resume
- a plain-text ATS version, because applicant tracking systems still choke on anything fancier than a text file
- a cover letter written for that specific role
- a follow-up email template for later

It works from a resume template you fill in once, with everything you've ever done in as much detail as you can stand to write. For each posting it picks, reorders and reframes what's true about you to match what that role is asking for.

## The rule I care about most

The application agent never invents experience. That sounds obvious until you watch a language model helpfully "improve" a resume by adding a technology you have never touched, because the posting mentioned it three times.

So the resume template has a section at the bottom called "Notes for the agent." Anything you put there, the agent checks on every run. Mine has a note that one of my former employers ran its own in-house platform, not WordPress, because so many postings ask for WordPress that early drafts kept crediting me with WordPress work at a company that never used it. If a draft ever claims something that isn't true, you add one line there and it doesn't happen again.

It's still a drafting tool, not an autopilot. It won't submit anything for you, and you should read every resume before it goes out. What it changes is the ratio: tailoring goes from most of an hour to a few minutes of reviewing.

## What you need

- [Claude Code](https://claude.com/claude-code) and a Claude subscription or API access
- Free API keys for [JSearch](https://www.openwebninja.com/api/jsearch) and [Adzuna](https://developer.adzuna.com/) (the setup guide walks you through both)
- About 15 minutes to fill in your resume and search criteria

You don't need a VPS, a cron job, or Obsidian. Both agents are plain folders you unzip anywhere on Mac, Windows or Linux, and you run them by opening a terminal in the folder and starting `claude`. If you do want the search to run on a schedule, the guide covers that too, but most people run it manually a couple of times a week while they're actively looking.

## Build it or buy it

The original tutorial is still free and still accurate, and if you like building things (which, if you read this site, you probably do), go build it. That post walks through the search agent end to end.

The kit is for when you'd rather spend your evening applying to jobs than debugging an agent: both agents, genericized, with a setup guide written for people who have never edited a `CLAUDE.md` file, plus the application agent that isn't in any post.

It's [$79 on Gumroad](https://cosmikdebris.gumroad.com/l/dzehiz). My own job search runs on the same two agents every week, which is either a strong endorsement or a sign that I should be applying to more jobs instead of writing blog posts about applying to jobs.

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
