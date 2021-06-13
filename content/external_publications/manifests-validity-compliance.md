---
title: 'Ensuring Kubernetes manifests validity & compliance - a tooling overview'
date: 2020-01-26T23:11:13
draft: false
externalUrl: "https://github.com/yannh/you-should-test-your-kubernetes-manifests/blob/main/cncf-berlin-meetup-20201218.pdf"
thumbnail: test-kubernetes-manifests.png
---

In an Infrastructure as code / GitOps world, testing that your Kubernetes configuration is correct,
secure, and compliant to your company's requirements &amp; best practices is more important than
ever. An increasingly large list of tools is there to help you - linters, validators, testing
frameworks, admission controllers... each working in subtly different ways.

To help you navigate these waters, I will present some of the most common tools for Kubernetes
manifests validation &amp; compliance testing, detail their use, limitations and provide some
usage examples.

I will also introduce Kubeconform, a new Kubernetes validation tool I authored.