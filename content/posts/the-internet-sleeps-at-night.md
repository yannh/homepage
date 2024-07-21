---
title: "😴 The Internet sleeps at night 🌙"
date: 2024-06-21T00:00:00Z
draft: true
tags: ["Fastly", "CDN", "internet"]
---

Does it ever feel like the internet is slower at night?

Like me, did you assume that the time of day should have nothing to do with Internet
speed... ?

Today I would like to explain the following graph, showing the average and median
latency for a large website, as seen by users in South America. By understanding Content
Delivery Networks (CDNs), TCP, and stateful HTTP connections better, we will demonstrate
_why_ a large portion of websites is indeed slightly slower at night.
 
![POP to origin latency, South America](/assets/latency_south_america_light.png "Average and Median Latency from South America (ms)")


## POP-to-origin latency

We will base our discussion on a large website hosted in AWS us-east-1 (in North
Virginia, US) that uses a CDN, in this case [Fastly](https://www.fastly.com). The
behaviour has only been observed using Fastly, but is likely to be also present
with other CDNs. If you are able to reproduce this experiment using a different CDN,
please get in touch!

The metric shown in this graph is what I call "pop-to-origin latency", which is the
duration of the request as measured by the CDN POP, minus the duration as measured by
the load balancer at the origin that received the request.

![POP to origin latency](/assets/pop-to-origin-latency.png "Pop to origin latency")

Fastly operates multiple POPs in South America, and we would expect different
POP-to-origin latencies for each of them, in function of their relative distance
to AWS us-east-1.

![Average POP-to-origin latency by POP](/assets/pop-to-origin-latency-by-pop.png "Average POP-to-origin latency by POP in South America (ms)")


The POP referred to as BOG is Bogota (Colombia), and usually experiences highest latency
around 9-11AM on this graph, where time is in CET (Berlin time): that would
 actually be 2-4AM in Bogota, in the middle of the night.

So - in every single PoP in South America, we see pop-to-origin latency degrade
significantly as people go to sleep, and then get better as they wake up. Not
something I expected!


## Latency distribution

We start to see a bit more clear when we look at the distribution of the pop-to-origin
latency (here, during the day in South America). We see most requests have a latency
of between 80-120ms, with a smaller amount of requests between 240 and 300ms, roughly
three times as much.

![POP-to-origin latency distribution for BOG (day, ms)](/assets/pop-to-origin-latency-distribution.png "POP-to-origin latency distribution for the POP BOG (day, ms)")

Why are some requests significantly slower? This is due to the CDN performing an
optimisation by keeping the TCP connection open to the origin (TCP Keep-Alive).

![TCP connection (Wikipedia, CC-BY-SA-3.0-migrated)](/assets/tcp-handshake.png "TCP connection (Wikipedia, CC-BY-SA-3.0-migrated)")

Requests that required the CDN to run a full TCP connection to the origin will
be impacted by the latency of the full TCP handshake, while requests that can 
benefit from an already established connection will skip that. The graph showing
the pop-to-origin latency distribution actually shows the quality of the
connection pooling from the CDN.

## The internet does sleep at night

Now, what happens during the night? Keeping connections open has a cost for the CDN
provider: there is a finite amount of open connections per POP it can keep, and that
is shared across all its customers.

During the night, your website is likely to receive fewer requests - which translates
to fewer opportunities to keep those worthy connections open. Opened connections are
more likely to time out: more connections will have to initialise a full new TCP
handshake.

![POP-to-origin latency distribution for BOG (night, ms)](/assets/pop-to-origin-latency-distribution-night.png "POP-to-origin latency distribution for the POP BOG (night, ms)")

So, for the same POP, but for a different timeframe: we see fewer requests, and more
requests having to do a full TCP handshake. On average: requests are slower.


## Your mileage may vary

I picked a pretty extreme example to describe what was happening here, taking one
of the CDN PoPs with very little traffic. In most large PoPs, the number of
requests will stay quite high 24/7 so the differences between day & night is
a lot less pronounced. Even though, I observe similar daily fluctuations all
across the globe.

The quality of the connection pooling is also likely very dependent on the CDN
provider. Our example here uses Fastly, which overall does an extremely good job
at keeping connections open as much as possible. Results might be better, or worse,
using different CDN providers. I would love to hear from you if you are able to
reproduce this experiment!

