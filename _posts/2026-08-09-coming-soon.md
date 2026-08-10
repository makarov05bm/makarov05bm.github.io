---
layout: post
title: "Comig soon, keep an eye!"
date: 2026-08-09 08:00:00 +0100
categories: [Web Securiy, Nginx, Proxy]
tags: [AppSec, Pentest, NGINX, Reverse Proxy]
---

# 1. Introduction
## What is Nginx, and Why Does it Matter?
Nginx is the most widely deployed reverse proxy in the world, powering roughly a third of all websites. And, as pentesters, bug bounty hunters or even security analysts we almost always come across it used as a load balancer, reverse proxy, or a simple server when doing engagements. Also, if you do configuration security reviews, you will need to have a good grasp of the common misconfigurations as well as some novel techniques usually weaponized by adversaries to bypass what looks seemingly secure.

In a reverse-proxy role, Nginx sits in front of one or more backend applications, deciding which requests go where, applying access controls, caching responses, terminating TLS and rewriting URLs before they reach the application code behind it. This critical position is what makes Nginx misconfigurations so consequential. A bug in a backend application is usually confined to that application, however, a bug in the proxy level sitting in front of multiple application can easily and quickly enlarge the blast radius, often without the backend applications having any bugs of their own at all.

Nginx configurations are full of directives whose exact semantics are easy to misjudge, whether a trailing slash strips a path prefix, whether one location's directive is inherited by another or not, whether a regex capture group is safe. None of these mistakes look wrong at a glance or a quick skim through the configuration code.

## Why a Deliberately Vulnerable Reverse-Proxy Lab
Most available hands-on web security training focuses on the application layer bugs, however, the reverse-proxy level is comparatively under-served; it's covered essentially in write-ups and reference wikis, but rarely as something a learner can actually, run, break and fix end-to-end. While diving into Nginx security this last month, I didn't want to reinvent the wheel, and looked for already implemented vulnerable Nginx proxies, but all I found was projects that didn't showcase novel techniques that reflect latest research done on Nginx security, and for the most part, those project presented only one or two misconfigurations in the most direct way possible, which is too good to be true.

Damn Vulnerable Nginx Proxy exists to close that gap. Every finding we discuss in the guide is reproducible against a real, running reverse proxy. And for the most learning outcome, several findings are not isolated bugs but chains taken directly from real world findings, previous and novel research.

This blog walks through almost all twenty documented findings twice; first from black-box perspective; what an external attacker with no idea on the configuration would try, and what they would observe, and then from a white-box perspective, explaining precisely which misused directive or line of code caused the bad behavior.

# 2. Lab Architecture
## 2.1 Component Diagram
The lab run three containers behind a single published port. An Nginx reverse proxy terminates all inbount traffic on port 8080 and routes to two independent Flask backends based on which of the three virtual hosts a request targets.

<img width="1598" height="984" alt="a0e809bb-9900-4751-b5f9-3e494abbf6ac" src="https://github.com/user-attachments/assets/d6c1c22c-acb9-41f6-9f68-5921234e8b30" />

## 2.2 Project Structure
```
├── app
│   ├── app.py
│   ├── requirements.txt
│   ├── sensitive
│   │   └── aws.key
│   ├── static
│   │   ├── css
│   │   │   └── style.css
│   │   └── js
│   │       └── main.js
│   ├── static-data
│   │   ├── cat.jpeg
│   │   ├── fish.jpg
│   │   └── README.txt
│   ├── templates
│   │   ├── admin.html
│   │   ├── api.html
│   │   ├── base.html
│   │   ├── download_error.html
│   │   ├── home.html
│   │   ├── private.html
│   │   ├── public.html
│   │   ├── secret.html
│   │   └── topsecret.html
│   ├── uploads
│   └── uploads.db
├── app-2
│   ├── app.py
│   ├── static
│   │   ├── css
│   │   │   └── style.css
│   │   └── js
│   │       └── main.js
│   └── templates
│       ├── admin.html
│       ├── base.html
│       ├── docs.html
│       ├── home.html
│       ├── repo.html
│       └── report.html
├── dev
│   ├── html
│   │   └── index.html
│   └── static
│       ├── db_backup_2026-08-01.sql
│       ├── deploy.sh
│       ├── img-1.jpg
│       └── img-2.jpg
├── docker-compose.yml
├── nginx
│   ├── flask.conf
│   └── flask-v2.conf
├── static-data
└── user-data
    ├── customer_data.csv
    └── product_list_2026.csv
```

