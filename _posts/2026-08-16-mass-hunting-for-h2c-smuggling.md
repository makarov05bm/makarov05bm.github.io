---
layout: post
title: "Mass Hunting For HTTP/2 Cleartext Smuggling"
date: 2026-08-09 08:00:00 +0100
categories: [HTTP2, Web Securiy, Nginx, Proxy]
tags: [Proxy, Request Smuggling, h2c, HTTP2, NGINX, Apache, HAProxy]
---

In this blog I'm going to go through a systematic approach to test for h2c smuggling on your targets, whether you are doing it full black-box or gray-box. We are going to brush off the foundations needed first, explaining the HTTP headers involved in the exploit, then we move to the fingerprinting you need to do on the target to decide if you should go into the rabbit hole or not, then I'm going to explain how I usually test for this bug when I get a sense that it might be present, and in parallel we are going to see why it happens with sample configurations, and testing on a lab specifically designed so you can play around with the exploit too.

## 1. Introduction
### 1.1 What is the HTTP/1.1 Upgrade Header?
`Upgrade` is a request and response header that can be used to upgrade an already-established client/server connection to a different protocol (over the same underlying transport protocol). For example, a client can use it to upgrade the connection from HTTP/1.1 to HTTP/2 (not important for our case), or to upgrade an HTTP(s) connection to a WebSocket and other similar protocols like H2C (HTTP/2 Cleartext).

> **Warning:** HTTP/2 explicitly disallows the use of this mechanism and header; it is specific to HTTP/1.1.

Read more about it on [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Upgrade)

### 1.2 How this Relates to Proxies?
Proxies support upgrades to WebSocket/H2C by keeping the original client/server connection established and use it as a channel to tunnel the TCP traffic to the backend, at this stage the proxy is no longer context-aware and cannot apply access control rules on the inbound/outbound traffic.

1. Client initiates an HTTP/1.1 upgrade request to protocol X
2. If backend supports protocol X, it responds with a `101 Switching Protocols` response code
3. Client receives that response and start sending data using the newly agreed-on protocol, over the same connection that was used to upgrade the proocol

<img width="983" height="748" alt="image" src="https://github.com/user-attachments/assets/82c9a704-7dbd-4515-bcc4-9a9435284bf8" />

Now, after the reverse proxy received the 101 response from the backend, it will maintain a persistent TCP connection without inspecting the content going back and forth between the client and the backend.
