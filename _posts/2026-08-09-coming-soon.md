---
layout: post
title: "Damn Vulnerable NGINX Proxy Full Guide"
date: 2026-08-09 08:00:00 +0100
categories: [Web Securiy, Nginx, Proxy]
tags: [AppSec, Pentest, NGINX, Reverse Proxy]
---

This blog covers all misconfigurations present in the Damn Vulnerable Nginx Proxy, explaining detection methods, exploitation from a black-box perspective, then reading the respective configuration code that caused the misconfiguration/exploit, and concludes with a suggested fix.

**DISCLAIMER:** Zero AI was used to write this blog, appreciate it? I know, you are welcome ;)

## 1. Introduction
### 1.1 What is Nginx, and Why Does it Matter?
Nginx is the most widely deployed reverse proxy in the world, powering roughly a third of all websites. And, as pentesters, bug bounty hunters or even security analysts we almost always come across it used as a load balancer, reverse proxy, or a simple server when doing engagements. Also, if you do configuration security reviews, you will need to have a good grasp of the common misconfigurations as well as some novel techniques usually weaponized by adversaries to bypass what looks seemingly secure.

In a reverse-proxy role, Nginx sits in front of one or more backend applications, deciding which requests go where, applying access controls, caching responses, terminating TLS and rewriting URLs before they reach the application code behind it. This critical position is what makes Nginx misconfigurations so consequential. A bug in a backend application is usually confined to that application, however, a bug in the proxy level sitting in front of multiple application can easily and quickly enlarge the blast radius, often without the backend applications having any bugs of their own at all.

Nginx configurations are full of directives whose exact semantics are easy to misjudge, whether a trailing slash strips a path prefix, whether one location's directive is inherited by another or not, whether a regex capture group is safe. None of these mistakes look wrong at a glance or a quick skim through the configuration code.

### 1.2 Why a Deliberately Vulnerable Reverse-Proxy Lab
Most available hands-on web security training focuses on the application layer bugs, however, the reverse-proxy level is comparatively under-served; it's covered essentially in write-ups and reference wikis, but rarely as something a learner can actually, run, break and fix end-to-end. While diving into Nginx security this last month, I didn't want to reinvent the wheel, and looked for already implemented vulnerable Nginx proxies, but all I found was projects that didn't showcase novel techniques that reflect latest research done on Nginx security, and for the most part, those project presented only one or two misconfigurations in the most direct way possible, which is too good to be true.

Damn Vulnerable Nginx Proxy exists to close that gap. Every finding we discuss in the guide is reproducible against a real, running reverse proxy. And for the most learning outcome, several findings are not isolated bugs but chains taken directly from real world findings, previous and novel research.

This blog walks through almost all twenty documented findings twice; first from black-box perspective; what an external attacker with no idea on the configuration would try, and what they would observe, and then from a white-box perspective, explaining precisely which misused directive or line of code caused the bad behavior.

## 2. Lab Architecture
### 2.1 Component Diagram
The lab run three containers behind a single published port. An Nginx reverse proxy terminates all inbount traffic on port 8080 and routes to two independent Flask backends based on which of the three virtual hosts a request targets.

<img width="1598" height="984" alt="a0e809bb-9900-4751-b5f9-3e494abbf6ac" src="https://github.com/user-attachments/assets/d6c1c22c-acb9-41f6-9f68-5921234e8b30" />

### 2.2 Project Structure
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

## 3. Methodology
### 3.1 Black-Box vs White-Box Testing is this Guide
Every finding below is presented twice. The black-box section describes what a tester with no access to the configuration or source code would try, and what response would tip them off that something is wrong and worth digging down the rabbit hole. The white-box section then opens the configuration and application code and explains, directive by directive, exactly why the behavior happened.

Only going through the black-box is generally enough if you your work does not involve doing code review, however it's still a good skill to have as it builds the deeper skill; recognizing patters and imagining how the proxy is configured without having actual access to the config file.

