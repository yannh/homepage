---
title: "😴 The Internet sleeps at night 🌙"
date: 2024-06-21T00:00:00Z
draft: false
tags: ["Fastly", "CDN", "internet"]
---
Does it ever feel like the internet is slower at night?

Did you, like me, assume that the time of day should have nothing to do
with internet speed?

Today, I would like to explain the following graph, which shows the average
and median latency for a large website as experienced by users in South
America. By understanding Content Delivery Networks (CDNs), TCP, and stateful
HTTP connections better, we will demonstrate why a large portion of websites
is indeed slightly slower at night.

![POP to origin latency, South America](/assets/latency_south_america_light.png "Average and Median Latency from South America (ms)")


## POP-to-origin latency

We will base our discussion on a large website hosted in AWS us-east-1 (in North
Virginia, US) that uses a CDN, in this case [Fastly](https://www.fastly.com).
This behavior has only been observed using Fastly but is likely present with
other CDNs as well.

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

It becomes clearer when we look at the distribution of pop-to-origin latency during
the day in South America. Most requests have a latency between 80-120ms, with a smaller
number of requests between 240-300ms, roughly three times as much.

![POP-to-origin latency distribution for BOG (day, ms)](/assets/pop-to-origin-latency-distribution.png "POP-to-origin latency distribution for the POP BOG (day, ms)")

Why are some requests significantly slower? This is due to the CDN performing an
optimisation by keeping the TCP connection open to the origin (TCP Keep-Alive).

![TCP connection (Wikipedia, CC-BY-SA-3.0-migrated)](/assets/tcp-handshake.png "TCP connection (Wikipedia, CC-BY-SA-3.0-migrated)")

Requests that require the CDN to run a full TCP connection to the origin will be
impacted by the latency of the full TCP handshake, while requests that benefit
from an already established connection will avoid this delay. The graph showing
the pop-to-origin latency distribution illustrates the quality of the connection
pooling from the CDN.

## The internet does sleep at night

Now, what happens during the night? Keeping connections open has a cost for the
CDN provider. There is a finite number of open connections per POP that it can
maintain, and this is shared across all its customers.

During the night, your website is likely to receive fewer requests, resulting
in fewer opportunities to keep those valuable connections open. As a result,
opened connections are more likely to time out, and more connections will have
to initiate a full new TCP handshake.

![POP-to-origin latency distribution for BOG (night, ms)](/assets/pop-to-origin-latency-distribution-night.png "POP-to-origin latency distribution for the POP BOG (night, ms)")

So, for the same POP but during a different timeframe, we see fewer requests
and more requests needing to perform a full TCP handshake. On average, this
results in slower requests.


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
