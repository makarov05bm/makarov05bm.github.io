---
layout: post
title: "Mass Hunting For HTTP/2 Cleartext Smuggling"
date: 2026-08-09 08:00:00 +0100
categories: [HTTP2, Web Securiy, Nginx, Proxy]
tags: [Proxy, Request Smuggling, h2c, HTTP2, NGINX, Apache, HAProxy]
---

In this blog I'm going to go through a systematic approach to test for h2c smuggling on your targets, whether you are doing it full black-box or gray-box. We are going to brush off the foundations needed first, explaining the HTTP headers involved in the exploit, then we move to the fingerprinting you need to do on the target to decide if you should go into the rabbit hole or not, then I'm going to explain how I usually test for this bug when I get a sense that it might be present, and in parallel we are going to see why it happens with sample configurations, and testing on a lab specifically designed so you can play around with the exploit too.
