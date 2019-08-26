---
layout: post
title:  "Using Little CMS on Azure Batch"
date:   2019-05-19
subtitle: Scaling product visualization
categories: [technical]
tags: [c#, azure]
---

An important part of selling stuff online is product visualization - allowing a shopper to see the product. A common component of product visualization is showing a product in all available colors. For example, if a t-shirt can be made in black or white, it's important to show an image of each color option so that a shopper can see whether the shirt design looks good over both colors.

I had a project recently where there was a need to visualize lots of products in dozens of colors. There were too many combinations of product and color to create all the necessary images by hand, so an automatic coloring solution was created that used [Little CMS](http://www.littlecms.com/) to apply color transforms expressed as [ICC profiles](https://en.wikipedia.org/wiki/ICC_profile). The solution worked but it needed to be scaled up to be able to color all the product images in a reasonable amount of time.

## Migrating to Azure Batch
To scale up the coloring solution (let's call it the Colorizer), I initially tried running it in an [Azure function](https://azure.microsoft.com/en-us/services/functions/). However the [memory limit on the Consumption plan is 1.5 GB](https://docs.microsoft.com/en-us/azure/azure-functions/functions-scale#service-limits), and the Colorizer used more than that since the product images were very large. Also I didn't want to use the App Service plan because the function didn't need to be always on, and the Premium plan hadn't been released yet.

Instead of Azure functions, I decided to use [Azure Batch](https://azure.microsoft.com/en-us/services/batch/). Batch seemed like a perfect fit because it supports running large parallel jobs on demand. Here is a short summary of a Batch workflow:
1. Create a pool, which is essentially a group of VMs
2. Create a job, which is a collection of tasks that run on a pool
3. Add tasks to the job, which Batch will distribute across the available VMs in the pool and run
4. Wait for all tasks to complete
5. Delete the job and the pool

Here's a C# client I wrote to use Batch, adapted from [this Microsoft example](https://github.com/Azure-Samples/azure-batch-samples/tree/master/CSharp/GettingStarted/01_HelloWorld):

{% highlight c# %}
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.Azure.Batch;
using Microsoft.Azure.Batch.Common;

public class AzureBatchColorizationService : IBatchColorizationService
{
    private BatchClient BatchClient { get; set; }
    private int NodeCount = 100;
    private string ApplicationName = "colorize";
    private string ApplicationVersion = "1";
    private string VMSeriesID = "Standard_D4_v3";
    private string VMSKUID = "batch.node.ubuntu 16.04";

    public AzureBatchColorizationService(AzureBatchColorizationServiceOptions options)
    {
        //Initialize Batch client
        BatchClient = BatchClient.Open(new Microsoft.Azure.Batch.Auth.BatchSharedKeyCredentials(/*your-batch-base-url*/, /*your-batch-account-name>*/, options.SharedKey));
    }

    //Main method for coloring
    //ColorizationRequest is a custom type that has a Key for creating a unique task ID and a Path to the image to color
    public async Task ColorizeAssetsAsync(IEnumerable<ColorizationRequest> requests)
    {
        var jobID = $"pvam-colorizer-{DateTime.Now.ToString("yyyyMMdd-HHmmss")}";
        try
        {
            Console.WriteLine("Submitting job");
            await SubmitJobAsync(jobID);

            Console.WriteLine("Adding tasks to job");
            //CloudTask takes a console command to run, in this case it's using the Ubuntu system shell to run the Colorizer application, uploaded in the Batch UI in the Azure portal
            var tasks = requests
                .Select(r => new CloudTask($"task-{r.Key}", $"/bin/sh -c '$AZ_BATCH_APP_PACKAGE_colorize_1/colorizer -i {r.Path}'"));
            await BatchClient.JobOperations.AddTaskAsync(jobID, tasks, new BatchClientParallelOptions
            {
                MaxDegreeOfParallelism = NodeCount
            });

            Console.WriteLine("Waiting for job to complete");
            await WaitForJobAndPrintOutputAsync(jobID);
        }
        catch (Exception e)
        {
            Console.WriteLine($"**Error during batch colorizing for job {jobID}: {e.Message}");
        }
        finally
        {
            Console.WriteLine($"Deleting job {jobID}");
            await BatchClient.JobOperations.DeleteJobAsync(jobID);
        }
    }

    private async Task SubmitJobAsync(string jobId)
    {
        // create an empty unbound Job
        CloudJob unboundJob = BatchClient.JobOperations.CreateJob();
        unboundJob.Id = jobId;

        var imageReference = new ImageReference(
            publisher: "microsoft-azure-batch",
            offer: "ubuntu-server-container",
            sku: "16-04-lts",
            version: "latest");
        // For this job, ask the Batch service to automatically create a pool of VMs when the job is submitted.
        unboundJob.PoolInformation = new PoolInformation()
        {
            AutoPoolSpecification = new AutoPoolSpecification()
            {
                AutoPoolIdPrefix = "Colorize",
                PoolSpecification = new PoolSpecification()
                {
                    TargetDedicatedComputeNodes = NodeCount,
                    VirtualMachineConfiguration = new VirtualMachineConfiguration(imageReference, VMSKUID),
                    VirtualMachineSize = VMSeriesID,
                    //Specify an application to pre-load onto each node in the pool:
                    ApplicationPackageReferences = new List<ApplicationPackageReference> { new ApplicationPackageReference { ApplicationId = ApplicationName, Version = ApplicationVersion } }
                },
                KeepAlive = false,
                //**Important**: limit the pool lifetime to the job lifetime, otherwise after the job's done the pool will stay allocated and accrue charges
                PoolLifetimeOption = PoolLifetimeOption.Job
            }
        };

        // Commit Job to create it in the service
        await unboundJob.CommitAsync();
    }

    private async Task WaitForJobAndPrintOutputAsync(string jobID)
    {
        Console.WriteLine("Waiting for all tasks to complete on job: {0} ...", jobID);

        // We use the task state monitor to monitor the state of our tasks -- in this case we will wait for them all to complete.
        TaskStateMonitor taskStateMonitor = BatchClient.Utilities.CreateTaskStateMonitor();

        List<CloudTask> ourTasks = await BatchClient.JobOperations.ListTasks(jobID).ToListAsync();

        // Wait for all tasks to reach the completed state.
        // If the pool is being resized then enough time is needed for the nodes to reach the idle state in order
        // for tasks to run on them.
        await taskStateMonitor.WhenAll(ourTasks, TaskState.Completed, TimeSpan.FromHours(8));

        // dump task output
        foreach (CloudTask t in ourTasks)
        {
            Console.WriteLine($"Task {t.Id} done.");
        }
    }
}
{% endhighlight %}