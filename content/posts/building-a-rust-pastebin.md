+++
title = "Building a Self-Hosted Pastebin in Rust"
date = "2026-03-11T21:00:00Z"
author = ""
tags = ["rust", "axum", "postgresql", "self-hosting", "linux", "sysadmin"]
keywords = ["self-hosted pastebin", "rust axum", "syntect", "syntax highlighting"]
description = "How I built a self-hosted pastebin in Rust with syntax highlighting, burn-after-read, password protection, and expiring pastes."
showFullContent = false
readingTime = false
hideComments = false
+++

At this point I'm just building every classic web utility from scratch in Rust. Last week it was a link shortener. This week: pastebin.

## Why

Same reason as everything else on this server — I want it to be mine. Pastebin.com is fine but it's ad-ridden, has a size limit, and I don't control what happens to the content. Running my own means I can paste whatever I want, set my own limits, and know exactly where the data lives.

## The Stack

Same pattern as the shortener: Rust, Axum, SQLx, PostgreSQL. At this point I have a pretty good mental model of how these pieces fit together, so the scaffolding went fast. The interesting new piece was syntax highlighting.

I used [syntect](https://github.com/trishume/syntect) for server-side syntax highlighting. It ships with a full set of TextMate grammars and a handful of themes — I went with `base16-ocean.dark` because it plays nicely with the site's aesthetic. The highlighted HTML gets pre-rendered at insert time and cached in the database, so views are just a lookup and a template render. No client-side JS, no flash of unstyled code.

The language selector is a curated dropdown of about 20 options rather than exposing the full syntect grammar list. Keeps it clean.

## Features

Pastes support optional titles, a language selector, time-based expiry, burn-after-read, and password protection. Burn-after-read deletes the row from the database on first view — raw view counts too, so no sneaking a peek via curl. Password-protected pastes return the rendered HTML directly on successful auth rather than redirecting, so the URL stays clean.

Raw view lives at `/raw/:code` and serves plain text with the correct content-type header, which means you can pipe it directly into a shell or curl it into a file.

The admin page uses the same bearer token auth pattern as the shortener — token in localStorage, data fetched from protected endpoints. No sessions, no cookies, no overhead.

## Deployment

Single binary, systemd service, nginx reverse proxy, Let's Encrypt cert. Same as everything else. `cargo build --release`, drop a `.env`, and it's running. The whole thing from `cargo new` to live was about two hours.

It's running at [p.seraph.ws](https://p.seraph.ws).
