---
layout: post
title: "Comig soon, keep an eye!"
date: 2026-08-09 08:00:00 +0100
categories: [Web Securiy, Nginx, Proxy]
tags: [AppSec, Pentest, NGINX, Reverse Proxy]
---

# Introduction
## What is Nginx, and Why Does it Matter?
Nginx is the most widely deployed reverse proxy in the world, powering roughly a third of all websites. And, as pentesters, bug bounty hunters or even security analysts we almost always come across it used as a load balancer, reverse proxy, or a simple server when doing engagements. Also, if you do configuration security reviews, you will need to have a good grasp of the common misconfigurations as well as some novel techniques usually weaponized by adversaries to bypass what looks seemingly secure.

In a reverse-proxy role, Nginx sits in front of one or more backend applications, deciding which requests go where, applying access controls, caching responses, terminating TLS and rewriting URLs before they reach the application code behind it. This critical position is what makes Nginx misconfigurations so consequential. A bug in a backend application is usually confined to that application, however, a bug in the proxy level sitting in front of multiple application can easily and quickly enlarge the blast radius, often without the backend applications having any bugs of their own at all.

Nginx configurations are full of directives whose exact semantics are easy to misjudge, whether a trailing slash strips a path prefix, whether one location's directive is inherited by another or not, whether a regex capture group is safe. None of these mistakes look wrong at a glance or a quick skim through the configuration code.

## What a Deliberately Vulnerable Reverse-Proxy Lab
Most available hands-on web security training focuses on the application layer bugs, however, the reverse-proxy level is comparatively under-served; it's covered essentially in write-ups and reference wikis, but rarely as something a learner can actually, run, break and fix end-to-end. While diving into Nginx security this last month, I didn't want to reinvent the wheel, and looked for already implemented vulnerable Nginx proxies, but all I found was projects that didn't showcase novel techniques that reflect latest research done on Nginx security, and for the most part, those project presented only one or two misconfigurations in the most direct way possible, which is too good to be true.

Damn Vulnerable Nginx Proxy exists to close that gap. Every finding we discuss in the guide is reproducible against a real, running reverse proxy. And for the most learning outcome, several findings are not isolated bugs but chains taken directly from real world findings, previous and novel research.

This blog walks through all twenty documented findings twice; first from black-box perspective; what an external attacker with no idea on the configuration would try, and what they would observe, and then from a white-box perspective, explaining precisely which misused directive or line of code caused the bad behavior.

# Lab Architecture
## Component Diagram
The lab run three containers behind a single published port. An Nginx reverse proxy terminates all inbount traffic on port 8080 and routes to two independent Flask backends based on which of the three virtual hosts a request targets.

<img width="1598" height="984" alt="a0e809bb-9900-4751-b5f9-3e494abbf6ac" src="https://github.com/user-attachments/assets/d6c1c22c-acb9-41f6-9f68-5921234e8b30" />


---

**Tags:** `web-security` `nginx` `security-research`
