---
title: Subdomain takeover by taking advantage of Wordpress
date: '2021-11-08'
tags: ['403 bypass', 'bug bounty']
draft: true
images: ['static/images/403-bypass/forbidden.png', 'static/images/403-bypass/intercept-1.png','static/images/403-bypass/intercept-2.png', 'static/images/403-bypass/login.png']
summary: How I was able to get 
---

Blogging has never been a habit I was able to maintain for long but every once in a while I have the urge to write something. Also, as I still benefit so much from other's peoples writeups, it almost feels like an obligation to give it back somehow. Anyways, this is me once again, attempting to stick with a habit.

## Overview

A couple months ago I was zapping through a few bug bounty programs when I found something that looked interesting. I had some spare time, so I quickly decided to put down the work and give it a go, just to see whether I could find anything relevant in it.
Before effectively starting out, I often just pipe together a bunch of tools to give me an initial understanding of what I'm dealing with. Then, I can go about my day and check it later once its done. This was no different.

__Quick heads up__: As I won't be disclosing what program it was, I will be using redacted.com instead just like the cool kids do. Moving on.

## Recon

[ffuf](https://github.com/ffuf/ffuf) was one of the tools I left collecting results and as I was brute forcing my way through subdomains — previously, [httpx](https://github.com/projectdiscovery/httpx) pointed out this one host was a [vhost](https://en.wikipedia.org/wiki/Virtual_hosting) — a few interesting things popped out as a result: 

```bash
$ ffuf -u https://redacted.com -w /share/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -H "Host: FUZZ.redacted.com" -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36" -fw 215

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.0-rc1
________________________________________________

 :: Method           : GET
 :: URL              : https://redacted.com
 :: Header           : Host: FUZZ.redacted.com
 :: Header           : User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
 :: Filter           : Response words: 215
________________________________________________

www                     [Status: 200, Size: 101319, Words: 12146, Lines: 1073]
de                      [Status: 200, Size: 101990, Words: 11958, Lines: 1057]
it                      [Status: 200, Size: 226558, Words: 14595, Lines: 1061]
es                      [Status: 302, Size: 0, Words: 1, Lines: 1]
payment                 [Status: 200, Size: 15, Words: 2, Lines: 1]
statistics              [Status: 403, Size: 548, Words: 69, Lines: 14]

...truncated 
```
I noticed a subdomain named as __statistics__ was giving back a [403 status](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/403). Bummer. I swiftly fired up the browser and just tried it out but as expected, still got the same thing: 

![forbidden.png](/static/images/403-bypass/forbidden.png)

Usually, I tend to poke around with 403's when I don't have much left but for whatever reason I felt inclined to give it a shot.

## Bypassing

I maintain a list of resources when it comes to 403 bypasses which I regularly revisit to add new things or just to make sure I'm not missing anything. If you spent some time around this topic, you probably already came accross things like the header [X-Forwarded-Host](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Host) and a bunch of variations of it. Those techniques are effective depending on which context are being used and this makes a whole lot of difference when you are trying to bypass 403/401's. Why? 

Let me explain: The same way you wouldn't throw a `<img src=x onerror="alert(1)">` onto javascript context aiming for a [XSS](https://owasp.org/www-community/attacks/xss/), `X-Forwarded-*` is unlikely to work if there's no proxy or load balancer sitting in between the client and the application as far as I know. Whether this a matter of saving time or just trying to get lucky, blindly catapult payloads against a target usually don't do much in my experience - or if it does, it is at least because you carefully optmized your wordlist somehow.

Regardless, by manually analising the request/response flow, I couldn't find any indication there was a proxy or anything along the lines so my assumption was if there was a way around, it had to be by messing with whatever _regex_ validation was being used to filter the request or maybe, by taking advantage of some unexpected redirect. I then fired up [Burp Suite](https://portswigger.net/burp) and proxied the request, made a few changes and give it a go but then... It didn't work, initially at least. I toyed around with the headers once more expecting to get a better understanding of what was going on underneath but all in all, it seemed pretty standard. 

I then, navigated back to www.redacted.com, intercepted the request once more but this time, changed the __Host__ header by setting its value to __statistics.redacted.com__. Request hung up for a second aaaand... 

![intercept-1.png](/static/images/403-bypass/intercept-1.png)

Interestingly enough, a redirect :) 

![intercept-2.png](/static/images/403-bypass/intercept-2.png)

As I hit forward, I quickly landed on a login page.

![login.png](/static/images/403-bypass/login.png)

I went ahead and added a rule on Burp to match/replace every single request with the new found bypass and it worked like charm. By-pass-ed 🥳 

__Worth mentioning__: Later I found out that every other subdomain returning 403 status could also be bypassed by following the same steps described above.

## Wrapping it up

As I was oddly celebrading, I noticed I still had a login page right in front of me and I didn't have any confidential access, so not necessarily a big win. Regardless, I had bypassed a 403 restriction and from now on, I could automate a brute force attack or look for other ways to leverage access. It wasn't much but definitely enough to be reported so I set for the report. After disclosing the issue, the team has quickly acknowledged my finding and promptly patched the issue.

That's it, hope you've learnt something out of this.

### Timeline

* Oct 12, 2021 11:44am — Reported
* Oct 12, 2021 03:10pm — Triage
* Oct 13, 2021 01:10pm — Accepted
* Oct 26, 2021 08:20am — Rewarded