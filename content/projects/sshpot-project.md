---
title: "SSH Honeypot"
date: 2026-04-28
tags: ["rust", "axum", "sqlite", "security", "self-hosting"]
summary: "A self-hosted SSH honeypot with a live web dashboard. Captures auth attempts, geolocates source IPs, and streams events in real time."
draft: false
---

A self-hosted SSH honeypot running at [sshpot.seraph.ws](https://sshpot.seraph.ws).

Built with Rust, Axum, and SQLite. Listens on port 22 via an iptables redirect,
captures password and public key auth attempts from bots and scanners, geolocates
source IPs, and streams events live to a web dashboard over SSE. Single binary,
two async tasks sharing state through a tokio broadcast channel.

[View live feed](https://sshpot.seraph.ws) · [Source code](https://github.com/more-robots-please/sshpot)
