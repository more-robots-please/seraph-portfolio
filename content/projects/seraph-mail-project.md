---
title: "seraph-mail"
date: 2026-03-11
tags: ["react", "node", "self-hosting", "email"]
summary: "A bespoke webmail client built on top of docker-mailserver. IMAP/SMTP, dark mode, inline images, and a UI that actually looks the way I want."
draft: false
---

A custom webmail client running at [mail.seraph.ws](https://mail.seraph.ws).

Built because I could have used Roundcube and chose not to. React frontend, Node/Express backend, IMAP via `imapflow`, SMTP via `nodemailer`. Handles inline images, threaded sent mail, and dark mode that doesn't bleed into rendered HTML emails.

Sits in front of [docker-mailserver](https://docker-mailserver.github.io/docker-mailserver/latest/) which handles Postfix, Dovecot, Rspamd, and ClamAV. fail2ban watches the auth log, nginx handles TLS and CSP headers.

The UI follows the same design language as the rest of seraph.ws — JetBrains Mono, dark background, pink accent. Built to be used daily, not demoed once.

[Source code](https://github.com/more-robots-please/seraph-mail)