## 4 Finding List: What We Will Go Through?
### 4.1 portal.skyblue.com
1. Access Control Bypass via a Permissive Alternate Root
2. Path Traversal / LFI via Root proxy_pass Without Upstream URI
3. Authorizaion Bypass via Unvalidated Regex Capture in proxy_pass
4. IP Spoofing via Missing proxy_set_header Inheritance
5. Authentication Bypass by Cookie Replay (Static Session Token via auth_request)
6. CORS Misconfiguration via Missing Regex Anchor
7. Denial of Service via Unbounded Request Body
8. Web Cache Deception via Extension-Based Cache Matching
9. Open Redirect via User-Registerable Cloud Storage Bucket Name
10. HTTP Response Splitting via Unsanitized Regex Capture
11. Default-Allow Gap in the map Directive
12. SSRF via Client-Controlled Host Header

### 4.2 sandbox-dev-001.skyblue.com
1. Location Match-Priority Bypass via ^~ Overriding a Regex deny Rule
2. Stored XSS via Unescaped Log Injection
3. Information Disclosure via Exposed stub_status

### 4.3 sandbox-dev-002.skyblue.com
1. Open Redirect via Missing Leading Slash in a Rewrite Capture Group
2. Access Control Inconsistency - auth_basic Not Inherited by a More-Specific Location
3. Credential Brute-Force via Missing Rate Limiting on Basic Auth
4. Authentication Bypass via satisfy any and a Broken IP Trust Boundary

---

## Findings: portal.skyblue.com
This is the main domain (set via the `default_server` directive in the nginx confi file), and it's what we as pentesters or bug bounty hunters don't usually spend much time navigating through, it's a boring status page!
<img width="2147" height="1389" alt="image" src="https://github.com/user-attachments/assets/6d45a19e-46c8-466f-95b9-ae1b8e3fab96" />

### 1. Authorization Bypass via a Permissive Alternate Root
#### Black-Box Discovery
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
GET /en-us/changelog
```
3. This prefix can be your friend in bypassing those `403`s, just append it to them too!
<img width="2159" height="328" alt="image" src="https://github.com/user-attachments/assets/d47f6ec9-cc39-48ed-9731-af58b04c9b6d" />
<img width="2159" height="694" alt="image" src="https://github.com/user-attachments/assets/4e0a0a05-5b87-4045-9a78-addaa0c0807d" />
**NICE**

**Impact:** Access controls bypass via alternate entry point.

#### White-Box Root Cause
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

**Remidiation:**
Deny-list at the alternate root's location block too:
```nginx
location /en-us/ {
  location ~ ^/en-us/(secret|private|topsecret|admin) { deny all; }
  proxy_pass http://flask:7000/;
}
```

### 2. Broken Access Control via Trim Inconsistencies
#### Black-Box Discovery
You did your fuzzing, and found a some paths that return `403`, if the bypass in finding #1 did not help you bypass the `403`, a misconfigured Nginx proxy can offer you another change to conduct a bypass.
1. You want to bypass the 403 given by `/admin`
2. Try to find any clues that leak which framework is running the target backend application (Flask, NodeJS, SpringBoot, Laravel...)
3. Fingerprinting the backend framework is good, but it's not mandatory, it just help you focus of the bypasses available for that exact framework, in our case the backend is application running Flask (you can determine this using many techniques, the one I usually use is find an endpoint that the app uses to serve me a file's content, and replace that with a file that does not exist and watch the response to the GET request). Find at the end of this section a table containing all possible bypass characters
4. If you know the Nginx version used use a respective character, if you don't you can try all of them (in our lab, the proxy is running `nginx/1.31.3`). To conduct the bypass, make sure to capture the request in Burp
<img width="1723" height="367" alt="image" src="https://github.com/user-attachments/assets/849a2808-2ce6-4352-aa60-52cb63a353be" />
Now, add a `/` after `/admin`, and select the slash and from **Inspector** on the right, change its code to `85`
<img width="2146" height="370" alt="Screenshot from 2026-08-11 08-42-53" src="https://github.com/user-attachments/assets/1e0de642-dc78-4ad5-b22b-29331b200a0a" />
<img width="2146" height="382" alt="Screenshot from 2026-08-11 08-44-24" src="https://github.com/user-attachments/assets/22de283d-3330-4b81-9aaa-03ef546c12c6" />
Send the request and notice that we bypassed the `403`, we get "unauthenticated" and that's coming from the Flask backend, so, we've already went past Nginx, and that's what we care about here.
<img width="1721" height="382" alt="image" src="https://github.com/user-attachments/assets/de8d7b0a-d6d4-4923-800c-1d8f3139439d" />
**NICE**

**Impact:** Access controls bypass and access to sensitive privileged paths.

#### White-Box Root Cause
In short, this technique works because there is a discrepancy between the Nginx proxy and the Flask backend, where Nginx would essentially keep certain character in the uri, while the backend app would remove those characters, which in many cases leads to broken access conrols.

This is the vulnerable code in our configuration:
```nginx
location = /admin {
    deny all;
}

