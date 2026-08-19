---
layout: post
title: "Mass Hunting For HTTP/2 Cleartext Smuggling"
date: 2026-08-19 08:00:00 +0100
categories: [HTTP2, Web Securiy, Nginx, Proxy]
tags: [Proxy, Request Smuggling, h2c, HTTP2, NGINX, Apache, HAProxy]
---

In this blog I'm going to go through a systematic approach to test for h2c smuggling on your targets, whether you are doing it full black-box or gray-box. We are going to brush off the foundations needed first, explaining the HTTP headers involved in the exploit, then we move to the fingerprinting you need to do on the target to decide if you should go into the rabbit hole or not, then I'm going to explain how I usually test for this bug when I get a sense that it might be present, and in parallel we are going to see why it happens with sample configurations, and testing on a lab specifically designed so you can play around with the exploit too.

## 1. Introduction
### 1.1 What is the HTTP/1.1 Upgrade Header?
`Upgrade` is a request and response header that can be used to upgrade an already-established client/server connection to a different protocol (over the same underlying transport protocol). For example, a client can use it to upgrade the connection from HTTP/1.1 to HTTP/2 (not important for our case), or to upgrade an HTTP(s) connection to a WebSocket and other similar protocols like h2c (HTTP/2 Cleartext).

> **Warning:** HTTP/2 explicitly disallows the use of this mechanism and header; it is specific to HTTP/1.1.

Read more about it on [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Upgrade)

### 1.2 How this Relates to Proxies?
Proxies support upgrades to WebSocket/H2C by keeping the original client/server connection established and use it as a channel to tunnel the TCP traffic to the backend, at this stage the proxy is no longer context-aware and cannot apply access control rules on the inbound/outbound traffic.

1. Client initiates an HTTP/1.1 upgrade request to protocol X
2. If backend supports protocol X, it responds with a `101 Switching Protocols` response code
3. Client receives that response and start sending data using the newly agreed-on protocol, over the same connection that was used to upgrade the proocol

<img width="983" height="748" alt="image" src="https://github.com/user-attachments/assets/82c9a704-7dbd-4515-bcc4-9a9435284bf8" />

Now, after the reverse proxy received the 101 response from the backend, it will maintain a persistent TCP connection without inspecting the content going back and forth between the client and the backend.

