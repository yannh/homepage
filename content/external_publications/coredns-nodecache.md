---
title: 'Creating greater reliability: CoreDNS-nodecache'
date: 2020-01-07T00:00:00Z
draft: false
location: Contentful Blog
externalUrl: "https://www.contentful.com/blog/2020/01/07/coredns-nodecache-blog/"
thumbnail: 'https://images.ctfassets.net/fo9twyrwpveg/tLuwOsjMj8BuHNy5qW44u/bbecfbabb15edec3f95e4ee559d53573/BLOG_Creating_stability-01.png?fm=webp&q=90&w=260'
---

Ongoing issues in the Linux kernel’s UDP connection tracking have caused challenges with DNS, and bugs
particularly affect DNS in Kubernetes in its default configuration; we saw elevated rates of DNS failures
that seemed to increase with load on our clusters. Other developers have reported these problems in blog
posts, such as Racy conntrack and DNS lookup timeouts and a reason for unexplained connection timeouts on
Kubernetes/Docker. In these cases, a race condition in the kernel [...]