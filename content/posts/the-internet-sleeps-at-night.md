---
title: "😴 The Internet sleeps at night 🌙"
date: 2024-06-21T00:00:00Z
draft: false
tags: ["Fastly", "CDN", "internet"]
---
Does it ever feel like the internet is slower at night?

Did you, like me, assume that the time of day should have nothing to do
with internet speed?

Today, we will look into the following graph, showing increased average
and median latency during night-time for a large website as experienced by
users in South America. By understanding Content Delivery Networks (CDNs),
TCP, and stateful HTTP connections better, we will demonstrate why a large
portion of websites is indeed slightly slower at night.

![PoP to origin latency, South America](/assets/latency_south_america_light.png "Average and Median Latency from South America (ms)")


## PoP-to-origin latency

The website in question is hosted in AWS us-east-1 (in North
Virginia, US) and uses a CDN, in this case [Fastly](https://www.fastly.com).
The behavior has only been observed using Fastly, but is likely present with
other CDNs as well.

The metric shown in this graph is what I call "PoP-to-origin latency", which is
the duration of the request as measured by the CDN Point-of-Presence (PoP),
minus the duration as measured by the load balancer at the origin that received
the request.

![PoP to origin latency](/assets/pop-to-origin-latency.png "PoP to origin latency")

Fastly operates multiple PoP in South America, and we would expect different
PoP-to-origin latencies for each of them, in function of their relative distance
to AWS us-east-1.

![Average PoP-to-origin latency by PoP](/assets/pop-to-origin-latency-by-pop.png "Average PoP-to-origin latency by PoP in South America (ms)")


The PoP referred to as BOG is Bogota (Colombia), and usually experiences
highest latency around 9-11AM on this graph, where time is in CET (Berlin
time): that would actually be 2-4AM in Bogota, in the middle of the night.

So - in every single PoP in South America, we see PoP-to-origin latency degrade
significantly as people go to sleep, and then get better as they wake up. Not
something I expected!

## Latency distribution

It becomes clearer when we look at the distribution of PoP-to-origin latency
for a single PoP (here: BOG). Most requests have a latency between 80-120ms,
with a smaller number of requests between 240-300ms, roughly three times as
much.

![Number of connections, by PoP-to-origin latency for BOG (day, ms)](/assets/pop-to-origin-latency-distribution.png "Number of connections, by PoP-to-origin latency for BOG (during the day, ms)")
![PoP-to-origin latency distribution for BOG (night, ms)](/assets/pop-to-origin-latency-distribution-night.png "PoP-to-origin latency distribution for BOG (during the night, ms)")

Why are some requests significantly slower? This is due to the CDN performing an
optimisation by keeping the TCP connections open to the origin (TCP Keep-Alive).

When the CDN performs a requests to a website, it has to open a TCP connection
to it, and perform what is called a TCP "handshake". This is somewhat time
intensive, and therefore once a TCP connection is open, the CDN provider will keep
it open for a little while, building a pool of open connections to the website.

![TCP connection (Wikipedia, CC-BY-SA-3.0-migrated)](/assets/tcp-handshake.png "TCP connection (Wikipedia, CC-BY-SA-3.0-migrated)")

The already-opened connections in that pool can be reused later by requests
going to the same origin. Requests that benefit from an already established
connection will avoid the delay incurred by initializing the TCP connection,
resulting in faster response times.

The graphs showing the PoP-to-origin latency distribution therefore also
illustrate the quality of the connection pooling provided by the CDN and
indicate how many requests were able to reuse an already-opened connection.


## The internet does sleep at night

Now, what happens during the night? Keeping connections open has a cost for the
CDN provider. There is a finite number of open connections per PoP that it can
maintain, and this is resource shared across all its customers.

During the night, your website is likely to receive fewer requests, resulting
in fewer opportunities to keep those valuable connections open. As a result,
opened connections are more likely to time out, and more connections will have
to initiate a full TCP handshake.

In our example, for the same PoP and the same amount of time but during the
night, we see fewer requests overall, but more requests needing to perform a
full TCP handshake. This results in slower requests, on average.

## CDN performance

CDN performance depends on many factors. The distance between a user and the
closest Point Of Presence and the ability of the CDN to cache as many files
as possible are the most well-known.

The quality of connection pooling between your CDN and your origin can also
significantly impact your website's performance. As we have seen, the amount
of traffic to your website can influence how well that connection pooling
performs.

During periods of low traffic, you may observe a noticeable increase in
connection latency.

## Your mileage may vary

I chose a particularly extreme example to illustrate what was happening, using
one of the CDN PoPs with very little traffic. In most large PoPs, the number
of requests remains quite high 24/7, so the differences between day and night
are much less pronounced. However, I observe similar daily fluctuations
globally.

The quality of the connection pooling is also likely very dependent on the
CDN provider. Our example here uses Fastly, which overall does an excellent
job of keeping connections open as much as possible. Results might be better
or worse with different CDN providers. I would love to hear from you if you
are able to reproduce this experiment!
