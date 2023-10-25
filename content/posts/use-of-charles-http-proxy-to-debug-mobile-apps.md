---
title: "Use of HTTP proxy (Charles Proxy) to debug mobile/desktop apps."
date: 2023-03-01
description: "Use of HTTP proxy (Charles Proxy) to debug mobile/desktop apps."
tags: ["http", "tools", "debug", "http proxy"]
---

When you're building API that is used by either mobile or desktop app you, eventually,
will encounter situation where you need to confirm how the app uses your API.

Mainly, but not only, cases like that happens when withing single company you have backend and mobile team.
There is some problem that you need to investigate, which might be related to wrongly used API or bug on the backend side.

Parts of the HTTP request that are useful during debugging are e.g.:
- type of HTTP request (POST, PUT, GET...)
- payload sent
- response payload from the backend
- HTTP status code

and so on. 

Obviously proper logging is something that would be ideal. But in reality sometimes it's not that easy.
Either there is lack of logging, it does not log all things you need (especially request and response payloads), or simply it's not that easy to get to the interesting information in high traffic environment.

Convenient solution in that case is **HTTP Proxy**.

## HTTP Proxy
What is **HTTP Proxy**? I will quote how **Charles Proxy** describes itself:
> HTTP proxy / HTTP monitor / Reverse Proxy enables a developer to view all of the HTTP and SSL / HTTPS traffic between their machine and the Internet. This includes requests, responses and the HTTP headers (which contain the cookies and caching information).

That's exactly what can help us understand the root of the problem when it comes to debugging `App <-> Backend` communication.

## Charles Proxy
Charles proxy is one of multiple tools, that you can use on your local machine, that work as **HTTP Proxy**, there is also: 
- [Fiddler](https://www.telerik.com/fiddler)
- [Wireshark](https://www.wireshark.org/)
- [Proxyman](https://proxyman.io/)

just to name a few. If you search for it, you will find plenty more. All of them have something in common, and some unique features.
I recommend checking them and choosing one that fits you best.

I base this post on **Charles Proxy** because with this tool I have most experience with.

## Useful functionalities
Let me try to show you few Charles Proxy functionalities that were useful for my day-to-day work,
when I had to understand what mobile apps are sending to one of our backends.

In my case I will base examples on iOS (iPhone) and Ikea app.

I assume that you have both, configured Charles Proxy SSL certificate on iPhone, and have Charles Proxy installed on your computer.

## Seeing requests sent by the app

That's what we all came here for, haven't we?

We want to know HTTP request method, endpoint url, query string, headers, cookies, payload, times, size...literally every aspect of request can be important when it comes to debugging, and solving problem.

![HTTP Proxy Summary](/img/use-of-charles-http-proxy/charles-proxy-request-summary.png)
![HTTP Proxy Payload](/img/use-of-charles-http-proxy/charles-proxy-request-payload.png)

## Saving and restoring session

Quite often it will not be only you who will be looking at a problem, or problem might occur on device with configuration which you can't really replicate on your device/emulator.

That's when Saving and Opening session comes handy. You can save current state, with all requests and their details, of session and save it to file. Then share it with your colleagues.

![HTTP Proxy Save Session](/img/use-of-charles-http-proxy/charles-proxy-save-session.png)

## Mapping URL to your local/remote endpoint

You can also map requests from the app to your local or remote endpoint. That can be useful when you want to handle requests from the app with your version of the backend and serve modified response.

![HTTP Proxy Map Remote 1](/img/use-of-charles-http-proxy/charles-proxy-map-remote-1.png)
![HTTP Proxy Map Remote 2](/img/use-of-charles-http-proxy/charles-proxy-map-remote-2.png)

## Other features

Those few features is not finite list of all that Charles Proxy has. There is [Throttling](https://www.charlesproxy.com/documentation/proxying/throttling/), [Reverse Proxy](https://www.charlesproxy.com/documentation/proxying/reverse-proxy/), [Mirror](https://www.charlesproxy.com/documentation/tools/mirror/), [DNS Spoofing](https://www.charlesproxy.com/documentation/tools/dns-spoofing/) and many more that can be useful
on a daily basis. 

Go through their documentation and explore! You will find it as a very useful tool!

