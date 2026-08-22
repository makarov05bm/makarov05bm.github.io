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

## 2. WebSocket Authentication Mechanics
As the official documentation states, the browser WebSocket API has no way to set custom HTTP headers, so it's up to the devs to choose the mechanism they wish to verify identity.
```js
// This is the entire browser WebSocket API for connection setup.
// Notice: no headers parameter, no options object.
const ws = new WebSocket("wss://example.com/ws");
// Compare with fetch, which supports arbitrary headers:
// fetch(url, { headers: { Authorization: "Bearer ..." } })
```

What we need to keep in mind, is that as we've seen already that WebSockets are stateful, meaning after the first handshake between the client and the server, they from now on know each other, and there is no need to send the auth token or cookie with each request (typically called a message in WS) unlike with HTTP. But, how does the server prove client's identity then? It happens at the handshake actually, with the first HTTP GET request sent by the client with the `Upgrade`, `Connection`, `Sec-WebSocket-Key`... headers, the client also usually attaches a cookie (or other methods we will see in just a bit) to the request, and the server verifies then caches the user's data from the cookie, only then it responds with `101`. And after that, a communication channel is open between the client and server and there is no need to send the cookie or any other auth mechanism at all. This is so important, so keep it in mind.

There exists currently three approaches to prove identity for WS, each comes with pros and cons.

### 2.1 URL query parameter authentication
Pass the token in the WebSocket URL. The server validates it during the HTTP upgrade handshake, before the connection is established.

**Client**
```js
const ws = new WebSocket(
  `wss://example.com/ws?token=${encodeURIComponent(token)}`
);
}
```

**Server**
```js
server.on("upgrade", (req, socket, head) => {
  const url = new URL(req.url, `http://${req.headers.host}`);
  const token = url.searchParams.get("token");

  const user = validateToken(token);

  // Auth token is not valid, reject the upgrade
  if (!user) {
    socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n");
    socket.destroy();
    return;
  }

  // Auth token is valid, so upgrade
  wss.handleUpgrade(req, socket, head, (ws) => {
    ws.user = user; //  <--  user's data caching
    wss.emit("connection", ws, req);
  });
});
```