location = /admin/ {
    deny all;
}
```
`=` is basically an exact match, so the request uri has to be EXACLY `/admin` nothing more nothing else. Any additional character will make the request be matched by the `/` location block, which routes whatever it gets as it is to the backend. so appending `\x85` after `/admin`, or `/admin/` will not be matched against neither of the above two location blocks, and would instead be matched against:
```nginx
location / {
  proxy_pass http://flask:7000;
}
```
And you can see that it sends everything in the uri to the backend, meaning `/admin\x85` will go to `http://flask:7000/admin\x85`, as Nginx doesn't trim `\x85` and a few other characters. Then, when Flask receives the request, it automatically trims the `\x85`, which means the request will be routed to `/admin` at the application level.

**Remidiation:**
Avoid using exact location matching:
```nginx
location /admin {
    deny all;
}
```

**Bypass codes for other backend frameworks:**

| Nginx Version | Flask Bypass Characters |
|---|---|
| 1.22.0 | `\x85`, `\xA0` |
| 1.21.6 | `\x85`, `\xA0` |
| 1.20.2 | `\x85`, `\xA0`, `\x1F`, `\x1E`, `\x1D`, `\x1C`, `\x0C`, `\x0B` |
| 1.18.0 | `\x85`, `\xA0`, `\x1F`, `\x1E`, `\x1D`, `\x1C`, `\x0C`, `\x0B` |
| 1.16.1 | `\x85`, `\xA0`, `\x1F`, `\x1E`, `\x1D`, `\x1C`, `\x0C`, `\x0B` |

| Nginx Version | Node.js Bypass Characters |
|---|---|
| 1.22.0 | `\xA0` |
| 1.21.6 | `\xA0` |
| 1.20.2 | `\xA0`, `\x09`, `\x0C` |
| 1.18.0 | `\xA0`, `\x09`, `\x0C` |
| 1.16.1 | `\xA0`, `\x09`, `\x0C` |

| Nginx Version | Spring Boot Bypass Characters |
|---|---|
| 1.22.0 | `;` |
| 1.21.6 | `;` |
| 1.20.2 | `\x09`, `;` |
| 1.18.0 | `\x09`, `;` |
| 1.16.1 | `\x09`, `;` |

### 3. HTTP Splitting via Unsanitized Regex Capture Leads to Open Redirect
#### Black-Box Discovery
1. You identify an endpoint that seems to be fetching static assets from a bucket (S3, GCS...), immediately think of HTTP splitting
```
GET /s3/static/js/admin.main.js
```
2. Think of injecting a CRLF payload after `/s3/static/` or after `/s3/static/js/`, with a host header containing the name of a bucket you own:
```
GET /s3/static/phish.html%20HTTP/1.1%0d%0aHost:evilbucket%0d%0a%0d%0a
```

This will cause the proxy to make this request to S3:
```
GET /original-bucket/phish.html
Host: evilbucket

anything_else HTTP/1.1
Host: original-bucket
```

Which will redirect `/original-bucket/phish.html` to your bucket, so all you have to do is host `phish.html` inside a directory literally named as the name of the original bucket the proxy was requesting.
<img width="1721" height="954" alt="Screenshot from 2026-08-11 08-47-14" src="https://github.com/user-attachments/assets/dbd53302-55e6-4eae-aaf6-0b1437aa9824" />

**Impact:** Redirect to malicious web pages, hosted on cloud storage buckets.

#### White-Box Root Cause

```nginx
location ~ /s3/static/([^/]*/[^/]*)? {
  access_log off;
  proxy_pass https://s3-eu-west-1.amazonaws.com/skyblue-prod/$1;
}
```
`[^/]` excludes only the slash character; it does not exclude spaces, carriage returns, or line feeds. A request containing a url-encoded CRLF sequence (`%0d%0a`) is decoded by Nginx into raw conrol bytes before the capture happens, so `$1` can end up containing a complete, attacker-chosen extra header line or even a second request line once spliced into the outbound request nginx builds for the upstream.

