---
title: "Solving the CDN post-purge thundering herd problem: The PurgeGroup pattern"
date: 2021-06-12T00:00:00Z
draft: false
tags: ["Fastly", "CDN"]
---

Running a full purge of a CDN's cache can be a risky endeavour. Since a CDN usually
heavily caches requests, the origin only receives a fraction of all customers' traffic.

For a short amount of time following a purge operation, the CDN's cache is empty and all
incoming requests are forwarded to the origin. If the cache/hit
ratio of the CDN is 98%, the number of requests hitting the origin immediately after the
purge might be up to 50 times greater than usual. This is a problem known as
[thundering herd](https://en.wikipedia.org/wiki/Thundering_herd_problem).
As the cache is being repopulated, the cache/hit ratio will gradually increase and the number
of requests hitting the origin will slowly go back to normal.

Often, the origin is not sized to deal with such sudden increase in load. Therefore,
fully purging the cache is prohibitively expensive, and a tool of last resort.

A solution to solve this problem would be to purge the cache gradually over time, rather than
in one fast operation. Unfortunately, no CDN vendor provides this as a feature.

*Enter Surrogate-Keys / Cache-Tags.*

Most CDN providers allow you to purge a cache by *Key*: [Fastly](https://docs.fastly.com/en/guides/getting-started-with-surrogate-keys),
[Akamai](https://developer.akamai.com/blog/2019/03/28/technical-deep-dive-purging-cache-tag),
[Cloudflare](https://support.cloudflare.com/hc/en-us/articles/200169246-Purging-cached-resources-from-Cloudflare#h_6d756ac9-c476-45e8-a5d4-e2a6e45d9dc7), 
and many more support this.

By assigning a random "PurgeGroupID" cache-tag to all requests, for example 
between PurgeGroup0 and PurgeGroup99, we can divide the cache in similarly-sized
buckets - each containing around 1% of requests.  Purging one of these PurgeGroups
will therefore purge about 1% of the cache.

Adding such a cache-tag can be done either at the origin, or directly in the CDN, depending
on the provider: the following (example with Fastly)[https://fiddle.fastlydemo.net/fiddle/96a1f42e]
adds a random PurgeGroup surrogate-key to each request.

The cache can then be purged gradually:
 * Purge Cache-Tag Purgegroup0
 * Wait 10 seconds
 * Purge Cache-Tag Purgegroup1
 * Wait 10 seconds
 * Purge Cache-Tag Purgegroup2
 * ...

The number of PurgeGroups, as well as the time to wait between each purge operation, will
depend on how fast the purge needs to happen, and on how many requests the origin is able
to receive.

I refer to this method as the "PurgeGroup pattern". It is pretty simple, and should work
with most major CDN providers.