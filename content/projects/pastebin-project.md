---
title: "Pastebin"
date: 2026-03-11
tags: ["rust", "axum", "postgresql", "self-hosting"]
summary: "A self-hosted pastebin written in Rust. Syntax highlighting, burn-after-read, expiring pastes, and password protection."
draft: false
---

A self-hosted paste sharing service running at [p.seraph.ws](https://p.seraph.ws).

Built with Rust, Axum, and PostgreSQL. Supports syntax highlighting via `syntect` (pre-rendered server-side), time-based expiry, burn-after-read, and password-protected pastes. Raw view at `/raw/:code` for piping into curl etc.

[Try it](https://p.seraph.ws) · [Source code](https://github.com/more-robots-please/pastebin)