# 3. Methodology
## 3.1 Black-Box vs White-Box Testing is this Guide
Every finding below is presented twice. The black-box section describes what a tester with no access to the configuration or source code would try, and what response would tip them off that something is wrong and worth digging down the rabbit hole. The white-box section then opens the configuration and application code and explains, directive by directive, exactly why the behavior happened.

Only going through the black-box is generally enough if you your work does not involve doing code review, however it's still a good skill to have as it builds the deeper skill; recognizing patters and imagining how the proxy is configured without having actual access to the config file.

## 3.2 Finding List: What We Will Go Through?
### 3.2.1 portal.skyblue.com
1. Path Traversal / LFI via Root proxy_pass Without Upstream URI
2. Authorizaion Bypass via Unvalidated Regex Capture in proxy_pass
3. IP Spoofing via Missing proxy_set_header Inheritance
4. Authentication Bypass by Cookie Replay (Static Session Token via auth_request)
5. CORS Misconfiguration via Missing Regex Anchor
6. Denial of Service via Unbounded Request Body
7. Web Cache Deception via Extension-Based Cache Matching
8. Open Redirect via User-Registerable Cloud Storage Bucket Name
9. HTTP Response Splitting via Unsanitized Regex Capture
10. Access Control Bypass via a Permissive Alternate Root
11. Default-Allow Gap in the map Directive
12. SSRF via Client-Controlled Host Header

### 3.2.2 sandbox-dev-001.skyblue.com
1. Location Match-Priority Bypass via ^~ Overriding a Regex deny Rule
2. Stored XSS via Unescaped Log Injection
3. Information Disclosure via Exposed stub_status

### 3.2.3 sandbox-dev-002.skyblue.com
1. Open Redirect via Missing Leading Slash in a Rewrite Capture Group
2. Access Control Inconsistency - auth_basic Not Inherited by a More-Specific Location
3. Credential Brute-Force via Missing Rate Limiting on Basic Auth
4. Authentication Bypass via satisfy any and a Broken IP Trust Boundary

---

# Findings: portal.skyblue.com
This is the main domain (set via the `default_server` directive in the nginx confi file), and it's what we as pentesters or bug bounty hunters don't usually spend much time navigating through, it's a boring status page!
<img width="2147" height="1389" alt="image" src="https://github.com/user-attachments/assets/6d45a19e-46c8-466f-95b9-ae1b8e3fab96" />

## 1. Access Control Bypass via a Permissive Alternate Root
### Black-Box Discovery
1. Doing fuzzing you stumble upon some paths that reutrn `403`, like the following:
```
GET /secret -> 403
GET /admin -> 403
GET /private -> 403
GET /topsecret -> 403
```
2. While browsing the application, you notice that multiple requests seem to always append a prefix to whatever path they aim to access, like:
```
GET /en-us/status
GET /en-us/public
```
3. This prefix can be your friend in bypassing those `403`s, just append it to them too!
<img width="2159" height="328" alt="image" src="https://github.com/user-attachments/assets/d47f6ec9-cc39-48ed-9731-af58b04c9b6d" />
<img width="2159" height="694" alt="image" src="https://github.com/user-attachments/assets/4e0a0a05-5b87-4045-9a78-addaa0c0807d" />
**NICE**

### White-Box Root Cause
A server block can define two locations to act as the root of the same backend upstream, in this case, whether you send a request to `/` or `/en-us`, they both land exacly at the same route (function) at the application level.
```nginx
location / {
  proxy_pass http://flask:7000;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection 'upgrade';
  proxy_cache_bypass $http_upgrade;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

# Alternative root, with http 1.0, and connetion "close"
location /en-us/ {
  proxy_pass http://flask:7000/; # trailing slash is added so the request to upstream goes to / and not append /en-us
  proxy_http_version 1.0;
  proxy_set_header Connection 'close';
}

location ~ ^/(secret|private|topsecret) {
  deny all;
}
```
Per-path deny rules provide no real coverage once any other location the same backend's full route surface; the entire model of individually blocking known sensitive paths is defeated when defining an alternate entry point.

**Impact:** Access controls bypass via alternate entry point.

