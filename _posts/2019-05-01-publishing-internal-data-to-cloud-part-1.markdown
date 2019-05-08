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
In my case I made a small C# command line program that queried the ERP database via a [SQLConnection](https://docs.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlconnection?view=netcore-2.2). I'm going to skip covering the SQL code because it doesn't really matter, the point is to get a hold of the data however is necessary.

## 2. Upload the order data to S3
After getting the data the C# program transformed the result into JSON, and then uploaded the JSON to S3 using the [AWS SDK for .NET](https://aws.amazon.com/sdk-for-net/).

Here's how to upload a file to S3 using the .NET SDK:

{% highlight c# %}
using Amazon;
using Amazon.Runtime;
using Amazon.S3;
using Amazon.S3.Model;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Orders.Uploader.Models;
using Newtonsoft.Json;

namespace Orders.Uploader.Services.FileStorage
{

    public class S3FileStorageService
    {
        private IAmazonS3 S3Client { get; }
        private S3FileStorageServiceOptions Options { get; }

        /// <summary>
        /// Create an S3FileStorageService, providing the S3 client to use.
        /// </summary>
        /// <param name="options">Options for this service</param>
        /// <param name="client">An IAmazonS3 client</param>
        public S3FileStorageService(S3FileStorageServiceOptions options, IAmazonS3 client)
        {
            S3Client = client;
            Options = options;
        }

        /// <summary>
        /// Create an S3FileStorageService, implicitly creating its own AmazonS3Client.
        /// </summary>
        /// <param name="options">Options for this service</param>
        /// <param name="endpoint">Endpoint for the AWS region in which the S3 bucket exists</param>
        public S3FileStorageService(S3FileStorageServiceOptions options, RegionEndpoint endpoint)
        {
            S3Client = new AmazonS3Client(new BasicAWSCredentials(options.AccessKey, options.SecretKey), endpoint);
            Options = options;
        }

        /// <summary>
        /// Upload a collection of orders as a JSON file to S3
        /// </summary>
        /// <param name="orders">The collection of orders</param>
        public async Task UploadOrders(IEnumerable<Order> orders)
        {
            var ordersJSON = JsonConvert.SerializeObject(orders);
            
            var request = new PutObjectRequest()
            {
                ContentBody = ordersJSON,
                BucketName = Options.BucketName,
                Key = $"orders-{DateTime.Now.ToString("yyyy-MM-dd-HH-mm-ss")}.json"
            };

            try
            {
                await S3Client.PutObjectAsync(request);
            }
            catch (AmazonS3Exception e)
            {
                if (e.ErrorCode != null &&
                    (e.ErrorCode.Equals("InvalidAccessKeyId") ||
                    e.ErrorCode.Equals("InvalidSecurity")))
                {
                    throw new Exception("Could not upload order data to S3. Make sure AWS credentials exist and are correct.", e);
                }
                else
                {
                    throw new Exception($"Could not upload order data to S3: {e.Message}", e);
                }
            }            
        }
    }
}
{% endhighlight %}

The key piece is the `UploadOrders` method, which creates a `PutObjectRequest` and executes that with the `S3Client`'s `PutObjectAsync` method. `Options.BucketName` is set to the name of an S3 bucket I created, for example `orders-bucket`. 

`Options.AccessKey` and `Options.SecretKey` come from an [IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) I made specifically for this uploader that was created with programmatic access. That user has a [policy](https://docs.aws.amazon.com/transfer/latest/userguide/users-policies-all-access.html) attached that only allows it to access the `orders-bucket` (the linked policy instructions are in the context of AWS Transfer for SFTP but they apply even though this solution doesn't use Transfer).

## 3. Make S3 send a message to an SNS topic
Next I [created a new SNS topic](https://docs.aws.amazon.com/sns/latest/dg/sns-tutorial-create-topic.html) named something like `new-order-file-notifications`. The topic has an access policy that only allows the `orders-bucket` to publish messages:

{% highlight json %}
{
  "Version": "2008-10-17",
  "Id": "__default_policy_ID",
  "Statement": [
    {
      "Sid": "__default_statement_ID",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:<region>:<AWS account number>:new-order-file-notifications",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::orders-bucket"
        }
      }
    }
  ]
}
{% endhighlight %}

Then I went back to the `orders-bucket` in S3 and on the Properties tab for the bucket, configured a new event that sent a message to the `new-order-file-notifications` topic after a PUT request:
![S3 event configuration screen](/static/img/s3-event.png)

Now when a new JSON file containing order data is uploaded to the `orders-bucket`, a message is sent to the SNS topic. In [part 2][part-2-post] I will show how to create a Lambda function that subscribes to the SNS topic and imports the latest order data into a database when it receives a new SNS message.

[part-2-post]:{{ site.baseurl }}{% post_url 2019-05-08-publishing-internal-data-to-cloud-part-2 %}