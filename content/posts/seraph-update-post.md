---
title: "Infrastructure updates, a new machine, and the usual suffering"
date: 2026-04-09
tags: ["homelab", "arch", "linux", "rust", "self-hosting", "devlog"]
draft: false
---

It's been a while since I've written anything up. A lot has happened. Let me try to cover it without this turning into a novel.

## Vaultwarden

I've been meaning to self-host a password manager for a long time. Vaultwarden is a lightweight, Bitwarden-compatible server — same clients, same browser extensions, your data stays on your own hardware. It's running in Docker at vault.seraph.ws, behind nginx with a Let's Encrypt cert. Signups are disabled after account creation. fail2ban watches the log for bad auth attempts. Honestly the easiest thing I've set up in months, which I'm choosing to read as a sign that I'm getting better at this.

## Uptime Kuma

Also running now at status.seraph.ws. It watches everything — the site, mail, vault, the link shortener, QR generator, pastebin, Plausible, IMAP, SMTP, Postgres, itself — and sends an email alert if something goes down.

The Postgres monitor took a minute to figure out. Kuma runs in Docker, and Postgres was only listening on localhost, which Docker containers can't reach. Fixed it by adding the Docker bridge host IP to `listen_addresses` in `postgresql.conf` and opening the firewall for the Docker subnet. Then it worked.

For catastrophic failures where the whole server goes dark, I added UptimeRobot as an external watchdog. It monitors from outside my infrastructure and alerts to my Proton inbox directly, bypassing the mail server entirely. Chicken-and-egg problem, solved.

## seraph-watch

This one was genuinely fun to build. seraph-watch is a terminal security dashboard for the server — Ratatui-based, written in Rust, very much in the same spirit as the seraph-mail scaffold from a while back.

It has five tabs. **Overview** gives you a top-level view of the server: SSH failures, banned IPs, container health (N/N healthy), disk usage, memory, and load average — all as stat boxes with color-coded thresholds. Below that is a live container status table. **SSH Auth** is a scrollable event log parsed from `/var/log/auth.log`, color-coded by event type (accepted, failed password, invalid user), with a "top offenders" panel alongside it that ranks IPs by failure count and cross-references them against the CrowdSec ban list. **CrowdSec** shows the raw output of `cscli decisions list` and `cscli alerts list` in split panels. **Docker** lets you navigate between containers and tail their logs in real time — scroll down to select a container, the log pane updates to show the most recent 60 lines, with error/warn/info levels highlighted. **Mail** is the same idea but scoped to the mailserver container specifically.

The whole thing refreshes on demand with `r`, keeps a persistent status bar with key counts at a glance, and uses the same palette as everything else on this server — `#ff2d6d` accent on near-black.

It lives on seraph and I use it more than I expected to.

## Heist Tactics

The side project that started as "I want to learn some game dev concepts" and has quietly become a real thing. It's a turn-based tactics game — four-agent squad, procedural maps, network topology stored as node graphs. Since the last update I've implemented:

- **Map generation** — seeded procedural floor plans with rooms and furniture placement (you can see one at the top of this post)
- **Cover system** — low and high cover with material-based thresholds, tracked per tile edge rather than per tile, which is the correct way to do it and makes everything else less painful
- **Enemy AI** — basic but functional; guards patrol, take cover, and react to player positions
- **Wall storage** — walls between tiles rather than on them

It's playable. It's not done. I'll write it up properly when there's more to show.

## Empyrean

I built a new desktop. I'd been running on aging hardware long enough that the "just reinstall and it'll be fine" approach had stopped working, so I assembled something new out of a mix of parts I had and parts I ordered: ASUS ROG STRIX X470-F, GTX 1660 Ti, TP-Link UB500 for Bluetooth. Named it `empyrean`, continuing the naming scheme. The build itself went fine.

Installing Arch on it did not go fine, and I say that as someone who has done this before and was not naive about what I was walking into.

The short version: NVIDIA + Wayland is still annoying in the specific way that NVIDIA + Wayland has always been annoying. Getting DKMS configured correctly, pinning the right driver version, making sure modules rebuild on kernel updates — there was a real stretch where I wasn't sure if the system was stable or just hadn't crashed *yet*.

It was the latter.

A non-exhaustive list of things I debugged:

- DKMS not rebuilding the NVIDIA module after a kernel update, silently, resulting in a black screen on next boot
- KDE Plasma on Wayland being fine until it wasn't, with the "until it wasn't" taking several days to surface
- Bluetooth requiring the UB500 adapter, the TP-Link driver, and `btusb` all cooperating simultaneously, which they eventually did
- An absolutely unholy rabbit hole trying to debug sleep and suspend that turned out to be, and I shit you not, the result of another debugging rabbit hole trying to fix NVIDIA rendering. Masked hybernate and suspend and it bit me in the ass a week later. 

It's stable now. Or at least, it's been stable long enough that I've stopped waiting for the next thing to break. KDE Plasma with the usual setup — Graphite Dark, `#ff2d6d`, JetBrainsMono Nerd Font, Monochrome Icons, Fish with Starship — is running on Wayland and looking exactly right. The machine is fast, quiet, and runs 15 year old games without chugging, which is what matters.

---

More things are in progress. More things will break. More posts to follow.
