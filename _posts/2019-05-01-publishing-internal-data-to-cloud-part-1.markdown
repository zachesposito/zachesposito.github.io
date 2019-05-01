---
layout: post
title:  "Publishing internal data to the cloud - Part 1"
date:   2019-05-01
---

Sometimes an organization might have some private data stored on-premises, and a need arises to share some of that data publicly. In my case there was a need to give customers a way to check the status of their orders without logging in. To support this the order status data needed to be pulled from an internal [ERP](https://en.wikipedia.org/wiki/Enterprise_resource_planning) system and published to a website running in the cloud on [AWS](https://aws.amazon.com/).

There are some network-based ways to share internal data with cloud systems, for example [AWS Direct Connect](https://aws.amazon.com/directconnect/). However in my case I didn't have access to make those kinds of network changes, and also there was a desire to limit internal network exposure as much as possible. Instead of connecting the internal data to the cloud systems that needed it via networking changes, I opted for a file-based approach using [S3](https://aws.amazon.com/s3/) and [Lambda](https://aws.amazon.com/lambda/).

