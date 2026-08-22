---
layout: post
title: "Cloud Storage Guide; What Matters For Bug Hunters"
date: 2026-08-19 08:00:00 +0100
categories: [Cache Deception, Web Securiy]
tags: [Cache Deception, Web Security, AppSec]
---

Hunting for cache deception does not always imply looking for pages that contain authenticated user's data, even unauthenticated app visitors can enter sensitive data in the website's input fields without being authenticated!

## 1. Finding Description
A cache deception is a bug that happens when a cache server is tricked to cache a response that contains sensitive data specific to a certain client and should not be cached and accessible to the public.

In the scenario of the bug I'm disclosing here, the client's email is the sensitive data being cached and leaked. When doing bug hunting or pentest, any information relating to an **identified** or **identifiable**
natural person is considered a personal data violation and should be flagged and reported.

**identified** -> information that directly and uniquely identifies a person

**identifiable** -> person can be identified using a combination of the data available and other data collected from other resources

## 2. Exploit Steps
1. Navigating the target, you notice a field in the footer where clients enter their email address to subscribe to the newsletter
2. Inject a query param in the URL, ending with a static extension, or other cache deception tricks, as follows:
  - `https://www.target.tld/en-us/fragrances?foo=bar.css`
  - `https://www.target.tld/en-us/fragrances;.jpeg`
  - `https://www.target.tld/en-us/fragrances#.js`
  - `https://www.target.tld/en-us/fragrances/.png`
  - `https://www.target.tld/en-us/fragrances.html`
3. Send the crafted link to a victim, when they enter their email in the newsletter field, the response will be cached
4. Visit the link you sent to the victim, notice that the newsletter field is populated with the email entered by the victim
<img width="1393" height="376" alt="6abe8c16-16e7-4952-8d96-73567d280858-Screenshot from 2024-08-09 02-38-33 (1)" src="https://github.com/user-attachments/assets/5230908a-48c5-42f4-8210-9cf1d41f23c3" />

## 3. The Caveat
While going through Burp history trying to find any pages being cached, the above endpoint actually returned the response header
```
Cache-Control: no-cache, no-store, must-revalidate
```
By just seeing that I thought nothing interesting here, but I decided to test it anyways and it turned out to be vulnerable. After inspection and communicating with the team, I came up to the realization
that their reverse proxy was caching by static file extensions, ignoring **origin** `Cache-Control`, and that's what caused the reverse proxy to cache the response even though the backend explicitly ordered it not to do so, and then the reverse proxy forwarded the header as is to the client, which has no effect, as the response was already cached.

## 4. Key Takeaways
Whenever you see a page that contains a data relating to current user whether authenticated or not, try cache deception on that page.

---

**HAPPY HACKING!**