**Remediation:**
Exclude whitespace/control characters explicitly from the capture:
```nginx
location ~ /s3/static/([^/\s]*/[^/\s]*)? {
  access_log off;
  proxy_pass https://s3-eu-west-1.amazonaws.com/skyblue-prod/$1;
}
```
`.*` is also safe here, since `.` never matches `\n` by default in PCRE.

### 4. Path Traversal / LFI via Root proxy_pass Without Upstream URI
#### Black-Box Discovery
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
4. Sometimes, there is no need to even look for a dir, the dir above the current dir can hold some important files, sus3/ne-1/assets-111111111111/logo.pngch as source code files.
<img width="1727" height="1157" alt="image" src="https://github.com/user-attachments/assets/9a0f7982-2df2-44d1-9d5b-39ee6c6b9d7d" />

**NICE**

**Impact:** Arbitrary file read of anything the Flask process's OS user can access

#### White-Box Root Cause
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

### 5. Authorizaion Bypass via Unvalidated Regex Capture in proxy_pass
#### Black-Box Discovery
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

#### White-Box Root Cause
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

### 6. IP Spoofing via Missing proxy_set_header Inheritance
#### Black-Box Discovery
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

#### White-Box Root Cause
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

### 7. Denial of Service via Unbounded Request Body
#### Black-Box Discovery
You come across a file upload endpoint, among all the impacts you may aim for is DoS, which in case a misconfigured reverse proxy can be done through uploading extremely large data streams.
1. Upload a small file to get an idea of how the endpoint responds normally
```
POST /upload (X-Filename: file.txt, small body) -> 201 Created
```
2. Now, upload a very large body (multiple GB) and observe backend memory / disk usage while the request is in flight (you can use `docker compose stats flask` in our lab)
3. Through several such uploads concurrently. If backend memory spikes, with no server-side rejection of any size, the endpoint effectively has no upload limit, both on the proxy and application level.

**Impact:** A single unauthenticated bad actor can drive unbounded memory and/or disk consumption on the backend with trivial bandwidth investment.

#### White-Box Root Cause
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

### 8. CORS Misconfiguration via Missing Regex Anchor
#### Black-Box Discovery
You spot an endpoint that leaks sensitive/auth/pii data of the logged-in user, you can immediately think of CORS. Let's quickly setup the victim's side:
1. Victim logs in via `/admin/portal` with cred: `admin:skyblue321`
2. Victim visits a page you send him
Now, as the attacker, open another browser to test if you can access the sensitive page, this usually involves the following:
1. Host a simple HTML page that make a request to the sensitive endpoint, in our case it's `/account/session`
2. Open it from another browser where you are logged in as the victim, if your page can access `/account/session` with no CORS problems, that's a confirmed bug.

**Impact:** Access to sensitive pages, reading authenticated responses cross-origin; including full session/account data theft against any logged-in user lured into the attacker's malicious page.

#### White-Box Root Cause
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

### 9. Open Redirect via User-Registerable Cloud Storage Bucket Name
#### Black-Box Discovery
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
<img width="1721" height="682" alt="Screenshot from 2026-08-11 08-50-47" src="https://github.com/user-attachments/assets/3ed4ab80-b4bf-4fd6-aec5-f2a25b7ecb91" />

**Impact:** Phishing via a trusted-domain redirect chain, compounded by 30-minute cache poisoning of the redirect target for any visitor requesting the same crafted number.

#### White-Box Root Cause
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

### 10. Access Control Bypass due to Default-Allow Gap in the map Directive and merge_slashes=off
#### Black-Box Discovery
1. You notice a set of endpoints that all share a prefix, and they seem to be privileged, and return `403`:
```
GET /map-poc/private -> 403
GET /map-poc/secret -> 403
GET /map-poc/topsecret -> 403
```
2. Think of a map directive is used in the proxy configuration, and these maps are usually bypassable if two conditions are present:
  - the `map` directive misses a default entry
  - `merge_slashes` is disabled at the server
3. You can easily test that by only adding extra slashes:
```
GET GET /map-poc/////private
```

<img width="1721" height="810" alt="image" src="https://github.com/user-attachments/assets/1ba9c085-181c-49cf-962c-3c526a3c9259" />
<img width="1723" height="693" alt="image" src="https://github.com/user-attachments/assets/c14adb5b-57b4-4e8d-8c94-1cafa6d09629" />