Quoting from this [F5](https://www.f5.com/company/blog/nginx/websocket-nginx) blog post:
> NGINX supports WebSocket by allowing a tunnel to be set up between a client and a backend server.

And it's the same for **h2c**, which is an unencrypted version of HTTP/2 that runs over a plain TCP connection without TLS, and retains request multiplexing over a single TCP connection.

### 1.3 Upgrade HTTP/1.1 Connection to Persistent h2c
Usage of HTTP/2 can be achieved using multiple methods, one of which is the use of the `Upgrade` header.

```http
GET / HTTP/1.1
Host: www.target.tld
Upgrade: h2c
HTTP2-Settings: AAMAAABkAARAAAAAAAIAAAAA
Connection: Upgrade, HTTP2-Settings
```

> **Note:** The Connection header with type upgrade must always be sent with the Upgrade header.

And extracted from [RFC 7540 Section 3.2.1](https://datatracker.ietf.org/doc/html/rfc7540#section-3.2.1)
```
   A request that upgrades from HTTP/1.1 to HTTP/2 MUST include exactly
   one "HTTP2-Settings" header field.  The HTTP2-Settings header field
   is a connection-specific header field that includes parameters that
   govern the HTTP/2 connection, provided in anticipation of the server
   accepting the request to upgrade.

     HTTP2-Settings    = token68
```

So `HTTP2-Settings` is only received by the proxy and no need to forward it to the backend.

## 2. White-Box Side of Things
### 2.1 When and Where h2c is Exploitable?
By default most of proxies do not forward the `Upgrade` and `Connection` headers, including NGINX, Apache, Envoy, AWS ALB/CLB... And some others forward them by default such as HAProxy, Traefik and Nuster.
However, this does not mean all NGINX instances are secure right away, as this is not a bug at the NGINX level, but a misconfiguration, so it can happen and is proved to happen in the wild.

### 2.2 Vulnerable Nginx Configuration
`Golang` micro services specifically tend to support h2c connections as it's a really good use case for communications between micro services in a network.

```nginx
server {
    listen 443 ssl;
    server_name www.skyblue.com;

    ssl_certificate /etc/ssl/certs/skyblue.com.pem;
    ssl_certificate_key /etc/ssl/certs/skyblue.com-key.pem;

    root /var/www/main/html;
    index index.html index.htm index.nginx-debian.html;

    location / {
      proxy_pass http://127.0.0.1:80;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $http_connection;
    }
}

server {
  default 80;

  server_name portal.skyblue.com;

  location /billing {
    proxy_pass http://billing:9999;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }

  location /billing/admin {
    deny all;
  }
}
```

The NGINX configuration above does the following:
- TLS termination on port 443
- The / location at the 443 server forwards the `Upgrade` and `Connection` headers, which is very typical if the backend supports WebSockets too
- The `/billing` endpoint forwards request to the h2c-supporting backend service (same thing if the endpoint was pointing to backend websocket route)
- `/billing/admin` is gated, no one should be able to reach it via the proxy

Let's test requesting the two endpoints:
<img width="1467" height="256" alt="image" src="https://github.com/user-attachments/assets/50b261e7-5780-43a6-bc2b-2f77749db81b" />

We see that `/billing/admin` causes a `403` from the NGINX, meaning NGINX inspected the traffic as it does with normal HTTP.

### 2.3 Demo Authorization Bypass
One of the many techniques that an adversary can weaponize to bypass `403`s is the use of `h2c` in case the backend happens to be supporting it, so from a black-box stand point, it's a game of trial and error (with some clever fingerprinting is some cases, which we will take a look at later).

The [spec](https://datatracker.ietf.org/doc/html/rfc7540#section-3.3) mentions that `h2c` upgrades over TLS are not allowed, so HTTP clients such as curl won't let us do that, so instead we will need to craft our own client to send the `h2c` upgrade request, and luckily we already have a script that does just that. [h2csmuggler by BishopFox](https://github.com/BishopFox/h2csmuggler).

To exploit h2c, we should determine the following:
1. The entry endpoint that proxies requests to a backend service which supports `h2c`
   - in our case it's `/billing`, which forwards the `Upgrade` and `Connection` headers
2. A gated endpoint which exists in the backend application that supports `h2c`
  - in our case it's `/billing/admin`

Now, We can use h2cSmuggler to confirm the proxy's insecure configuration using `--test` (or `-t`):
<img width="1427" height="130" alt="image" src="https://github.com/user-attachments/assets/d884a232-b672-4e4a-8697-d1fe55d258b2" />

This means that `/billing` Nginx location successfully forwarded the `Upgrade` and `Connection` to the backend as follows:
```
Upgrade: h2c
HTTP2-Settings: AAMAAABkAARAAAAAAAIAAAAA
Connection: Upgrade, HTTP2-Settings
```

And the backend responded with `101 Switching protocols`. However, keep in mind that the backend may return `101` but not serve requests on the TCP tunnel we aim to create, that's why we test against an endpoint that previously returned `403` to confirm and verify the exploitability (required for bounty hunters). So, let's do just that using h2csmuggler:
<img width="1594" height="534" alt="image" src="https://github.com/user-attachments/assets/5aff64e9-bdbf-4a73-8125-ede2a7919795" />

This confirms the bypass, i.e, the TCP tunnel was established between the client and backend and NGINX became blind. 

## 3. Fingerprinting and Recon at Scale
As I already mentioned, some reverse proxies handle forwarding the `Upgrade` and `Connection` headers by default such as HAProxy and other don't like NGINX. So finding out which is used would give us a better view if the next steps, however this is not a hard requirement and we can test for the exploit blindly. Nonetheless, there are multiple techniques to fingerprint the reverse proxy being used, these inspection techniques include:
- **Examine HTTP Headers:** Check the Server or Via fields in response headers, though administrators often change or hide these
- **Analyze Error Pages:** Request a non-existent path or trigger a 400 Bad Request; default error page HTML structures, styling, or specific phrasing are often unique to software like Nginx, Envoy, or HAProxy
- **Test Malformed Requests:** Send invalid HTTP methods or strange characters to see how the edge handles syntax rejection differently, which can reveal what reverse proxy is being used

Next step is collecting the endpoints that point to the backend services that may support `h2c`. As we've seen in the sample configuration above, it's not necessary that the location that proxies to an h2c-supporting backend is `/`, but can be anything, such as in the sample; `/billing`. So, the logical thing to do now, is fuzz for all endpoints, at multiple levels (e.g, `/billing`, `/billing/admin`, `/billing/admin/user-list`...), then save the `200`s in a file, and the `403`s in another file alone.

Let's see the difference response behavior to sending a request with the headers:
```
Upgrade: h2c
HTTP2-Settings: AAMAAABkAARAAAAAAAIAAAAA
Connection: Upgrade, HTTP2-Settings
```

<img width="1723" height="556" alt="image" src="https://github.com/user-attachments/assets/6289affa-5a90-436d-b1cc-547fb262545d" />
*/changelog is an endpoint that does not route to an h2c backend, reponse is 200, or it can be any other except for 101*

<img width="1723" height="556" alt="image" src="https://github.com/user-attachments/assets/4ca259de-11d0-4e8f-a9a8-8bfb1bee54e2" />
*/billing routes traffic to a backend service that supports h2c, so response was 101*

Next, we use `h2csmuggler` to test if any of the endpoints that returned `200` can be used for tunneling:
```
python3 h2csmuggler.py --scan-list urls.txt --threads 5 2>errors.txt 1>results.txt
```

Alternatively, you can write a simpler script that sends h2c upgrade requests to the endpoints that returned `200`, and save the ones that respond with `101 Switching Protocols`.

Last step, now you have two final lists, one for the endpoints that can used for tunneling, and the other for any endpoint that return 403 that you aim to bypass, all that we need to do now is write another script that would use h2csmuggler to issue the command:
```
python3 h2csmuggler.py -x https://127.0.0.1:8090/XXXX https://127.0.0.1:8090/ZZZZ
```
Where `XXXX` is the endpoint that can be used to upgrade to `h2c` and `ZZZZ` is the `403` endpoint we want to bypass.

# TO BE CONTINUED
