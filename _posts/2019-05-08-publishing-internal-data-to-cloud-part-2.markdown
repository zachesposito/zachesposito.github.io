---
layout: post
title:  "Publishing internal data to the cloud - Part 2"
date:   2019-05-08
---

In [part 1][part-1-post] we covered how to upload a file containing order data to an S3 bucket, and how to make the bucket emit a notification after receiving the file. In part 2 we'll finish by using that notification to trigger a Lambda function that will import the data into a [RDS](https://aws.amazon.com/rds/) database.

## 4. Import order data into a database when a new file is uploaded



[part-1-post]:{{ site.baseurl }}{% post_url 2019-05-01-publishing-internal-data-to-cloud-part-1 %}