Something worth noting, by sending a request to a random value after the shared prefix, like `GET /map-poc/foobar` you can tell if the proxy's `map` has a default entry or not. If it gives `403` for any random value after the prefix, this ascetain you that there is a default entry that rejects any value not present in the map, however, if it gives you `404` this means there is a chance that the map is missing a default entry. We will see this in more detail just below.

<img width="1721" height="810" alt="image" src="https://github.com/user-attachments/assets/1ba9c085-181c-49cf-962c-3c526a3c9259" />
<img width="1723" height="364" alt="image" src="https://github.com/user-attachments/assets/c0452330-83b5-475c-a742-d7a9fff50fae" />
*404 Error returned by an NGINX custom page*

#### White-Box Root Cause
```nginx
map $uri $mappocallow {
  /map-poc/private   0;
  /map-poc/secret     0;
  /map-poc/public     1;
}

server {
  merge_slashes off;

  location /map-poc/ {
    if ($mappocallow = 0) {return 403;}
  
    proxy_pass http://flask:7000/;
  }
}
```

Without a default entry, the `map` directive returns an EMPTY STRING for any `$uri` that does not match a listed key; not `0`, and not an error, but just a literal `""`. The check `if ($mappocallow = 0)` only blocks the value `0` specifically; an empty string does not satisfy that comparison, so any unlisted URI silently passes through unblocked. Compounding this, `merge_slashes off` means `/map-poc//private` with two or more slashes also does not match the literal key `/map-poc/private`, producing the empty-string bypass.

Another thing to mention here, is that a missing default value can cause bugs even if `merge_slashes` was activated, in case developers add a sensitive path in the backend application like `/map-poc/vip` but forget to add it to the `map` at the proxy level thinking it's properly gated like the others in the map.

**Remidiation:**
Add a default entry to be the default value for anything not matching a key in the map:
```nginx
map $uri $mappocallow {
  default 0;
  /map-poc/private   0;
  /map-poc/secret     0;
  /map-poc/public     1;
}
```

### 11. Cache Poisoning via Client-Controlled X-Forwarded-Host Header and Unkeyed Input
#### Black-Box Discovery
1. You are going through the web application and notice a page that seems to be reflecting the same host it's currently on:
<img width="2161" height="1136" alt="untitled (1)" src="https://github.com/user-attachments/assets/f7167d00-bfc3-4cdb-8fd9-c0eadedb327a" />
2. Quickly think of cache poisoning, and make sure the endpoint is being cached (hopefully by a misconfigured reverse proxy)
<img width="1727" height="477" alt="image" src="https://github.com/user-attachments/assets/368ed271-bc74-4fbe-add6-798a81cf7c29" />
Now we are sure that the page is being cached, now we gotta see whether the host we saw earlier in the page is static or dynamically reflecting a request header.
3. We try injecting a random value in `Host` then `X-Forwarded-Host`, we notice that both reflect, but what really matters is which one is not present in the cache key? hat's the one we used to conduct the attack
<img width="1727" height="1103" alt="image" src="https://github.com/user-attachments/assets/6c784638-0cbc-4937-ba64-803677487a16" />
<img width="1727" height="1103" alt="image" src="https://github.com/user-attachments/assets/acf968f6-96bc-470e-aa34-d48422a9feae" />
We should tell which one is missing from the cache key, but from a black-box perspective that's a matter of trial and error, so play around with both and see which one is not being used as a cache key, and make sure to used a cache buster while trying. However, always keep in mind that sometimes both act as they are the same header, meaning they reach the backend app with the same value **(host==x_forwarded_for)** when there is only one proxy **(client -> proxy -> server)**, and when there are multiple proxies in the line **(client -> proxy_1 -> proxy_2 -> server)** then they will not hold the same value, so it's always trying to poison both.
4. Now to exploit, we inject a malicious value at `X-Forwarded-For`, set a cache buster, and send the request potentially multiple times until we see response header `X-Cache-Status: HIT`
<img width="1727" height="1163" alt="image" src="https://github.com/user-attachments/assets/d55c0b51-7ec9-4a44-b5d1-2c3a3c79b9ea" />
<img width="2140" height="1260" alt="image" src="https://github.com/user-attachments/assets/d151a6dc-b6fc-42da-8340-8f9d57cf8459" />
**NICE**

