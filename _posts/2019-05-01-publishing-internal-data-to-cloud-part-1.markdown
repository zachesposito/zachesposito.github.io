---
layout: post
title:  "Publishing internal data to the cloud - Part 1"
date:   2019-05-01
---

Sometimes an organization might have some private data stored on-premises, and a need arises to share some of that data publicly. In my case there was a need to give customers a way to check the status of their orders without logging in. To support this the order status data needed to be pulled from an internal [ERP](https://en.wikipedia.org/wiki/Enterprise_resource_planning) system and published to a website running in the cloud on [AWS](https://aws.amazon.com/).

There are some network-based ways to share internal data with cloud systems, for example [AWS Direct Connect](https://aws.amazon.com/directconnect/). However in my case I didn't have access to make those kinds of network changes, and also there was a desire to limit internal network exposure as much as possible. Instead of connecting the internal data to the cloud systems that needed it via networking changes, I opted for a file-based approach using [S3](https://aws.amazon.com/s3/) and [Lambda](https://aws.amazon.com/lambda/).

Here's the outline of the steps that take the order data from the internal ERP system to the cloud database used by the website:
1. Get the order data by querying the ERP's database on the internal network
2. Upload the order data to an S3 bucket as a JSON file
3. Upon receiving a new file, the S3 bucket sends a message to an [SNS](https://aws.amazon.com/sns/) topic
4. A Lambda function listens to the SNS topic, and upon receiving a new message that a JSON file has been uploaded to S3, reads the data from that file and imports it into the website's database

Let's dig into each step:
## 1. Get the order data
 In my case I made a small C# command line program that queried the ERP database via a [SQLConnection](https://docs.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlconnection?view=netcore-2.2), transformed the result into JSON, and then uploaded the JSON to S3 using the [AWS SDK for .NET](https://aws.amazon.com/sdk-for-net/).