---
layout: post
title: "SPYDing the web"
description: "It was the first time i looked into the source code of an Android component deeply.I wrote it
long back before this blog was made.Found it very interesting and now digging into the source code of every
other thing has become a habit."
category: web
tags: [Threads, AsyncTask, Android]
subheading: What was SPDY and what is HTTP/2
permalink: web/spdy-and-http2
---

HTTP/2 is in the news for some time now.I was using [OkHttp](https://github.com/square/okhttp),which Square made compatible with HTTP/2 and now OkHttp also powers Android's HTTP client. But i still did't know what this SPDY is because probably
it didn't change anything much for me.

## What was SPDY
It sounded a bit strange that a single TCP connection was capable of handling only 1 HTTP request at a time.There is no
concurrency involved here.So for the browser to fetch multiple resources at once,it will have to open multiple TCP connections per host. So most of the browsers(including chrome) have the limitation of opening a max of 6 TCP connections for a host. That means if you have a host and you fire 7 downloads from it at the same time from the same browser, the 7th one will wait until one of them is successfully downloaded.
<br><br>
This was one of the several glaring flaws that existed in HTTP/1.X. As it was formulated in 1999 and continued for about
another 15 years, it was really a time to improve it in certain aspects to fit the current needs. So Google took the responsibility upon **herself** and came out with SPDY. IETF,after some time started with HTTP/2 considering SPDY as its base and they improved on SPDY by taking the inputs from the community and the implementors of HTTP clients of various other languages. Thereafter,Google decided to deprecate SPDY and support HTTP/2 only with both Google's lead SPDY engineers Mike Belshe and Roberto Peon working on the HTTP/2 implementation.

## What is HTTP/2

HTTP/2 aims to solve the fundamental concurrency problem mentioned above and also reduce the latency of resource fetching.
HTTP/2 is not a replacement for HTTP but as its name suggests, just few more functions to support additional features.

### Multiplexed requests

With pipelining,the browser can send multiple requests in a single TCP stream but it imposes a condition that the server should
send the response in the same order as the received requests. If if the previous request is taking too much of time, the server
cant send the response of a request which arrived later than the previous one even if its response has been calculated. This is
HOL blocking. Pipelining is illustrated by the below image (taken from WIKI).

![alt text][logo]

HOL blocking prevents pipelining's implementation and use in the major browsers. HTTP/2 allows for unlimited concurrent streams
over a single TCP connection.

### Header compression
As both requests and response headers vary in size from few bytes to few kilobytes,sending them in plain text mode again contributes to latency.Also the same headers are exchanged again and again between a host and a client.Once a connection with
a host has been established, there is no need to send such redundant headers. So HTTP/2 compresses request and response header metadata using the HPACK compression format

[logo]: /images/pipelining.png "Logo Title Text 2"
