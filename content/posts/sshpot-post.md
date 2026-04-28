---
title: "Building a Self-Hosted SSH Honeypot in Rust"
date: 2026-04-28
draft: false
tags: ["rust", "axum", "sqlite", "security", "self-hosting", "sysadmin"]
keywords: ["ssh honeypot", "rust", "axum", "self-hosting", "crowdsec", "iptables", "threat feed"]
description: "How I built a live SSH honeypot in Rust that captures bot auth attempts, geolocates source IPs, and streams everything to a real-time web dashboard."
---

I built a honeypot.

This has been on my list since I first noticed how many things were hitting port 2222 — my
actual SSH port — within minutes of bringing seraph.ws online. fail2ban and CrowdSec handle
the problem just fine, but I wanted to *see* it. I wanted a live feed of every bot, scanner,
and credential-stuffing script that touched my server. So I built one.

## What It Does

The honeypot listens on port 22 (via an iptables PREROUTING redirect, since binding to
privileged ports without root is annoying). When a connection comes in, it does a proper
SSH handshake using [russh](https://github.com/warp-tech/russh) — a Rust SSH library —
and then captures whatever credentials the client tries. Passwords get stored verbatim.
Public key attempts get logged as `algo:fingerprint` before the client even has to sign
anything, since we reject at the "offered" stage. Either way, the auth always fails.

Every captured attempt gets a geo-IP lookup via ip-api.com (free tier, no key required),
written to SQLite, and broadcast over a tokio channel to any active SSE subscribers. The
web dashboard at [sshpot.seraph.ws](https://sshpot.seraph.ws) consumes that stream and
renders a live firehose feed: timestamp, source IP, country flag, ASN/org, username,
credential. High-value usernames like `root` and `admin` light up in pink.

The whole thing is a single binary — two async tokio tasks (honeypot listener and Axum
HTTP server) sharing an `AppState` with a broadcast channel and a SQLite connection pool.

## The Stack

- **russh** for the SSH server implementation
- **Axum** for the HTTP API and SSE endpoint
- **SQLite via sqlx** for persistence
- **ip-api.com** for geo-IP (country, country code, ASN/org)
- **tokio** broadcast channel for real-time event distribution

The frontend is a single HTML file embedded in the binary with `include_str!()`, styled
to match the rest of seraph.ws — same near-black background, same hot pink accent, same
JetBrains Mono throughout.

## The Deploy

Port 22 redirects to 2223 via iptables:

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2223
sudo netfilter-persistent save
```

The service runs as `www-data` under systemd, the API listens on localhost, and nginx
proxies it to the public subdomain with SSL. One thing worth noting for the nginx config:
SSE requires buffering to be disabled, otherwise the browser never actually receives the
events and just stares at an empty feed looking sad.

```nginx
proxy_buffering off;
proxy_cache off;
proxy_read_timeout 3600s;
proxy_set_header Connection '';
chunked_transfer_encoding on;
```

## CrowdSec Coexistence

I was briefly worried that CrowdSec would start banning IPs before the honeypot got
a chance to log them, which would defeat the purpose. Turns out this isn't an issue:
CrowdSec reads `/var/log/auth.log`, which only gets written to by the real sshd process
on port 2222. russh doesn't touch auth.log at all. The honeypot is completely invisible
to CrowdSec, which means they operate on separate tracks — CrowdSec guards the real SSH
port, the honeypot sees everything that hits port 22 before any banning happens.

## What I've Learned So Far

It has been online for about an hour and the thing I have learned is that people really
like trying `root` with the password `wawa`.

There are also a truly impressive number of IPs from Chinese cloud providers. Not judging,
just noting. The ASN column is doing a lot of work.

I'm planning to add CrowdSec cross-referencing at some point — flag attempts from IPs that
are already in the blocklist as "known bad" with a little badge. The data will be there
from day one since CrowdSec is already running on seraph.ws and has a local API.

[View the live feed](https://sshpot.seraph.ws) · [Source code](https://github.com/more-robots-please/sshpot)
