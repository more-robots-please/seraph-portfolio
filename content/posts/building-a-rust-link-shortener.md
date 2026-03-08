---
title: "Building a Self-Hosted Link Shortener in Rust with Axum and PostgreSQL"
date: 2026-03-07
tags: ["rust", "axum", "postgresql", "self-hosting", "nginx", "linux", "sysadmin"]
summary: "How I built and deployed a self-hosted link shortener in Rust using Axum, PostgreSQL, and SQLx — with input sanitization, URL validation, click analytics, and a token-protected admin interface."
draft: false
---

## Intro

Link shorteners are honestly not really fashionable anymore - a relic of a bygone era, before QR codes, and before every messaging platform known to man implemented page previews. They can be an attack vector for phishing attempts, and if you have to share your page through a print medium? Just scan the QR code. Nevertheless, it's a fun project, and I'm sure I'll find some utility for it one way or another.

I built the application in Rust, using Tokio to handle multiple requests simultaneously, Axum to handle HTTP requests, SQLx and PgPool to implement database connections, and PostgreSQL for the database backend.

There's also a basic admin page that uses token authentication to protect the data endpoints.

## The Stack

I chose Rust mostly because I'd like to familiarize myself with it, but it's pretty well suited to this kind of application. There's a single binary file that runs as a systemd service, Cargo is honestly just really fucking cool, Rust itself has really good performance, and I personally despise C syntax coming from Python as my first language.

I used Axum as the web framework - it's built around Tokio's async runtime, so the two work well together. The library handles all of the HTTP requests and, truth be told, the syntax feels **really** elegant.

I used PostgreSQL for the database which is really overkill, but it's got robust functionality, and I'd really like to be more familiar with it - sooner rather than later. Besides, this is good practice, and this project is about teaching myself.

I used SQLx to communicate with the database. It maintains a pool of open connections via PgPool and runs database queries. It's asynchronous, so this is where Tokio comes in.

Since my site is already being served with nginx, I didn't really see any reason to switch to Apache for this project. The application runs as a systemd service, so I can start and restart it as I like. Pretty nifty if you ask me!

## Setting Up

This project started like any other Rust project — `cargo new shortener` — and a `Cargo.toml` with dependencies. The interesting part of the setup is the database migration.

SQLx has a CLI tool that manages migrations — versioned SQL files that define your database schema. The idea is that instead of manually running SQL commands against your database, you write them as files and SQLx tracks which ones have been applied. This means your schema is version-controlled alongside your code, and setting up a fresh database is just `sqlx migrate run`. The migration for this project is a single file that creates the `links` table:

```sql
CREATE TABLE links (
    id SERIAL PRIMARY KEY,
    code TEXT NOT NULL UNIQUE,
    url TEXT NOT NULL,
    clicks INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Configuration lives in a `.env` file with three values — the database connection string, the base URL for building short links, and the admin token. The `dotenvy` crate loads this at startup so the values are available as environment variables. The `.env` file is gitignored so secrets don't end up in version control.

## How the Shortener Works

### The Runtime — Tokio

Rust is normally synchronous — code runs one thing at a time and a thread blocks while waiting for I/O. This is fine for a lot of programs but terrible for a web server, where you might have hundreds of requests all waiting on database queries simultaneously.

Tokio is an async runtime that solves this. Instead of blocking a thread while waiting for PostgreSQL to respond, an async function suspends at the `.await` point and hands control back to the runtime. Tokio can then pick up another request, work on that, and come back when the database responds. The `#[tokio::main]` macro at the top of `main()` is what starts this runtime — it's what makes the whole program async.

### The Router — Axum

Axum's `Router` is a lookup table that maps URLs and HTTP methods to handler functions. When a request comes in, Axum checks the URL and method, finds the matching entry, and calls the right function. Path parameters like `/:code` match any value in that position and make it available to the handler.

The `State` system is how shared data gets passed into every handler without resorting to global variables. The database pool, base URL, and admin token are bundled into an `AppState` struct, attached to the router once with `.with_state(state)`, and then any handler that needs them just declares `State(state): State<AppState>` as a parameter and Axum injects it automatically.

### The Database Connection — SQLx + PgPool

