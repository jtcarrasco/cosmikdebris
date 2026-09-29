---
title: "My Audiobooks Now Live in My Status Bar (Audiobookshelf for Omarchy)"
date: 2026-09-28 09:00:00 -0700
categories: [Linux, Plugins]
tags: [omarchy, audiobookshelf, linux, self-hosted, plugin]
author: jason
pin: false
image:
  path: /assets/img/posts/audiobookshelf-omarchy-plugin.webp
  alt: "Audiobookshelf for Omarchy: the player popout in the Omarchy bar"
---

My audiobooks and podcasts live on a self-hosted [Audiobookshelf](https://www.audiobookshelf.org/) server. Listening to them on my phone is great. Listening to them at my desk meant keeping a browser tab open, finding it among forty other tabs every time I wanted to pause, and losing my place whenever I closed the wrong window. On a desktop where everything else is one keystroke away, that tab started to feel like a personal insult.

So I built a player that lives in the bar instead. It's called **Audiobookshelf for Omarchy**, and as of this week it's a verified plugin on the [Omarchy plugin marketplace](https://omarchyplugins.com/plugin.html?id=abs-player).


## What is Audiobookshelf?

[Audiobookshelf](https://www.audiobookshelf.org/) is a free, open-source server for your audiobooks and podcasts. You run it yourself (a Docker container on a home server or a VPS is the usual route), point it at your library, and it keeps track of what you've listened to and where you stopped, across every device. If you've ever wished Audible synced with your own files instead of theirs, that's the idea. The [install docs](https://www.audiobookshelf.org/docs#install) get a server running in a few minutes.

This plugin is a client for that server. It needs one to talk to.

## What the plugin does

Click the headphones icon in the bar and you get a dropdown styled by your Omarchy theme:

- **A real player.** Cover art, a seek bar, 30-second skips, speed control from 0.8x to 2x, chapters, and podcast show notes.
- **Your whole library.** Books and podcasts with covers, search across both, per-show unplayed counts, and progress bars on anything you've started.
- **Progress that syncs both ways.** Pause at your desk, pick it up on your phone at the exact same spot. The plugin reports your position back to the server as you listen.
- **Right-click to mark finished**, the same way the web app does it.
- **A pop-out window** when the dropdown feels too small, and back again with one key.

Playback runs through [mpv](https://mpv.io/), so audio quality and format support are whatever mpv gives you, which is everything.

## It's built for the keyboard

[Omarchy](https://omarchy.org/) is a keyboard-first desktop, so every action in the plugin has a key. The ones you'll use most:

| Key | Action |
|---|---|
| `j` / `k` | Move through the list |
| Enter | Play the selected book or episode |
| Space | Play / pause |
| `h` / `l` | Back / forward 30 seconds |
| `n` / `p` | Next / previous chapter |
| `[` / `]` | Slower / faster |
| `1` / `2` / `3` | Home / Books / Podcasts |
| `/` | Search |
| `z` | Pop out into its own window |

The full list lives at the bottom of the plugin's settings page, and every button shows its key in the tooltip.

## Install it

You need Omarchy, an Audiobookshelf server you can log into, and mpv (the setup screen checks for it).

```
omarchy plugin add https://github.com/jtcarrasco/audiobookshelf-player --enable
```

Click the headphones icon, enter your server address, username and password, and your library shows up. The password goes to your server once to get a login token, which is stored in your system keyring. Nothing sensitive gets written to a config file.

**On DankMaterialShell?** The repo also carries a [DMS](https://danklinux.com/) version in the `dms/` folder with the same features and keys. The [README](https://github.com/jtcarrasco/audiobookshelf-player#dankmaterialshell) has the install steps.

## It passed a real review

Marketplace plugins get a human code review before they're listed, and this one went four rounds. The reviewer read the backend closely: server text that could smuggle in HTML, a login token that could leak through the desktop's media controls, responses with no size limit, and a server that dribbles bytes to keep a connection open. Each fix came with tests. The plugin you install is better for it, and so is every plugin I write after it.

The code is [on GitHub](https://github.com/jtcarrasco/audiobookshelf-player), MIT licensed. The browser tab is closed. I don't miss it.

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
