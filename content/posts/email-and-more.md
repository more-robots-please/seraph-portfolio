+++
title = "Building a Self-Hosted Mail Stack (And Everything Else)"
date = "2026-03-11T00:58:44Z"
author = ""
tags = ["self-hosting", "email", "docker", "security", "linux", "sysadmin", "rust", "react"]
keywords = ["self-hosted email", "docker-mailserver", "vaultwarden", "uptime kuma", "fail2ban", "abuseipdb"]
description = "How I built a self-hosted mail server, a custom webmail client, a password manager, a monitoring stack, and somehow kept my sanity. Mostly."
showFullContent = false
readingTime = false
hideComments = false
+++

This one got away from me a little bit.

What started as "I want to host my own email" turned into two days of infrastructure work that I am genuinely proud of, and which I am now going to document before I forget what I actually did.

## Email, Or: How Hard Could It Be

Famously, the answer is: pretty hard. Email is one of those things that looks simple from the outside — you send a message, it arrives somewhere — and reveals itself to be an absolutely cursed protocol the moment you try to run your own server.

I used [docker-mailserver](https://docker-mailserver.github.io/docker-mailserver/latest/) to handle the heavy lifting. It's a battle-tested, single-container mail server that bundles Postfix (outbound), Dovecot (IMAP), Rspamd (spam filtering), ClamAV (virus scanning), and fail2ban into one reasonably sane package. The alternative was configuring all of those individually, which is a project for someone with more patience than me.

DNS is where email lives or dies. SPF, DKIM, and DMARC records tell receiving servers that your mail is legitimate and not phishing bait. Getting those right? Surprisingly easy. Right on the first try. I must have been a savant. A DNS savant. Because that's a thing. I'm starting with `p=none` on DMARC and tightening it to `p=quarantine` once I've warmed up the domain's reputation — sending from a fresh domain gets you into spam folders, and you have to earn your way out.

## I Built a Webmail Client

Okay so. I could have used Roundcube. I did not use Roundcube.

I built my own webmail client — seraph-mail — in React and Node.js. It talks to the mail server over IMAP and SMTP, handles inline images, has a working sent folder, dark mode that doesn't bleed into rendered emails, and a toolbar with the features I actually use. The UI is the aesthetic I want everything on this server to have: monospace, dark, pink accent, clean. It lives at `mail.seraph.ws`.

Is this the practical choice? No. Was it a fantastic learning experience that gave me exactly the interface I wanted? Absolutely.

Security-wise: fail2ban watches the auth log for failed logins and bans offending IPs, nginx serves it with a Content Security Policy header, and the Express backend enforces a 25MB body limit for attachments. The `trust proxy` setting is set correctly so fail2ban sees real IPs rather than localhost. I learned this the hard way.

## Monitoring

I set up [Uptime Kuma](https://github.com/louislam/uptime-kuma) at `status.seraph.ws`. It watches all my services — the main site, webmail, link shortener, QR generator, Plausible, IMAP, SMTP, Postgres, and itself — and sends me an email alert via the mail server if anything goes down.

The Postgres monitor was a fun one. Uptime Kuma runs in Docker and Postgres was only listening on `localhost`, which Docker containers can't reach. The fix was adding `172.17.0.1` (the Docker bridge host IP) to `listen_addresses` in `postgresql.conf` and opening the firewall for the Docker subnet. Then it just worked.

For catastrophic failures — the whole server going dark — I added [UptimeRobot](https://uptimerobot.com/) as an external watchdog. It monitors from outside my infrastructure and alerts directly to my Proton inbox, bypassing the mail server entirely. Chicken and egg problem, solved.

## Vaultwarden

I've been meaning to self-host a password manager for a while. [Vaultwarden](https://github.com/dani-garcia/vaultwarden) is a lightweight, Bitwarden-compatible server — same clients, same browser extensions, your data stays on your hardware.

It's running in Docker at `vault.seraph.ws`, behind nginx with a Let's Encrypt cert. Signups are disabled after account creation so nobody else can register. fail2ban watches the Vaultwarden log for failed logins and bans accordingly.

## AbuseIPDB Integration

All my fail2ban jails now report bans to [AbuseIPDB](https://www.abuseipdb.com/) automatically. Every IP that gets blocked for SSH brute force, bad HTTP auth, webmail stuffing, or Vaultwarden probing gets reported with the appropriate category. It's a small contribution to a collective blocklist that other server operators benefit from, and it costs me nothing.

Speaking of which — CrowdSec has blocked 63,600 attacks in the week since I put this server online. SSH brute force, HTTP scans, exploit attempts, DoS probing. It's genuinely wild out there.

## Mail Forwarding

One small quality-of-life thing: I set up a Sieve script that forwards a copy of everything sent to my admin email to my Proton inbox. This was necessary because docker-mailserver won't let you create an alias for an existing account, which is fair enough. The `:copy` flag means it forwards without removing the original from the seraph inbox.

## What's Left

- Tighten DMARC to `p=quarantine` after the domain warms up
- Set up homelab rsync backups (my girlfriend is handling the architecture on that one, for good security reasons)
- Redesign this site — the theme is a leftover from day one and it's starting to show