#### White-Box Root Cause
```nginx
location /preview-link {
  proxy_pass http://flask:7000/preview-link;
  proxy_cache static_cache;
  proxy_cache_valid 200 10m;
  proxy_cache_key $scheme$host$request_uri; <-- vulnerable
  add_header X-Cache-Status $upstream_cache_status always;
}
```
We see that only the scheme, host and the request uri (path and query params) are used as cache keys, but `X-Forwarded-Host` is missing, meaning the victim only needs to provide the same scheme, host and uri and he will receive the cached malicious response.

**Remidiation:**
Add `X-Forwarded-For` to the cache key:
```nginx
location /preview-link {
    proxy_pass http://flask:7000/preview-link;ca
    proxy_cache static_cache;
    proxy_cache_valid 200 10m;
    proxy_cache_key $scheme$host$request_uri$http_x_forwarded_host;
    add_header X-Cache-Status $upstream_cache_status always;
}
```

### 12. Web Cache Deception via Extension-Based Cache Matching
#### Black-Box Discovery
1. Fuzzing, you found an authenticated endpoint that seems to be intended to return logged-in admin session data (`/account/session`) which returns `401 UNAUTHORIZED`, as a hacker you need to quickly think of possible cache deception. The easiest way to tell this is by one or more of the following tactics:
   - Immediately append a static file extension to the endpoint: `/account/session.css`
       - If the application returns the same response as for `/account/session`, with a cache header, then it's a confirmed bug
       - If you get `404`, check the next technique
   - add a path and watch if the backend application seems to accept it and returns the same response: `/account/session/foo`
       - In this case, proceed to add a static extension, and watch if the response includes a cache header: `/account/session/foo.css` --> 401 with header `X-Cache-Status`
       - Even if the value of `X-Cache-Status` is `MISS` in the above test, that doesn't mean that the endpoint will not get cached, what matters in the detection phase is that there is a cache header in the response, and the caching period is long enough (however in case of a very sensitive endpoint like he one we are testing, even caching for a few minutes is very impactful)
    - Another technique is appending a query param that ends with a static extension: `/account/session?foo=bar.js`
<img width="1733" height="363" alt="image" src="https://github.com/user-attachments/assets/45c0323d-3fbb-4825-babb-80621ffcadc6" />
2. To exploit, send the crafted URL to a privileged admin, and periodically check the same URL from your own browser so that when the victim visits it and it gets cached you will access the admin's sensitive session data

**PS:** to login as the victim admin, there is a specific endpoint that can be found through fuzzing

**Impact:** sensitive per-user data becomes retrievable by any anonymous visitor as long as the cache key entry lives, triggered simple by luring an authenticated user to a crafted link.

#### White-Box Root Cause
```nginx
location ~* \.(css|js|html|json)$ {
  access_log off;
  proxy_cache static_cache;
  proxy_cache_valid 200 60m;
  # note: no cookie/session in the key
  proxy_cache_key $scheme$host$request_uri;
  add_header X-Cache-Status $upstream_cache_status always;
  proxy_pass http://flask:7000;
}
```
There are two misconfiguration in the above configurations that made the exploit posible:
  - The location matches ANY request path ending in one of those extensions
  - The cache key is built purely from the URL (scheme + host + URI) with no cookie/session component

And it's worth noting that the technique we showed in the black-box discovery section, where I said that you can use a query param with a static extension, that's doable in certain other proxies, however in Nginx that will not help as Nginx strips off query params when matching, so `/account/seesion?foo=bar.png` will not even match the above location block.

**Remidiation:**
1. Anchor a static-asset prefix instead of leaving the choice to users as long as the request ends with a static extension
  ```nginx
  location ~* ^/static/.*\.(css|js|html|json)$ {
      access_log off;
      proxy_cache static_cache;
      proxy_cache_valid 200 60m;
      proxy_cache_key $scheme$host$request_uri;
      add_header X-Cache-Status $upstream_cache_status always;
      proxy_pass http://flask:7000;
  }
  ```
  Adding `^/static` means a request has to genuinely live under the static-asset path to match at all, since the backend app's `/account/session/<path:extra>` route does not live under `/static`, no fake extension trick can make the request satisfy this location anymore