Opening a database connection is slow. `PgPool` keeps a pool of open connections that get reused across requests rather than opened and closed on every query. The pool is stored in `AppState` and cloned into each handler — cloning a `PgPool` just gives you another handle to the same pool, not a new set of connections.

SQLx sends raw SQL to PostgreSQL and maps the results back to Rust structs. The `#[derive(sqlx::FromRow)]` attribute on the `Link` struct generates the mapping code automatically — column names are matched to struct field names. There's no ORM magic, no generated queries — you write the SQL, SQLx sends it, and you get typed results back.

### The Redirect Flow

When someone visits `s.seraph.ws/home`, nginx receives the request and forwards it to the Rust process on port 3000. Axum matches `/:code` and calls the redirect handler, which runs this query:

```sql
UPDATE links SET clicks = clicks + 1 WHERE code = $1 RETURNING *
```

This is an atomic operation — it increments the click counter and returns the updated row in a single round trip. If the row exists, Axum sends a 308 redirect with the `Location` header set to the original URL and the browser follows it. If nothing matches, it's a 404.

### The Shortening Flow

`POST /api/shorten` accepts a JSON body with a `url` field and an optional `code` field. Axum and Serde handle deserializing the JSON automatically. From there:

1. If no scheme is present, `https://` is prepended — so `google.com` becomes `https://google.com`
2. The URL is validated for length and scheme
3. If a custom code was provided, it's validated to only contain alphanumeric characters, hyphens, and underscores, and checked against a banned words list
4. `reqwest` makes a HEAD request to the URL to verify it actually resolves — if the URL starts with `http://`, it first tries the `https://` version and upgrades if that works
5. The database is checked for an existing entry for this URL — if one exists, the existing short link is returned instead of creating a duplicate
6. The new row is inserted and the short URL is returned as JSON

### Auth

The auth system is intentionally simple. There are no sessions, no cookies, no JWTs. Every request to a protected endpoint just checks the `Authorization: Bearer <token>` header against the value from `.env`. If it matches, the request proceeds. If not, it gets a 401. The admin page HTML is served to anyone — the security comes from the fact that the data endpoints are useless without the token.

## Security Considerations

Beyond the token auth, a few other things are worth mentioning.

Custom codes are validated against an allowlist of characters — only letters, numbers, hyphens, and underscores are permitted. This prevents injection attacks and ensures codes are always URL-safe.

The banned words list is a plain text file with one word per line, loaded at startup. It's gitignored so it doesn't end up in version control. Any code that contains a banned word as a substring is rejected.

URL deduplication means the same URL can only be in the database once. Without this, someone could flood the database by repeatedly shortening the same link with slightly different payloads.

The URL resolution step means the shortener won't accept links that don't actually work. It also means that if someone tries to shorten `http://example.com` and `https://example.com` exists, the stored URL will be the https version.

## The Admin Interface

The admin page at `/admin` is a single HTML file with inline JavaScript. On load it checks `localStorage` for a saved token — if none is found it shows a login form, and if one is found it immediately fetches and renders the link table. The token is saved to `localStorage` on first login so you don't have to enter it every time.

All the data comes from API endpoints that require the token — the page itself is just a shell that fetches JSON and builds the table dynamically. There's no server-side rendering involved beyond serving the initial HTML.

## Deployment

Building for production is just `cargo build --release`. Rust compiles everything — the application code and all dependencies — into a single self-contained binary. No runtime to install, no dependencies to manage on the server beyond the PostgreSQL instance it connects to.

The binary runs as a systemd service, which means it starts automatically on boot, restarts if it crashes, and logs to the journal. It listens on `127.0.0.1:3000` — localhost only — and nginx proxies requests to it from the outside world. Certbot handles TLS.

The nginx config also sets a Content Security Policy header that restricts what resources the browser is allowed to load. This is a defence-in-depth measure — even if someone managed to inject content into a page, the CSP would prevent it from loading external scripts or making requests to unexpected domains.

## What's Next

A few things I'd like to add at some point:

- **Expiring links** — links that automatically deactivate after a set time or number of clicks
- **QR code generation** — auto-generate a QR code for each shortened link
- **Per-link passwords** — optional password protection on individual links
- **Better analytics** — the click counter works but it would be nice to see referrers and rough geographic data