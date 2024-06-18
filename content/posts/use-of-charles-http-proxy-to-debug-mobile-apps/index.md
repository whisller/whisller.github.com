---
title: "Use of HTTP proxy (Charles Proxy) to debug mobile/desktop apps."
date: 2023-03-01
description: "Use of HTTP proxy (Charles Proxy) to debug mobile/desktop apps."
tags: ["http", "tools", "debug", "http proxy"]
---

When you're building an API that is used by either a mobile or desktop app, you will eventually encounter a situation where you need to confirm how the app uses your API.

This situation often arises, but is not limited to, cases within a single company where you have both a backend and a mobile team. There is a problem that you need to investigate, which might be related to the wrongly used API or a bug on the backend side.

Parts of the HTTP request that are useful during debugging include:
- Type of HTTP request (POST, PUT, GET...)
- Payload sent
- Response payload from the backend
- HTTP status code
- and more.

Obviously you can think about using logging, but in reality, it's not always that easy or convenient. Either there is a lack of logging, it does not log all the things you need (especially request and response payloads), or simply, 
Or you just want to do a little bit more than just see requests.

A convenient solution in that case is an **HTTP Proxy**.

## HTTP Proxy
What is an **HTTP Proxy**? I will quote how **Charles Proxy** describes itself:
> HTTP proxy / HTTP monitor / Reverse Proxy enables a developer to view all of the HTTP and SSL / HTTPS traffic between their machine and the Internet. This includes requests, responses and the HTTP headers (which contain the cookies and caching information).

That's exactly what can help us understand the root of the problem when it comes to debugging `App <-> Backend` communication.

## Charles Proxy
Charles Proxy is one of several tools that you can use on your local machine as an **HTTP Proxy**. There are also:
- [Fiddler](https://www.telerik.com/fiddler)
- [Wireshark](https://www.wireshark.org/)
- [Proxyman](https://proxyman.io/)
  and many more. If you search for it, you will find plenty more. All of them have something in common and some unique features. I recommend checking them out and choosing the one that suits you best.

I base this post on **Charles Proxy** because it's the tool I have the most experience with.

## Useful Functionalities
Let me show you a few of Charles Proxy's functionalities that were useful for my day-to-day work when I had to understand what mobile apps are sending to one of our backends.

In my case, I will base examples on iOS (iPhone) and the Ikea app.

I assume that you have both configured the Charles Proxy SSL certificate on your iPhone and have Charles Proxy installed on your computer.

## Seeing Requests Sent by the App

That's what we all came here for, haven't we?

We want to know the HTTP request method, endpoint URL, query string, headers, cookies, payload, times, size... literally every aspect of the request can be important when it comes to debugging and solving problems.

![HTTP Proxy Summary](/img/use-of-charles-http-proxy/charles-proxy-request-summary.png)
![HTTP Proxy Payload](/img/use-of-charles-http-proxy/charles-proxy-request-payload.png)

## Saving and Restoring Sessions

Quite often, it will not be only you who will be looking at a problem, or the problem might occur on a device with a configuration that you can't really replicate on your device/emulator.

That's when saving and opening sessions come in handy. You can save the current state, with all requests and their details, of the session and save it to a file. Then, share it with your colleagues.

![HTTP Proxy Save Session](/img/use-of-charles-http-proxy/charles-proxy-save-session.png)

## Mapping URLs to Your Local/Remote Endpoint

You can also map requests from the app to your local or remote endpoint. That can be useful when you want to handle requests from the app with your version of the backend and serve a modified response.

![HTTP Proxy Map Remote 1](/img/use-of-charles-http-proxy/charles-proxy-map-remote-1.png)
![HTTP Proxy Map Remote 2](/img/use-of-charles-http-proxy/charles-proxy-map-remote-2.png)

## Other Features

These few features are not an exhaustive list of all that Charles Proxy has. There is [Throttling](https://www.charlesproxy.com/documentation/proxying/throttling), [Reverse Proxy](https://www.charlesproxy.com/documentation/proxying/reverse-proxy), [Mirror](https://www.charlesproxy.com/documentation/tools/mirror), [DNS Spoofing](https://www.charlesproxy.com/documentation/tools/dns-spoofing), and many more that can be useful on a daily basis.

Go through their documentation and explore! You will find it to be a very useful tool!