2. Don't cache responses that vary by identity
  ```nginx
  location ~* ^/static/.*\.(css|js|html|json)$ {
      access_log off;
  
      # Never cache a response for a request that carried authentication
      proxy_no_cache $http_cookie $http_authorization;
      proxy_cache_bypass $http_cookie $http_authorization;
  
      proxy_cache static_cache;
      proxy_cache_valid 200 60m;
      proxy_cache_key $scheme$host$request_uri;
      add_header X-Cache-Status $upstream_cache_status always;
      proxy_pass http://flask:7000;
  }
  ```
  - `proxy_no_cache`: if the request carries a cookie or auth header, never trigger a cache
  - `proxy_cache_bypass`: if the request carries a cookie or auth header, never read from the cache, always go to backend fresh

Together these mean that even a genuinely static-looking URL under /static can never be poisoned by an authenticated user (victim in case of attack), and an authenticated user can never receive someone else's cached data

---

## Findings: sandbox-dev-001.skyblue.com
This vhost presents just the default Nginx index page, and we need to go from just that to real hacker mode! No buttons, no fancy UI, just like we see everyday everywhere
<img width="2162" height="484" alt="image" src="https://github.com/user-attachments/assets/14baa0c9-96a6-4fc1-9960-32b929fa4b6e" />

