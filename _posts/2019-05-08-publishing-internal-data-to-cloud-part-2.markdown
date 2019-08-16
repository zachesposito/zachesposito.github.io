---
layout: post
title:  "Publishing internal data to the cloud - Part 2"
date:   2019-05-08
subtitle: Automating data flow to AWS using S3 and Lambda
---

In [part 1][part-1-post] we covered how to upload a file containing order data to an S3 bucket, and how to make the bucket emit a notification after receiving the file. In part 2 we'll finish by using that notification to trigger a Lambda function that will import the data into a [RDS](https://aws.amazon.com/rds/) database.

## 4. Import order data into a database when a new file is uploaded
### Make a new Lambda function
In the Lambda service in the AWS console choose **Create function**:
* Author from scratch
* **Function name** - choose `order-import-function`
* **Runtime** - Node.js 10.x
* **Permissions > Execution role** - choose **Create new role from AWS policy templates**, give the role a name, and then choose at least the **Amazon S3 object read-only permissions** template (my real function had additional permissions related to VPC access but I'm omitting the setup that for brevity)

### Trigger on SNS
Once the function is created, then on the function configuration screen you can choose SNS as a trigger from the list on the left. On the **Configure triggers** card enter the [ARN](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html) of the SNS topic created in [part 1][part-1-post]. 

### Write Lambda
Here's the Node.js Lambda function that reads in the order JSON from S3 and inserts it into a SQL Server database:
{% highlight javascript %}
const AWS = require('aws-sdk');
const sql = require('mssql');
const parser = require('./parser');

const insertOrders = async (orders) => {
    const sqlconfig = {
        user: process.env.sqlUser,
        password: process.env.sqlPass,
        server: process.env.rdsEndpoint,
        database: process.env.databaseName,
        port:2429,
        abortTransactionOnError: true
    };
    var pool;
    
    await (async () => {
        try {
            sql.close();
            pool = await sql.connect(sqlconfig);
            const transaction = new sql.Transaction(pool);
            
            let rolledBack = false;
            transaction.on('rollback', aborted => {
                rolledBack = true;
            });
            
            try {
                await transaction.begin();
            }
            catch (e){
                throw new Error(`Could not begin transaction: ${e}`);
            }            
            
            const dropRequest = new sql.Request(transaction);
            await dropRequest.query("IF OBJECT_ID('dbo.Orders', 'U') IS NOT NULL BEGIN drop table orders END");

            const table = new sql.Table('Orders');
            table.create = true;
            table.columns.add('OrderStatus', sql.NVarChar(100), {nullable: false});
            table.columns.add('OrderNumber', sql.NVarChar(100), {nullable: true});
            table.columns.add('OrderDate', sql.NVarChar(100), {nullable: true});
            for (let order of orders){
                table.rows.add(
                    order.OrderStatus,
                    order.OrderNumber,
                    order.OrderDate
                );
            }
                        
            const bulkRequest = new sql.Request(transaction);
            try {
                await bulkRequest.bulk(table);
            }
            catch (e) {
                if (!rolledBack) {
                    try {
                        await transaction.rollback();
                    }
                    catch (rollbackError){
                        throw new Error(`Error during bulk insert and transaction not rolled back: ${rollbackError}`);
                    }
                }
                throw new Error(`Error during bulk insert and transaction rolled back: ${e}`);
            }
            
            try {
                await transaction.commit();
            }
            catch (e){
                throw new Error(`Error during transaction commit: ${e}`);
            }
            
        } catch (err) {
            throw new Error(`SQL error: ${err}. Log: ${reducedMessages}`);
        }
        finally {
            pool.close();
            sql.close();
        }
    })();
};

const getS3JSON = async (bucket, key) => {
    var s3 = new AWS.S3();
    var params = {Bucket: bucket, Key: key};
    var response = await s3.getObject(params).promise();
    return response.Body.toString();
};

exports.handler = async (event) => {
    var eventMessage = JSON.parse(event.Records[0].Sns.Message);
    var key = eventMessage.Records[0].s3.object.key;
    try {
        var json = await getS3JSON('orders-bucket', key);
        var orders = parser.parseOrders(json); 
        await insertOrders(orders);
    }
    catch (e){
        console.log(e);
    }
};
{% endhighlight %}

Dependencies:
* `aws-sdk` - [AWS SDK for JavaScript](https://github.com/aws/aws-sdk-js)
* `mssql` - [SQL Server client for Node](https://github.com/tediousjs/node-mssql)
* `./parser` is a utility module that parses the orders JSON and makes sure that it's an array.

On the **Environment variables** card create these variables with these keys, filling in the values as appropriate for your database:
* `databaseName`
* `rdsEndpoint`
* `sqlPass`
* `sqlUser`

`exports.handler` is the function that runs when the Lambda is triggered. This gets the name of the newly uploaded S3 file, gets the JSON contents of that file, parses the JSON back into an array of order objects, then sends the orders to the `insertOrders` function. 

`insertOrders` uses `mssql` to begin a transaction, drop the orders table if it already exists in the database, then add a new orders table that has a row for each order. If any errors are encountered it tries to rollback the transaction.


[part-1-post]:{{ site.baseurl }}{% post_url 2019-05-01-publishing-internal-data-to-cloud-part-1 %}