**Remidiation:**
Deny-list at the alternate root's location block too:
```nginx
location /en-us/ {
  location ~ ^/en-us/(secret|private|topsecret|admin) { deny all; }
  proxy_pass http://flask:7000/;
}
```

## 2. Path Traversal / LFI via Root proxy_pass Without Upstream URI
### Black-Box Discovery
While fuzzing, or by viewing HTML page source code you get a `200 OK` response for the request `GET /assets/README.txt`, or `GET /assets/cat.jpeg`. These types of directories that server static files are the best candidate to test for LFI and path traversal. The detection goes like this:

1. You find a valid URL that serves a static file, now there is potential for information disclosure by replacing the correct file with a random file, like so `GET /assets/anything`, this might return an unintended response that leaks the current directory being used to host and serve the static files (the error depends on the application stack used in the backend, in our case the response is coming from a Flask application).
<img width="1293" height="664" alt="image" src="https://github.com/user-attachments/assets/6ac9b150-0d93-4a0b-a588-549b45e7a4cc" />
<img width="1025" height="136" alt="image" src="https://github.com/user-attachments/assets/0a79b2ca-1ee6-47a0-b213-ad4d156d3dc5" />
This gives us a better vision of where we are, and an easy way to brute force for any potentially sensitive files under this dir if we want to go in that route.
2. Now, to the juicy stuff, fire up your favorite proxy. To traverse up out the static files dir, we use the good old `../`, and same as above, the backend application can give us more than we should get. As shown in the screenshot below, we see that Flask by default is verbose, we can easily tell if the dir exists or not, which makes it so easy for brute-forcing a valid dir.
<img width="1724" height="367" alt="image" src="https://github.com/user-attachments/assets/89694d91-a69e-47ab-9a23-83abab53ca39" />
<img width="1724" height="367" alt="image" src="https://github.com/user-attachments/assets/464154a8-bfa9-49ee-8d22-c253ac7936de" />
Notice the difference in response, the first tells us that there is no such resource (not a file not a dir) names `foo`, but the second says something else; `Is a directory...`, meaning Flask is trying to open and read a dir, which cannot be done. So, that's a valid dir!
3. Now, as a last step, we can fire Intruder or your favorite fuzzing tool to try and find files inside the `sensitive` directory.
<img width="1724" height="367" alt="image" src="https://github.com/user-attachments/assets/85bd7fe3-576c-4544-82f7-762ce1f0f3bb" />
4. Sometimes, there is no need to even look for a dir, the dir above the current dir can hold some important files, such as source code files.
<img width="1727" height="1157" alt="image" src="https://github.com/user-attachments/assets/9a0f7982-2df2-44d1-9d5b-39ee6c6b9d7d" />

**NICE**

**Impact:** Arbitrary file read of anything the Flask process's OS user can access

### White-Box Root Cause
Now you are thinking how is this very cliché bug even still relevant with all the security advancements we have made? A very innocent misconfiguration your sysadmin will push thinking it's too safe!
The important part to notice here is the missing **trailing slash** in `proxy_pass` below at `/`.
```nginx
location / {
  proxy_pass http://flask:7000;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection 'upgrade';
  proxy_cache_bypass $http_upgrade;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

location /assets {
  proxy_pass http://flask:7000/assets;
}
```
In such case, where location `/` has no upstream URI (i.e, a trailing slash at least), any request that is not matched to any location block, will go as it is (unormalized) into the `proxy_pass` at location `/`, So the flow would go like this:
```
GET /assets/../sensitive/file.txt

nginx: normalizes and gets /sensitive/file.txt

NGINX: there is not matching block for /sensitive

NGINX routes the uri as it is (unormalized) to location /

NGINX: constructs the upstream path => 
  http://flask:7000 + /assets/../sensitive/file.txt

Flask: the route /assets catches the request, and treats anything after /assets as a file name, leading to LFI
```
So the normalization is not happening at the proxy level, but at the upstream Flask application level. And that's due to the missing URI trailing slash at the root location. 

**Remidiation:**
We can fix all this by simply appending a trailing slash to the root location's upstream.
```nginx
  location / {
    proxy_pass http://flask:7000/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_cache_bypass $http_upgrade;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
```
<img width="1727" height="556" alt="image" src="https://github.com/user-attachments/assets/6b4f5d18-624c-4d28-b14c-57206194fd63" />
And always make sure that we don't trust the reverse proxy, and perform server-side validation at the app level regardless.

