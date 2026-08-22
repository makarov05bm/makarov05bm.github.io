---
layout: post
title: "Bug Hunter's Practical Manual on WebSockets"
date: 2026-08-22 08:00:00 +0100
categories: [WebSocket, Web Security]
tags: [WebSocket, ws, AppSec]
---

**WORKING ON IT!**

## 1. WebSocket Protocol Overview
The WebSocket protocol enables ongoing, full-duplex, bidirectional communication between web servers and web clients over an underlying TCP connection.

```
        ┌──────────┐                    ┌──────────┐
        │  Client  │                    │  Server  │
        └────┬─────┘                    └────┬─────┘
             │                               │
             │      Initial HTTP             │
             │       handshake               │
             ├──────────────────────────────►│
             │◄──────────────────────────────┤
             │                               │
             │      WebSockets               │
             │      full-duplex              │
             │      persistent               │
             │◄─────────────────────────────►│
             │                               │
             │         Close                 │
             ├──────────────────────────────►│
             │◄──────────────────────────────┤
```

In simple terms, the WS protocol consists of two phases; an opening handshake done over HTTP, where the HTTP connection is upgraded to WS, leading to opening a communication channel over the same underlying connection, which is then used by the two parties of the connection (client/server) to send data back and forth at will, independently of each other.

As stated by the [official website](https://websocket.org/guides/websocket-protocol/):
> The WebSocket protocol (RFC 6455) upgrades an HTTP connection to a persistent, full-duplex channel. After a handshake, client and server exchange lightweight frames - text, binary, or control - with minimal overhead. It works over HTTP/1.1, HTTP/2, and HTTP/3.

