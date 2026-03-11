---
title: "QR Code Generator"
date: 2026-03-11
tags: ["rust", "axum", "self-hosting"]
summary: "A self-hosted QR code generator written in Rust. Outputs SVG and PNG, supports logo embedding and print mode."
draft: false
---

A personal QR code generator running at [qr.seraph.ws](https://qr.seraph.ws).

Built in Rust with Axum. Generates QR codes client-side as SVG or high-res PNG, with optional logo overlay and a print mode (black on white). The SVG preview updates live as you type.

Uses the `qrcode` crate with high error correction (Level H) to keep codes scannable even with a logo in the centre. PNG export is rendered at 40px per cell — large enough for print without upscaling artifacts.

[Try it](https://qr.seraph.ws) · [Source code](https://github.com/more-robots-please/qr)
