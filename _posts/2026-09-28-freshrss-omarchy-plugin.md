---
title: "FreshRSS in the Omarchy Bar, With FreshRSS's Own Keyboard Shortcuts"
date: 2026-09-28 10:00:00 -0700
categories: [Linux, Plugins]
tags: [omarchy, freshrss, rss, linux, self-hosted, plugin]
author: jason
pin: false
image:
  path: /assets/img/posts/freshrss-omarchy-plugin.webp
  alt: "FreshRSS for Omarchy: the feed reader popout in the Omarchy bar"
---

I read my feeds in [FreshRSS](https://freshrss.org/), and I'd trained my hands on its keyboard shortcuts: `j` for the next article, `r` to mark it read, `f` to star it. Then I started testing [Omarchy](https://omarchy.org/) on my desktop (my daily driver is [DankMaterialShell](https://danklinux.com/) from Dank Linux). Omarchy is a desktop where the whole point is never reaching for the mouse, and I discovered that checking my feeds meant opening a browser, finding the tab, and waiting for the web app to load. The Omarchy plugin marketplace had ten RSS readers. None of them could talk to my FreshRSS server.

So I wrote one that does. **FreshRSS for Omarchy** is now a verified plugin on the [Omarchy plugin marketplace](https://omarchyplugins.com/plugin.html?id=freshrss-reader).


## What is FreshRSS?

[FreshRSS](https://freshrss.org/) is a free, open-source RSS reader that you host yourself. It runs on your own server, checks your feeds, and keeps one shared list of what you've read and starred. The web app, the phone apps and this plugin all read from it, so marking something read in one place marks it read everywhere. You can try it on the [public demo](https://demo.freshrss.org/) before installing anything.

If you want your own, the official [Docker image](https://hub.docker.com/r/freshrss/freshrss) with its [Docker guide](https://github.com/FreshRSS/FreshRSS/tree/edge/Docker) is the quickest route. The [installation guide](https://freshrss.github.io/FreshRSS/en/admins/03_Installation.html) covers a regular PHP setup.

## What the plugin does

The RSS icon in the bar shows your unread count, refreshed every five minutes. Click it and you get:

- **All unread, Starred, and every category** with its own unread count.
- **An article list with thumbnails**, pulled from the feed's image or the first image in the article, with the site's icon when there's no picture.
- **An article view** with the lead image, author, date and summary, plus buttons to open it in your browser, star it, or mark it unread.
- **Mark all read** for a category, which asks you to click twice so you don't wipe out a week of reading by accident.
- **A pop-out window** for longer reading sessions.

Everything syncs with your server through FreshRSS's Google Reader API, the same one the phone apps use.

## The keys are FreshRSS's keys

This was the part I cared about most. The plugin uses FreshRSS's own default shortcuts, so muscle memory from the web app carries straight over:

| Key | Action |
|---|---|
| `j` / `k` | Next / previous article |
| `h` | Next unread article |
| `n` / `p` | Next / previous category |
| Enter | Open the article |
| Space | Open it in your browser |
| `r` | Toggle read |
| `f` | Toggle star |
| `m` | Load more |
| `u` | Unread only / all items |
| `q` | Refresh |

A few extras exist because Omarchy needs them: `z` switches between the dropdown and its own window, and `,` opens settings. The full list is on the settings page.

## Install it

You need Omarchy and a FreshRSS server with API access turned on.

```
omarchy plugin add https://github.com/jtcarrasco/freshrss-reader --enable
```

Before you connect, do this one step in FreshRSS, because it trips up everyone:

1. In FreshRSS, open **Settings → Authentication** and enable **Allow API access**.
2. Open **Settings → Profile** and set an **API password**.

The plugin logs in with that API password, not your normal web password. FreshRSS's [mobile access guide](https://freshrss.github.io/FreshRSS/en/users/06_Mobile_access.html) explains the same setup. Then click the RSS icon, enter your server address, username and API password, and your categories appear. The login token goes into your system keyring.

**Using DankMaterialShell?** The repo includes a [DMS](https://danklinux.com/) version in `dms/` with the same features and keys. The [README](https://github.com/jtcarrasco/freshrss-reader#dankmaterialshell) has the install steps.

## One FreshRSS quirk worth knowing

If a category shows unread items but opens empty, check its name for an `&`. FreshRSS can store a category name containing `&` in a form its own API can't look up, which happens a lot after an OPML import. Rename the category or save it again in FreshRSS and it works. Every app that talks to FreshRSS's API hits this, not only this one; I found it in my own feeds while testing.

The code is [on GitHub](https://github.com/jtcarrasco/freshrss-reader), MIT licensed. My feeds are one keystroke away again, and the browser tab has been retired.

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
