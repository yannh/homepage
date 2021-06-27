---
title: "Solving the CDN post-purge thundering herd problem: The PurgeGroup pattern"
date: 2021-06-12T00:00:00Z
draft: false
tags: ["Fastly", "CDN"]
requires: ["fiddle", "prism"]
---

Purging a Content Delivery Network (CDN) cache can be a risky endeavour. Since a CDN usually
heavily caches requests, the origin only receives a fraction of all customers' traffic.

For a short amount of time following a purge operation, the CDN cache is empty and all
incoming requests are sent to the origin. If the cache hit ratio is usually around 98%,
the number of requests hitting the origin immediately after the
purge might be up to **50 times greater** than usual. This is an example of the
[thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem).

___

When the origin is not preemptively over-provisioned to deal with such sudden increases in load,
fully purging the cache leads to an overload of the origin service, ultimately leading to request errors.
Only as the cache is being repopulated will the number of requests sent to the origin go down, 
and the origin service recover.

Therefore, fully purging a CDN cache is often a tool of last resort.

A solution to this problem would be to purge the cache gradually over time, rather than
in one fast operation. Unfortunately, no CDN vendor provides this as a feature, to my knowledge.

**Enter Surrogate-Keys / Cache-Tags.**

Many CDN providers allow you to purge a cache by *Key*, referred to as "Cache Tag", "Content Tag" or "Surrogate Key":
 * [Fastly](https://docs.fastly.com/en/guides/getting-started-with-surrogate-keys) 
 * [Akamai](https://developer.akamai.com/blog/2019/03/28/technical-deep-dive-purging-cache-tag)
 * [Cloudflare](https://support.cloudflare.com/hc/en-us/articles/200169246-Purging-cached-resources-from-Cloudflare#h_6d756ac9-c476-45e8-a5d4-e2a6e45d9dc7)
 * [Imperva](https://docs.imperva.com/bundle/cloud-application-security/page/settings/caching-settings.htm#Purgethecache)
 * [Limelight](https://www.limelight.com/resources/data-sheet/smartpurge/)
 * and maybe more?

By assigning a random "PurgeGroupID" cache-tag to all requests, for example 
between PurgeGroup0 and PurgeGroup99, we can divide the cache in similarly-sized
buckets - each containing around 1% of requests. Purging one of these PurgeGroups
will therefore purge about 1% of the cache.

Adding such a cache-tag can be done either at the origin, or directly in the CDN, depending
on the provider. The following example with Fastly adds a random PurgeGroup surrogate-key to each request
response:

{{< rawhtml >}}
<p><a href='https://fiddle.fastlydemo.net/fiddle/d4b88fa4/embedded'>Go look at this fiddle</a></p>
{{< /rawhtml >}}

The cache can then be purged gradually:

{{< prism >}}#!/bin/bash
FASTLY_TOKEN=secrettoken
SERVICE_ID=123456
for PURGE_GROUP in `seq 0 99`; do
  curl -X POST -H "Fastly-Key:${FASTLY_TOKEN}" https://api.fastly.com/service/${SERVICE_ID}/purge/PurgeGroup${PURGE_GROUP}
  sleep 5
done
{{< /prism >}}

The number of PurgeGroups, as well as the time to wait between each purge operation, will
depend on how fast the purge needs to happen, and on how many requests the origin is able
to receive.

I refer to this method as the "PurgeGroup pattern". It is pretty simple, and should work
with all CDN providers that support cache tags.