### 1. Location Match-Priority Bypass via ^~ Overriding a Regex deny Rule
#### Black-Box Discovery
1. While doing your usual fuzzing, you found two entries that return an interesting `403`
  - A deploy script: `/deploy.sh`
  - A static assets dir: `/maintenance/` (notice that it's `/maintenance/` and not `/maintenance`, those two are not always the same in Nginx, so make sure to fuzz with and without an appended `/` for all the words in your wordlist)
2. Notice how they seem related, what if the deploy script is under the maintenance dir, but the proxy is applying deny rules in the wrong order? To figure that out:
```
GET /maintenance/deploy.sh --> 200 OK
```
**NICE**

The above behavior is more common that you may think, sysadmin may think that when he denies any request that ends with a sensitive extension that's enough and secure, but it's not in most cases

**Impact:** Any backup, script, or config file placed under the exempted prefix bypasses the deny rule entirely, despite the rule appearing correct and secure in isolation.

#### White-Box Root Cause
```nginx
# unsafe use of ^~
location ^~ /maintenance/ {
  root /var/www/dev;
}

# Intended to block backup/config file extensions everywhere on this vhost
location ~* ^/.*\.(bak|old|swp|orig|sh|sql)$ {
  deny all;
}
```

Nginx's location-selection algorithm finds the longest matching PREFIX first, and if that prefix was declared with the `^~` modifier, Nginx stops and never evaluates any regex location at all for that request.

`^~` is often used for performance optimization, without realizing it also skips security-relevant regex deny rules declared elsewhere in the same server block.

**Remidiation:**
Repeat the extension check inside the prefix location too
```nginx
location ^~ /maintenance/ {
  root /var/www/dev;
  location ~* \.(bak|old|swp|orig|sh|sql)$ { deny all; }
}
```

### 2. Blind Stored XSS via Unescaped Log Injection
#### Black-Box Discovery
It's always worth injection XSS payloads in different headers in all different subdomains of your target, especially the User Agent, as it usually get logged, and if there a web application used to view the logs, there is a high possibility the XSS will go though as those web apps tend to be quickly set up and don't go through security testing as they are only intended for internal use.

1. Send a request with a crafted User-Agent containing an XSS payload (notice that we are sending the request to `portal.skyblue.com`, as the logs view web page can be used to only view the logs of the main web app and not the dev/internal hosts)
```
GET / HTTP/1.1
Host: portal.skyblue.com:8080
User-Agent: <script>alert(document.domain)</script>
```
2. Victim visits the logs view page (`http://sandbox-dev-001.skyblue.com/logs` and authenticates `admin:skyblue321`), the malicious log executes the injected Javascript
<img width="2162" height="591" alt="image" src="https://github.com/user-attachments/assets/ac5850b6-b86d-48dc-9027-5f96bc9e8a0e" />
**NICE**

#### White-Box Root Cause
```nginx
log_format portal_logs '$remote_addr - $http_user_agent - $request_uri';

server {
  server_name portal.skyblue.com;
  access_log /var/log/nginx/access_portal.log portal_logs;
}
```

None of the variables used in `log_format` are sanitized before being written to the log file. For example, `$http_user_agent` is entirely client-controlled and can any set of characters the client/attacker chooses, including but not limited to XSS payloads.

In addition to this, the logs viewer Flask app in the backend reads the logs file and renders each line using Jinja2's `| safe` filter:
{% raw %}
```jinja2
{% for line in lines %}{{ line | safe }}{% endfor %}
```

`| safe` deliberately disables Jinja's default auto-escaping. So, anything injected into the logs including a full `<script>` tag renders into the page exactly as stored, with not encoding.

**Remidiation:**
Making every current or future log viewer web page secure is not an easy task, unsanitazied logs should never be written to the logs files at all, or at least cleaned before they are written. Imagine that we fixed the current log viewer web page, and after a year, new devs come in and they decide they don't like the UI, so they vibe code their own quickly, in this case, if the new page does not sanitize the inputs from the log file then we are back to zero, that's why it's always better to do it at the proxy level as well as the application level.

We can add these maps to the `http` block:
```nginx
map $http_user_agent $safe_user_agent {
    default                    $http_user_agent;
    "~[<>\"'&]"                "[user-agent contained invalid characters]";
}

map $request_uri $safe_request_uri {
    default                    $request_uri;
    "~[<>\"'&]"                "[uri contained invalid characters]";
}
```
This will replace the user-agent and request uri (path+query params) with static values if it detects an HTML-significant character in them.

Also make sure to enable auto-escaping in the backend application, in out Flask case, we need to remove `| safe`:
{% raw %}
```jinja2
{% for line in lines %}{{ line }}{% endfor %}
```

### 3. Information Disclosure via Exposed stub_status
#### Black-Box Discovery
When coming across an Nginx proxy, it's always good to hunt for exposed Nginx metrics at the proxy level. This in its own is a low hanging low severity issue, but, it becomes a useful recon signal when conducting DoS/DDoS against the proxy to confirm whether the attempts are actually driving connection counts up and draining the resources.

1. Craft your own special wordlist to fuzz for this endpoint, combining words like `nginx`, `status`, `stub`, `metrics`
2. We did just that and found it at:
```
GET /nginx_status
Host: portal.skyblue.com:8080
```
<img width="1588" height="213" alt="image" src="https://github.com/user-attachments/assets/7c729eb5-905d-45bb-aedd-d592d1fcce03" />

#### White-Box Root Cause
```nginx
location /nginx_status {
  stub_status;
}
```

This endpoint which uses the `stub_status` directive to reports statistics on server usage, including current number of requests being handled, total number of handled requests, number of requests waiting to be handled... is exposed to the public with no rules.

**Remidiation:**
Deny external access:
```nginx
location /nginx_status {
  stub_status;
  allow 127.0.0.1;
  deny all;
}
```

---

## Findings: sandbox-dev-002.skyblue.com
This is another vhost that seems to offer no UI, and aimed for development and staging, let's see what we can uncover!

### 1. Open Redirect via Missing Leading Slash in a Rewrite Capture Group
#### Black-Box Discovery
1. We visit the bare root of the vhsot `http://sandbox-dev-002.skyblue.com/`, we notice that we are redirected to a third party domain
2. Some misconfigured load balancers/reverse proxy implement the redirect in a vulnerable way, we can test that by appending anything after `/` like `http://sandbox-dev-002.skyblue.com/evil.com`
3. We get redirected to `https://skyblue.comevil.com/`! That's a random domain, we can easily register it and we have a proved open redirect

Notice it's really to miss this finding, if all you do is take the vhost and give to **ffuf** and filter match for `200` and `403` responses, as the status code in this case would be `301`, so always keep that in mind, as this finding is a documented misconfigured redirect behavior found in the wild.

#### White-Box Root Cause
```nginx
location / {
  proxy_ssl_verify on;
  rewrite ^/(.*)$ https://skyblue.com$1 permanent;
}
```

The regex `^/(.*)$` feels safe as it matches the leading slash, but he slash sits OUTSIDE the capturing parentheses; so it's consumed by the match bu never included in `$1`. So, for a request to `http://sandbox-dev-002.skyblue.com/evil.com` , `$1` becomes `evil.com`. Concatenating `https://skyblue.com` + `evil.com` with no separator; which is a syntactically valid, registerable domain name an attacker can own and abuse..

**Remidiation:**
Move `/` inside the capture group:
```nginx
location / {
  rewrite ^(/.*)$ https://skyblue.com$1 permanent;
}
```