## 3. Authorizaion Bypass via Unvalidated Regex Capture in proxy_pass
### Black-Box Discovery
1. You notice an endpoint that appears to do generic backend routing, like the following:
  - `GET /matchall/status` => you get a status page
  - `GET /matchall/changelog` => you get a changelog page
2. We can imagine that the server is routing to whatever comes after `/matchall`, and maybe the upstream is a docker container name, for example `http://conainer_name:container_port`, then the above requests are routed to:
  - `GET /matchall/status` => `http://flask/status`
  - `GET /matchall/changelog` => `http://flask/changelog`
3. To detetct and verify this from a black-box perspective, let's first send a request to just `GET /matchall` or `GET /matchall/`
<img width="1727" height="724" alt="image" src="https://github.com/user-attachments/assets/f4038bad-4fdb-4d35-9165-7b836b3b78f1" />
Then, we send a request to `/status` instead of `/matchall/status` and `/changelog` instead of `/matchall/changelog`, and if we see the same exact responses, that confirms the bug.
<img width="2158" height="723" alt="image" src="https://github.com/user-attachments/assets/4fa101c4-59eb-42c9-a14a-86ecc89587a1" />
Indeed, we can conclude that the application is doing just that, taking whatever comes after `/matchall` and passing it to the upstream backend.
4. This means that we can control whatever the proxy requests from the upstream? Exactly, and let's verify by trying to access a path that we normally return `403`.
<img width="2158" height="294" alt="image" src="https://github.com/user-attachments/assets/a17b6152-fb52-4927-bd06-e9c7f24b6564" />
<img width="2158" height="696" alt="image" src="https://github.com/user-attachments/assets/5a3fc304-a4f8-4042-86cf-354e34d39889" />
**NICE**

**Impact:** Unauthorized access to protected resources.

### White-Box Root Cause
```nginx
location ~ /matchall/(.*) {
  proxy_pass http://flask:7000/$1;
  proxy_redirect off;
}
```
`$1` is the regex capture, everything after `/matchall` is the request is fully attacker-controlled, and it's directly concatenated onto the upstream uri.
Although there are already access controls to protect the privileged paths:
```nginx
location = /admin {
    deny all;
}

location = /admin/ {
    deny all;
}

location ~ ^/(secret|private|topsecret) {
  deny all;
}
```
But that's entirely bypassed and never reached by the client request. The client request only matches `location ~ /matchall(.*)` and never `location ~ ^/(secret|private|topsecret)` or any other location block intended to protect against unauthorized access.

**Remidiation:**
Add a new deny list to deny any requests trying to access the sensitive pages through the `/matchall` route.
```nginx
location ~ ^/matchall/(admin(?:/|$)|secret(?:/|$)|private(?:/|$)|topsecret(?:/|$)) {
    deny all;
}
```

## 4. IP Spoofing via Missing proxy_set_header Inheritance
### Black-Box Discovery
1. Doing fuzzing you notice that `/internal/debug` returns `403`
2. Add a forged [`X-Forwarded-For`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Forwarded-For) or `X-Real-IP` header claiming an internal address:
```
GET /internal/debug
Host: portal.skyblue.com:8080
X-Forwarded-For: 10.1.1.1
```
3. If this return `200`, this means one of the three:
   - IP checking is only done at the proxy level and the proxy blindly trusts the header
   - The reverse proxy is not passing the `X-Forwarded-For` header to the backend
   - The backend is accessing the wrong `X-Forwarded-For` value in the list

<img width="1726" height="648" alt="image" src="https://github.com/user-attachments/assets/cf9a8858-7b7f-4d03-b0d8-3b047284cd74" />
<img width="1726" height="648" alt="image" src="https://github.com/user-attachments/assets/d800fb88-d794-4bcc-9a1e-2c1d717f7cc4" />
**NICE**

**Impact:** Full bypass of an IP-based access control using a single trivially forged request header.

### White-Box Root Cause
This absolutely looks like a simple trusted spoofable header bug, but the precise mechanics are more specific and worth understanding.

```nginx
location /internal/debug {
  set $trusted 0;
  if ($http_x_forwarded_for ~* "^10\.") { set $trusted 1; }
  if ($trusted = 0) { return 403; }

  add_header Cache-Control "no-cache, no-store, max-age=0, must-revalidate" always;
  proxy_pass http://flask:7000;
}
```

