---
layout: post
title: "SPYDing the web"
description: "Some thing about SPDY and HTTP/2. I had heard a lot about both these two and it was time to go explore them."
category: web
tags: [Threads, AsyncTask, Android]
subheading: What was SPDY and what is HTTP/2
permalink: web/spdy-and-http2
---

HTTP/2 is in the news for some time now.I was using [OkHttp](https://github.com/square/okhttp),which Square made compatible with HTTP/2 and now OkHttp also powers Android's HTTP client. But i still did't know what this SPDY is because probably
it didn't change anything much for me.

## What was SPDY
It sounded a bit strange that a single TCP connection was capable of handling only 1 HTTP request at a time.There is no
concurrency involved here.So for the browser to achieve concurrency and fetch multiple resources at once,it will have to open multiple TCP connections per host. So most of the browsers(including chrome) have the limitation of opening a max of 6 TCP connections for a host. That means if you have a host and you fire 7 downloads from it at the same time from the same browser, the 7th one will wait until one of them is successfully downloaded.
<br><br>
This was one of the several glaring flaws that existed in HTTP/1.X. As it was formulated in 1999 and continued for about
another 15 years, it was really a time to improve it in certain aspects to fit the current needs. So Google took the responsibility upon **herself** and came out with SPDY. IETF,after some time started with HTTP/2 considering SPDY as its base and they improved on SPDY by taking the inputs from the community and the implementors of HTTP clients of various other languages. Thereafter,Google decided to deprecate SPDY and support HTTP/2 only with both Google's lead SPDY engineers Mike Belshe and Roberto Peon working on the HTTP/2 implementation.

## What is HTTP/2

HTTP/2 aims to solve the fundamental concurrency problem mentioned above and also reduce the latency of resource fetching.
HTTP/2 is not a replacement for HTTP but as its name suggests, just few more functions to support additional features.

### Multiplexed requests and response

With pipelining of course,the browser can send multiple requests in a single TCP stream but it imposes a condition that the server should
send the response in the same order as the received requests. If the previous request is taking too much of time, the server
cant send the response of a request which arrived later than the previous one even if its response has been calculated. This is
HOL ( Head Of Line ) blocking. Pipelining is illustrated by the below image (taken from WIKI).

![alt text][logo]

HOL blocking prevents pipelining's implementation and use in the major browsers. But HTTP/2 solves this issue and  allows for unlimited concurrent streams
over a single TCP connection.

### Header compression
As both requests and response headers vary in size from few bytes to few kilobytes,sending them in plain text mode again contributes to latency.Also the same headers are exchanged again and again between a host and a client.Once a connection with
a host has been established, there is no need to send such redundant headers. So HTTP/2 compresses request and response header metadata using the HPACK compression format

### Binary Protocol
HTTP/2 is a binary protocol unlike HTTP/1.x which was a text based protocol. Now obviously there are some both bright
and dark sides of both protocols but a binary protocol will always be more efficient when it comes to parsing of data and processing it. Text based protocols are always heavy even if the data has been compressed using any compression
mechanism.  

According to the HTTP/2 spec, there are a number of issues to handle in case of text protocols like whitespace handling, capitalization, line endings, blank links and so on. [This](http://stackoverflow.com/questions/2525188/are-binary-protocols-dead) answer gives a very good comparison
for text and binary.Obviously text protocols were designed because it was easier for us to read or interpret.Speed and latency was not given the top priority in that case. But now since every millisecond counts in case of serving web pages, a binary protocol is the need of the hour.

### Server push
Generally what happens in case of HTTP is that a client requests a resource from a server and if there are multiple resources present in that page, then again a request is made by the client individually for fetching each of the resource.
But since the server while sending the first page already knows what extra resources the client will need, then why
cant the server send all the resources in a single request? This is actually server push.
<br>
With resource inlining, the same thing can be achieved in HTTP/1.x also but the inlined resource will be fetched with every page separately and also it cannot be cached individually as a standalone resource by the browser but it will exist only as a part of that page with which it arrived. With HTTP/2, the aim is to fetch the inclined resources once, cache it and use it across multiple pages.
<br>
Before sending out the actual response for a page, the server will send frames which contains information about which
other resources the server intends to push along with the requested resource  so that the client can reject the push for a certain resource if it has already been cached or at least avoid creating duplicate request for it.
<br><br>

These were some major changes that are there in HTTP/2. There are many other things like Request prioritization etc. The full spec can be read from [here](https://http2.github.io/). HTTP/2 will have a major effect on websites where milliseconds lag can makes a huge difference and also mobile/non-browser clients where memory and processing power is still very precious.Client implementations have been done for most of the languages and are available [here](https://github.com/http2/http2-spec/wiki/Implementations). Also Tomcat already supports it and Apache support is on the way i think.So its all shifting rapidly.


[logo]: /images/pipelining.png "Logo Title Text 2"
