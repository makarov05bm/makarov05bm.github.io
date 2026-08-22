---
layout: post
title: "Bug Hunter's Practical Manual on WebSockets"
date: 2026-08-22 08:00:00 +0100
categories: [WebSocket, Web Security]
tags: [WebSocket, ws, AppSec]
---

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

<img width="1041" height="767" alt="image" src="https://github.com/user-attachments/assets/9385d9a7-dc11-4c66-bb84-4a1f3e540bce" />
*Websockets vs HTTP request/response model*

### URI Structure
```
wss://example.com:443/websocket/demo?foo=bar
└─┘   └──────────┘ └─┘ └────────────┘ └─────┘
 │         │        │        │           │
 │         │        │        │           └── Query
 │         │        │        └───────────── Path
 │         │        └────────────────────── Port
 │         └─────────────────────────────── Host
 └───────────────────────────────────────── Scheme
```

- The WebSocket protocol defines two URI schemes for traffic between server and client
  - **ws**: used for unencrypted connections -> default port 80
  - **wss**: used for secure, encrypted connections over Transport Layer Security (TLS) -> default port 443
- Fragment identifiers are not allowed in WebSocket URIs
  - The hash character (`#`) must be escaped as `%23`
 
### Opening Handshake
Client does not directly hop to using WS, instead, client has to issue an upgrade request over an HTTP GET request. The server then, if it supports WS, and the conditions it imposes are met, responds with a WebSocket handshake response.
```
   ┌──────────┐                              ┌──────────┐
   │  Client  │                              │  Server  │
   └──────┬───┘                              └──────┬───┘
         │                                          │
         │  GET /chat HTTP/1.1                      │
         │  Host: example.com                       │
         │  Upgrade: websocket                      │
         │  Connection: Upgrade                     │
         │  Sec-WebSocket-Key: [base64]             │
         │  Sec-WebSocket-Version: 13               │
         ├─────────────────────────────────────────►│
         │                                          │
         │  HTTP/1.1 101 Switching Protocols        │
         │  Upgrade: websocket                      │
         │  Connection: Upgrade                     │
         │  Sec-WebSocket-Accept: [hash]            │
         │◄─────────────────────────────────────────┤
         │                                          │
         │       WebSocket Connection               │
         │◄────────────────────────────────────────►│
         │                                          │
```

#### 1. Client handshake request
A Websocket handshake request is a normal GET request to a special endpoint, with some special required headers, to indicate to the server that the client is willing to upgrade the connection from HTTP to WS:
```http
GET /chat HTTP/1.1
Host: example.com:8000
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

- `Upgrade: websocket` - indicates that the client wants to upgrade the connection to use the WebSocket protocol
- `Connection: Upgrade` - tells proxies or other intermediaries to also upgrade the connection
- `Sec-WebSocket-Key` - a Base64-encoded random value that helps the server prove it’s a WebSocket-capable server (automatically set by the browser)
- `Sec-WebSocket-Version` - the version of the WebSocket protocol the client wishes to use (almost always 13)

#### 2. Server handshake response
Once the server receives the handshake request, if it supports WS and accepts the request to upgrade, it responds with a `101 Switching Protocols` HTTP response:
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- `HTTP/1.1 101 Switching Protocols` - indicates the successful upgrade from HTTP to WebSocket
- `Upgrade: websocket` - confirms the protocol upgrade
- `Connection: Upgrade` - indicates that the connection has been upgraded
- `Sec-WebSocket-Accept` - a value calculated from the client’s Sec-WebSocket-Key, which helps verify that the server understood the WebSocket handshake request