The most important thing to keep in mind is that `proxy_set_header` directives do NOT inherit between sibling location blocks. In our case, the location `/` correctly sets `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` which appends the real client's IP to the `X-Forwarded-For` array. But, as this directive is living in `location /`, not at the server level, any other sibling location, specifically `location /internal/debug` will not inherit it.

Because `location /internal/debug` defines no `proxy_set_header` directive of its own, Nginx forwards the `X-Forwarded-For` list as it received it, without appending the real IP of whoever it received the request from.

Then, Flask's `ProxyFix(x_for=1)` trusts that the real client' IP is one hop from the right of the `X-Forwarded-For` list. And since Nginx never appended its own trustworthy value, the single value and rightmost first value is the value spoofed by the attacker, so Flask accepts it as `request.remote_addr`.

**With vulnerable configuration:**
```
Attacker sends:
X-Forwarded-For: 127.0.0.1

        ↓

/internal/debug has NO proxy_set_header
        ↓

Nginx forwards it unchanged:
X-Forwarded-For: 127.0.0.1

        ↓

Flask ProxyFix(x_for=1)

Header is split into:
["127.0.0.1"]

        ↓

Take 1 value from the right:
127.0.0.1

        ↓

request.remote_addr = 127.0.0.1
```

**With correct configuration:**
```
Attacker sends:
X-Forwarded-For: 127.0.0.1

        ↓

Nginx appends the real client IP:
X-Forwarded-For: 127.0.0.1, 192.168.1.50

        ↓

Flask ProxyFix(x_for=1)

Header is split into:
["127.0.0.1", "192.168.1.50"]

        ↓

Take 1 value from the right:
192.168.1.50

        ↓

request.remote_addr = 192.168.1.50
```

Another thing worth mentioning, is that doing a check against `$http_x_forwarded_for` at Nginx level is generally bad practice, because `$http_x_forwarded_for` is basically whatever value Nginx received from the client, which is trivially forged.

**Remidiation:**
Move the XFF handling to the server level so every location inherits it and never rely on XFF alone, pair with nginx's own realip module against a real trust boundry:
```nginx
set_real_ip_from 172.16.0.0/12; # XFF can only be set by whoever belongs to this range
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

## 5. Denial of Service via Unbounded Request Body
### Black-Box Discovery
You come across a file upload endpoint, among all the impacts you may aim for is DoS, which in case a misconfigured reverse proxy can be done through uploading extremely large data streams.
1. Upload a small file to get an idea of how the endpoint responds normally
```
POST /upload (X-Filename: file.txt, small body) -> 201 Created
```
2. Now, upload a very large body (multiple GB) and observe backend memory / disk usage while the request is in flight (you can use `docker compose stats flask` in our lab)
3. Through several such uploads concurrently. If backend memory spikes, with no server-side rejection of any size, the endpoint effectively has no upload limit, both on the proxy and application level.

**Impact:** A single unauthenticated bad actor can drive unbounded memory and/or disk consumption on the backend with trivial bandwidth investment.

### White-Box Root Cause
```nginx
location /upload {
  client_max_body_size 0;
  proxy_pass http://flask:7000/upload;
}
```
`client_max_body_size 0` explicitly means "no limit on body size". By default, Nginx sets the body at 1m, unless the directive is explicitly changed by the devs if they see that the 1m is not enough, and setting it to 0 to make things easy during development to bypass the `413 Request Entity Too Large` error, and then forgetting bringing it back down to a sane limit.

The backend can compound the issue if it's reading the entire body into memory in one call with no streaming or size check.
```py
@app.route("/upload", methods=["POST", "PUT"])
def upload():
data = request.get_data() # buffers the ENTIRE body in memory
with open(target_path, "wb") as f:
f.write(data)
```

**Remidiation:**
Don't set `client_max_body_size` to 0, and make sure to rate limit requests to the endpoint:
```nginx
location /upload {
  client_max_body_size 10m;
  client_body_timeout 10s;
  limit_req zone=upload_limit burst=5 nodelay;
  proxy_pass http://flask:7000/upload;
}
limit_req_zone $binary_remote_addr zone=upload_limit:10m rate=1r/s;
```

## 5. CORS Misconfiguration via Missing Regex Anchor
### Black-Box Discovery
You spot an endpoint that leaks sensitive/auth/pii data of the logged-in user, you can immediately think of CORS. Let's quickly setup the victim's side:
1. Victim logs in via `/admin/portal` with cred: `admin:skyblue321`
2. Victim visits a page you send him
Now, as the attacker, open another browser to test if you can access the sensitive page, this usually involves the following:
1. Host a simple HTML page that make a request to the sensitive endpoint, in our case it's `/account/session`
2. Open it from another browser where you are logged in as the victim, if your page can access `/account/session` with no CORS problems, that's a confirmed bug.

**Impact:** Access to sensitive pages, reading authenticated responses cross-origin; including full session/account data theft against any logged-in user lured into the attacker's malicious page.

### White-Box Root Cause
```nginx
location /account/session {
  if ($http_origin ~* ^https://([a-zA-Z0-9-]+\.)*skyblue\.com) {
      add_header 'Access-Control-Allow-Origin' "$http_origin" always;
      add_header 'Access-Control-Allow-Credentials' 'true' always;
  }

  proxy_pass http://flask:7000/account/session;
}
```
The root cause of the bug we described in the black-box section most of the time comes one reason; bad regex. In the above code, the pattern is anchored at the start `^` but has NO trailing `$`, meaning this regex only needs the prefix to line up, and here is nothing stopping additional characters from following `skyblue.com` in the matched string.

`https://portal.skyblue.com.attacker.tld` satisfies this regex because it starts with the required prefix; the alteration for subdomains is not the bug here, it's the missing end-anchor.

