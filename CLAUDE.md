# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cosmik Debris is a personal blog built with **Jekyll 4.4.1** using the **Chirpy theme (~> 7.4.1)**, deployed to GitHub Pages at https://cosmikdebris.site. Ruby 3.2.3 with Bundler.

## Common Commands

```bash
# Install dependencies
bundle install

# Development server (live reload, host 0.0.0.0:4000)
./tools/run.sh

# Build and test (production build + html-proofer)
./tools/test.sh

# Direct Jekyll
bundle exec jekyll serve -l          # Dev server with live reload
bundle exec jekyll serve -l --drafts # Dev server with live reload + drafts
bundle exec jekyll build             # Production build
JEKYLL_ENV=production bundle exec jekyll build  # Full production build

# HTML validation
bundle exec htmlproofer _site --disable-external \
  --ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"
```

## Architecture

### Theme Customization Model

The site uses a **gem-based Chirpy theme** — theme files live in the gem (vendor/bundle), and local files override them:

- `_sass/themes/_dark.scss` / `_light.scss` — Cobalt2 color scheme overrides
- `_includes/head.html` — Custom Google Fonts injection (Exo 2, Inconsolata)
- `assets/css/jekyll-theme-chirpy.scss` — Main stylesheet entry point
- `assets/lib/` — Static assets via git submodule (chirpy-static-assets)

### Key Directories

- `_posts/` — Blog posts (Markdown with YAML front matter)
- `_tabs/` — Navigation pages (about, archives, categories, tags)
- `_data/` — Site data (contact.yml for social links, share.yml for sharing buttons)
- `_plugins/posts-lastmod-hook.rb` — Auto-sets `last_modified_at` from git history

### Post Front Matter Format

```yaml
---
title: "Post Title"
date: YYYY-MM-DD HH:MM:SS -0800
categories: [Category1, Category2]
tags: [tag1, tag2, tag3]
author: jason
pin: false
image:
  path: /assets/img/posts/image.jpg
  alt: Description
---
```

## Design System

**Cosmik Pulp dark theme** with these key colors:
- Background: `#1a1c1d` / Sidebar: `#141617`
- Text: `#d1c9bd` / Muted: `#8a857e`
- Orange/accent: `#d9573d` (sidebar active, TOC highlight, avatar border)
- Teal: `#5db8c8` (links, code, search)
- Mustard: `#d9ad46` (warnings, filepath, numbers)

**Fonts**: Exo 2 (headings), Inconsolata (body + code) — loaded via Google Fonts in `_includes/head.html`. Variables in `assets/css/jekyll-theme-chirpy.scss`: `$font-body`, `$font-heading`, `$font-code`.

## Deployment

GitHub Actions (`.github/workflows/pages-deploy.yml`) auto-deploys on push to main: checkout with full git history → Ruby 3.3 setup → Jekyll build → html-proofer → deploy to GitHub Pages.
