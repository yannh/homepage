---
title: "The curious case of the CDN Cache-HISS"
date: 2023-08-06T00:00:00Z
draft: false
tags: ["Fastly", "CDN"]
---

A few years ago, I was asked to take a more active role maintaining the CDN configuration of the company I work for. My knowledge of CDNs at the time was _pretty basic_ and could be summarised by the following diagramm:

![Cache-Hit or Cache-MISS](/assets/cache-hit-miss.png)

"Sure, I got this". And for a short time, things were good. But soon after, I was asked the following question:


  "We are reviewing the performance of a client's requests - several of their requests are quite slow, despite being Cache HITs. What's going on?".


And I soon realised, who would have guessed - things are never that simple. Cache-HITs? Cache-MISSes? Cache-HIT ratios? Things have gotten complicated, and those terms don't make all that much sense anymore..


Let me introduce you: the cache HISS: a little bit cache-HIT, a little bit cache-MISS... and it comes in different flavours!


# Definitions

But before going further, let's define some important terms. In this blog post, we will be using the following definitions:

 * A cache *HIT* is a request where the response is served using content that was previously cached by the CDN.
 * A cache *MISS* is a request where the response comes from the CDN origin / the "backend"  behind the CDN.

These are _derived_ from the [Fastly documentation](https://developer.fastly.com/reference/vcl/variables/miscellaneous/fastly-info-state/). I am not aware of an industry-standard definition - consider this my attempt at defining those terms!


# HTTP 304 - Not Modified

Imagine you are managing a CDN in front of a file repository. All the files in the repository are quite large, maybe several GBs. Those files do change, but really quite rarely. It is acceptable if new versions of a file are served within one hour of being uploaded, therefore you configure a Time-To-Live (TTL) in your CDN: the CDN should cache the file for not more than one hour after retrieval.  

Assuming each file is requested several times per hour, this would mean that the CDN would need to download each file again every hour, to ensure it has the latest version in cache. In practice, a CDN has multiple caches around the globe (Points-of-Presence or POPs), so it might be worse: each POP might need to refresh every file every hour.

This could quickly become very expensive and impractical: your file server would be constantly sending many large files that have not changed to the CDN.

To not overload the server, modern CDNs use [ETags](https://en.wikipedia.org/wiki/HTTP_ETag). If the CDN receives a request for which the cached file's TTL has expired, it will compute a _hash_ of the file in cache that is not valid anymore, and send that hash along with the request. This is akin to saying to your fileserver "Can you send me file X? I already have this version in cache, so it has not changed, don't sent me the whole file again". 

If your fileserver supports ETAGs, it will compute the ETAG of the file requested. If the file has not changed and the computed ETAG matches the ETAG sent by the CDN, instead of sending the whole file back, your fileserver will return a HTTP/304 Not Modified response. The CDN would then renew the TTL of the file in cache for another 60 minutes, *and serve the file it has in cache*.

What happened?

 * The request was served by the CDN, using content that was previously cached
 * However, the request went to the fileserver, and potentially required some computation there.


According to our definition - this is a cache-HIT. Somewhat unintuitively though, the request was sent to the backend server, which had to do some computation... and is therefore quite different from a request that was immediately served from cache without revalidation. 

Cache-HISS?


# Request collapsing


To improve on the previous architecture, you decide that setting a TTL of 1h on your files is not that great after all. You need to wait up to an hour for the new versions of your files to be served, and the revalidations requests from the CDN are not very efficient - most of your files never change!

You decide that having a much higher TTL (1 year), and purging files from the CDN when they change would mean fewer revalidation requests to your fileserver, and ensuring the new versions of the files are available much sooner.


Now - some of the files are requested quite a lot. Especially after a file changes, a lot of people want to download the new version! So let's see what would actually happen.

 * You upload a new version of a large file. 
 * You send a purge request for that file to the CDN.
 * You send an email advertising that there is a new version of the file
 * Over the next 10 seconds after you sent the email, 100 different persons start downloading the file.
 * _Assuming it takes 10 seconds for the CDN to retrieve the file from the backend server_, for 10 seconds following the first request, the file will not be in the CDN's cache yet.
 * Since the CDN does not have the file in cache when those 100 requests are received, the CDN makes 100 requests to the backend, which serves the same large file 100 times.

This is quite impractical - and to reduce the impact on the backend server, modern CDNs will use [request collapsing](https://developer.fastly.com/learning/concepts/edge-state/cache/request-collapsing/). Request collapsing is an optimisation where the CDN will realise, upon receiving one of these 100 requests, that it already is retrieving that file from the backend for an earlier request. It will therefore "park" the request, waiting for the earlier request to complete. Once the earlier request is completed, the file will be served from cache for all "parked" requests. We say that those "parked" requests are "collapsed" into the earlier request.

What happened?
 
 * The first request returned content served by the fileserver, and was a cache MISS.
 * The other 99 requests served data from the CDN cache, populated by the first requests. They are, technically, cache HITs.

Unintuitively enough - while the requests that returned cache HITs did not go to the backend server, they were waiting, some for up to 10second, for the first request to complete. 

Cache HISS?


# Multi-Tiered architectures

In an earlier blog post about the [execution model of AWS Lambda@edge in Cloudfront](https://yann.mandragor.org/posts/lambda-execution-model/), I briefly touched on the concept of multi-tiered CDN architectures. Indeed most CDN nowadays [[Cloudflare's orpheus](https://blog.cloudflare.com/orpheus/), [Cloudfront Origin shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html), [Fastly shielding](https://developer.fastly.com/learning/concepts/shielding/)] enable the configuration of multiple layers of caching, 2, sometimes even 3. 


![Cloudfront's tiered architecture](/assets/cloudfront-tiered-architecture.png)

In this example of a three-tier setup with Cloudfront, the "edge location" will be geographically quite close to the user - but the "origin shield" will be geographically much closer to the origin file server, and potentially quite far from the user. What this architecture implies - is that a request might be a HIT or a MISS in every of the different caching layers.

Let's take the following use-case: you have a fileserver located in Virgina (US-East), and use a CDN using a three-tiered architecture such as the one presented above. Your fileserver is in Virginia (us-east), and a user from India requests a 1KB file. Noone in india requested that file in India before, therefore neither the edge location nor the Regional Edge cache has the file in cache. The request therefore goes all around the globe to US-East to the origin shield cache. Someone in the US requested that file earlier, therefore the file is in cache in the origin shield, and the request is therefore a Cache-HIT in the shield. 

What happened?

 * The request returned to the user was served by the CDN cache, it is a Cache-HIT
 * However, the request had to go all around the planet to reach the origin shield, and was barely faster than a non-cached request.


Cache-HISS?


# Stale content (stale-on-5xx)

When a file is cached by a CDN and its TTL expires, or if you purge that file from the CDN, the CDN will often keep the file in cache for a little while and mark it as "stale", instead of directly deleting it. This is because _in some cases_, you might decide that serving content that is slightly out of date is better than the alternative.

The most common case is when the TTL of a cached file in the CDN expires, and the CDN tries to fetch a new version from your backend server. What if there is an error fetching the new version, because your fileserver is currently overloaded or down? Would it be better to serve a 5xx HTTP error - or a slightly outdated file? 

This behavious is known as _stale-on-error_ and is available with [most](https://aws.amazon.com/about-aws/whats-new/2023/05/amazon-cloudfront-stale-while-revalidate-stale-if-error-cache-control-directives/) [CDN](https://developers.cloudflare.com/cache/concepts/cache-control/#revalidation) [providers](https://docs.fastly.com/en/guides/serving-stale-content#serving-stale-content-on-errors). Depending on your business requirements, this might be a good option to turn on.

Let's take the following example:
 * A user requests a file
 * The CDN has a cached version of that file, but since the TTL of that cached object expired, it forwards the request to the backend server
 * The backend server is currently overloaded - the connection times out after several seconds.
 * The option "stale-on-error" is turned on - therefore the CDN serves the stale object from cache.

What happened?
 * The file was served from cache - it is a cache HIT
 * The request went to the origin (or tried to). It was very slow to complete since the CDN waited for a timeout to occur before giving up and serving the stale file.

Cache HISS!



## Final thoughts


The fact that a request is a cache "HIT" tells us very little about what happened. In some cases, the request might hit a caching layer half way around the planet, wait for another request to complete, hit a timeout or an error, or even hit the origin and require computation there. It tells us nothing about the two things that really matter:

 * Was the user experience good - was the file served to the user with low latency?
 * Did the request put load on the backend server?

By extension: cache HISSes might completely skew what you believe the cache-HIT ratio reported by your CDN is telling you. It does not necessarily correlate with a good user experience, not is it necessarily an indication of how many requests reach your origin. There is also no agreed standard on how to measure the cache-HIT ratio - depending on your provider, it might be unclear how cache HISSes are counted.

So - forget about cache hit ratios! And measure what matters:

 * With what latency was the file served (Time To First Byte, Total)? You might measure this either at the edge of your network, in the CDN - or using instrumentation on the client side.
 * How many requests, from all requests, reached the origin? Especially if the requests require heavy computation (in the case of a REST or GraphQL API), returning 304s might not save you all this much.