**Remidiation:**
Bound the entire origin string in the regex, not just the prefix:
```nginx
if ($http_origin ~* ^https://([a-zA-Z0-9-]+\.)?skyblue\.com$) {
  add_header 'Access-Control-Allow-Origin' "$http_origin" always;
  add_header 'Access-Control-Allow-Credentials' 'true' always;
}
```

## 6. Open Redirect via User-Registerable Cloud Storage Bucket Name
### Black-Box Discovery
1. You go through your Burp history, and you notice and endpoint that seems to be requesting images from S3:
```
GET /s3/ne-1/assets-1/logo.png
```
2. You send it it Repeater and send it, you notice that it's a `301` redirect to:
```
https://s3-ap-northeast-1.amazonaws.com/skyblue-assets-1/logo.png
```
3. You notice the `1` from our domain's original endpoint `http://portal.skyblue.com/s3/ne-1/assets-1/logo.png` is being reflected in the S3 bucket name at the amazonaws URL we are being redirected to
4. To confirm this, you sent the request:
```
GET /s3/ne-1/assets-111111111111/logo.png
```
If the response is a redirect to:
```
https://s3-ap-northeast-1.amazonaws.com/skyblue-assets111111111111/logo.png
```
Then, you got a confirmed bug and you now you should register the bucket `skyblue-assets111111111111` and host an HTML page there.
<img width="1727" height="688" alt="image" src="https://github.com/user-attachments/assets/ad1604c2-e5b7-48ab-83ad-dea4b67f3ed9" />

**Impact:** Phishing via a trusted-domain redirect chain, compounded by 30-minute cache poisoning of the redirect target for any visitor requesting the same crafted number.

### White-Box Root Cause
```nginx
location ~ ^/s3/ne-1/assets-([0-9]+)/([^s]+)$ {
    access_log off;
    proxy_cache static_cache;
    proxy_cache_valid 200 30m;
    proxy_cache_key $scheme$host$request_uri;
    add_header X-Cache-Status $upstream_cache_status always;
    return 301 https://s3-ap-northeast-1.amazonaws.com/skyblue-assets$1/$2;
}
```
`$1` which is a fully attacker-controllable string is spliced directly into the target bucket name. Anyone can register an S3 bucket named `skyblue-assets<N>` in the same region `ap-northeast-1`. The redirect is also cached for 30 minites; meaning a malicious bucket poisons the caches for every subsequent visitor requesting that N, indeprndent of whether the underlying route is later fixed.

**Note:** Same behavior is present on another endpoint that fetches static files from Google's GCS, so make sure to take a look at that to understand the structure of Google Cloud Buckets.

**Remidiation:**
Use a fixed, single bucket name with the identifier only in the path, never in the bucket name itself as anyone can takeover a bucket that is not yet registered and use it to host whatever they want.
```nginx
return 301 https://s3-ap-northeast-1.amazonaws.com/skyblue-assets/$1/$2;
```

