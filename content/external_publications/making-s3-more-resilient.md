---
title: 'Making S3 more resilient using Lambda@Edge'
date: 2019-02-27T00:00:00Z
draft: false
location: AWS Summit Berlin (Júlia Biró / Yann Hamon)
externalUrl: "https://aws-de-marketing.s3-eu-central-1.amazonaws.com/Field%20Marketing/Summit-Berlin-2019/Presentations/AWS_Summit_Berlin_2019_Feb27_Making_S3_more_resilient_Using_Lambda%40Edge.pdf"
thumbnail: make-s3-more-resilient.png
---

Using Lambda@Edge in CloudFront, Contentful created a solution to actively serve requests
from multiple S3 backends (circumventing an AWS restriction) to ensure resiliency to regional
outages. As the company progressed, it built tooling to adopt Lambda@Edge as a software platform
into its stack, which, at the end, allowed it to build high-performant complex logic on it.