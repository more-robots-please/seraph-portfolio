---
title: "Self-Hosting a Hugo Site with a Git Push Deploy Pipeline"
date: 2026-03-06
draft: false
tags: ["linux", "self-hosting", "hugo", "nginx", "git", "sysadmin"]
keywords: ["self-hosting", "hugo", "nginx", "git deploy hook", "vps", "certbot", "crowdsec", "ubuntu"]
description: "How I built and deployed a personal site on a hardened Ubuntu VPS using Hugo, nginx, and an automated git push deploy pipeline — from scratch."
---

I recently set up this site on a self-managed VPS instance rather than using a managed platform like GitHub Pages or Vercel. This post is a writeup of everything involved — partly as documentation for my future self, partly because I couldn't find a single resource that covered all of it end to end.

## Why Self-Host?

I've been using Linux for most of my adult life but truth be told, I am an absolute novice when it comes to system administration. My Arch installation is an absolute mess full of deprecated packages, broken symlinks, and the SSD isn't even living in its original machine since my desktop started failing. This project is first and foremost a way to teach myself proper system administration, development pipelines, get some familiarity with the tools I've just never fully bothered to learn, let alone master, and eventually be a cute little portfolio I can stick on my resume ^.^

Why should you self-host? Let's be honest, if you're thinking about self-hosting, you're probably the exact kind of person that *wants* to self-host, so you don't need any convincing from me. For the rest of you, well, let's put it this way: managed hosting providers prey on your inability to maintain your own website, webspace, system, etc. Wordpress sites are notoriously insecure and overreliance on database files leads to poor performance, and god forbid you use a server management panel. Might as well just give up and go back to a managed hosting provider.

I'm going to cut it short here before I sound like a broken record, but what I primarily value is **control.** And you just can't get that without root access.

## Provisioning and Hardening the VPS

This part's easy. I got a VPS from [IONOS](https://ionos.com), but you can pick any provider you like. I didn't have any machines lying around I felt like sacrificing to Proxmox, and hypervisors are somewhat beyond me for the moment, but I'm sure that's a future project.

### SSH Hardening

First thing I did once the VPS was ready was harden SSH access. You would be amazed by the number of people who simply don't do this and fall prey to brute force attempts. So with my first (and last) ssh attempt into root, I set up a user administrator, gave them sudo permissions, and disabled root ssh access.

Log out and back in, this time with the new user account. I probably should have waited to disable root ssh until I made sure I was able to log in with the new user, but hindsight is 20/20. Nevertheless, I didn't lock myself out inadvertently and I was free to continue.

I generated a public/private key pair on my workstation and saved the public key on the server. I then forced key pair authentication and disabled password fallback. Finally, I moved ssh access to arbitrary port 2222 and closed port 22. From there, I enabled ufw, set default incoming to closed, default outgoing to open, and manually opened the following incoming ports: 2222, 80 (HTTP), 443 (HTTPS).

From there, I restarted the SSH service, logged out, and logged back in.

And I locked myself out.

Rather than panicking, I decided to do the easiest thing first: turning it off and on again. A quick reboot got me back in, so I spent the next few minutes making sure this wouldn't happen again. I opened port 2223 as a backup, set up a listener daemon that would restart the ssh service if it ever ends up crashing, and triple checked my configuration. Satisfied with my work, I moved on to hardening the rest of my system.

### fail2ban and CrowdSec

Setting up fail2ban was easy. Install the package, modify the config file to your liking, and make sure you have your jail set up. I made sure to set it up for ssh and nginx, set the number of max attempts, and fired up the service. Easy peasy.

[CrowdSec](https://www.crowdsec.net/) was a bit more involved. It's a fantastic preemptive security tool. While fail2ban blocks known bad actors after failed intrusion attempts, CrowdSec is a collaborative tool that allows you to pool together a database of malicious IP addresses with users around the world. There's a simple install script for each component, and then you register your machine in their online panel. The free version of the software gives access to a handful of blacklist databases, some remediation components, and analytics for it all. It felt great coming back two days later to see all of the blocked traffic, what kind, where they're located, etc. Phenomenal tool, and I can't recommend it enough.

### Cron Jobs

I set up a few daily cron jobs - rkhunter and chkrootkit to scan for rootkits, certbot renewal, and an AIDE scan for file integrity. I also set up a firewall health check (daily) to prevent myself from being locked out again.

## Domain, DNS, and HTTPS

When I bought the VPS, I also registered a domain (seraph.ws). Pointing my A record to the VPS was easy enough - your registrar should have easy-to-follow instructions on how to configure those. Eventually I'd like to route traffic through Cloudflare, but that's a project for another day.

My DNS config is pretty barebones, but I don't have a lot of services connected to the domain yet, and I'm saving mail server setup for another day. 

Setting up an SSL certificate with Certbot couldn't be easier (and free!). Install nginx, add your domain to the config file, install certbot, and run the setup script. Agree to the TOS, and add a renewal cronjob. Done, done, and done. 

## Building the Site with Hugo

I set up the site with [Hugo](https://gohugo.io/), a static site builder written in Go. I wanted to get the site online same-day, and I've always been a tumblrina at heart, so the blog structure appeals to me. I used the [Terminal](https://github.com/panr/hugo-theme-terminal) theme to get me started, added a brief about page, set up a favicon, as well as a custom color scheme with css. 

## The Git Deploy Pipeline

This was the most satisfying part to get working.

I set up two git repositories: a working repo in the site folder (linked to GitHub for remote backup) and a bare repo in my home folder with a post-receive hook that would have Hugo rebuild the site for me every time I pushed a commit. I was originally going to have it restart nginx as well, but tore my hair out troubleshooting the permission and ownership structure, so I decided against it. It's a static site anyway, so there's no need to involve more services than I have to. 

## Version Control and Remote Backup

I set up my git repo so that every time I pushed an update, it would push to both the local repo and my github (which gives me a remote backup and ability to version control). It's very satisfying to hit enter on the git push command and see my site build and my github update with the new files. I did some coding projects in the past, and git felt like a chore to set up and get in the habit of using, but this is pretty nifty!

## Remote Editing Workflow

Almost everything was done in the terminal. I've uploaded maybe three files since setting up the VPS. I'm not always at the same workstation, so it's nice to have a workflow that allows me to do everything remotely, and [Fresh editor](https://getfresh.dev/) has been awesome for making that happen. It lives in the terminal, so I can use it over SSH, and it even supports mouse input, text selection, etc. I can have a terminal for troubleshooting and a file explorer to navigate between project files, all without having to learn vim or emacs (sorry oldheads <3) I'm planning on setting up a repo on my home computer so that I can have local backups and work with Kate (my favorite text editor) but Fresh has honestly been so convenient that I might not even bother.

## Key Takeaways

Setting up a website is honestly not as difficult as I thought it would be! I really enjoyed the process of hardening the system, and it feels really good to keep everything organized. I learned some really valuable things about scripting (cron is a really powerful utility omg omg omg) and I want to write up a couple bash scripts to do automated system maintenance on my home computer via cronjobs. I've gotten really comfortable navigating a system in the terminal (tree is a lifesaver when you feel lost btw) and gained some really valuable experience for my professional career. Fresh is awesome, go check it out! If you need anything more than a quick edit, it really really shines. I might want to build something similar in [Textual](https://textual.textualize.io/) for Python (or maybe something different altogether!)

## What's Next

I'm planning to build a URL shortener in Rust next — it'll live at `s.seraph.ws` and I'll write it up here when it's done.
