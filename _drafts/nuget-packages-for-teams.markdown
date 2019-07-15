---
layout: post
title:  "Creating and sharing NuGet packages in a team"
---

There is always more development work to do than there is time to do it. This year I have been focusing on improving my development efficiency. Specifically, one of my goals has been to take advantage of more opportunities to reuse code.

In my JavaScript work I've encountered many useful modules from [Sindre Sorhus](https://github.com/sindresorhus/). Wondering how he is able to be so productive led me to [this post](https://blog.sindresorhus.com/answering-anything-678ce5623798#8513), where he explains that one of his tricks is to write [small, focused modules](https://github.com/sindresorhus/ama/issues/10#issuecomment-117766328) to avoid ever solving the same problem more than once. This approach not only saves time, but also leads to better code organization since multiple solutions for the same problem, e.g. file storage, aren't scattered across projects.

Here's how I've implemented small, focused modules for C# for my team in the form of NuGet packages using Azure DevOps, using an example project called [Ice Cream Library](https://dev.azure.com/zachesposito-devops-example/_git/ice-cream-library).

## Create a .NET Standard class library
![Ice cream library files](/static/img/ice-cream-library-files.png)

First, create a package to share. For maximum compatibility with .NET projects, make it a [.NET Standard class library](https://docs.microsoft.com/en-us/dotnet/core/tutorials/library-with-visual-studio). Be sure to [initialize a Git repo](https://www.atlassian.com/git/tutorials/setting-up-a-repository/git-init) in the solution directory.

In this case I created a super-simple `IceCreamService` that provides a `Serving` of ice cream given a number of scoops and a `Flavor`:

{% highlight c# %}
namespace IceCreamLibrary
{
    public class IceCreamService : IIceCreamService
    {
        public Serving Serve(int scoops, Flavor flavor)
        {
            return new Serving
            {
                Scoops = scoops,
                Flavor = flavor
            };
        }
    }
}
{% endhighlight %}

Notice that `IceCreamService` implements the `IIceCreamService` interface. It's important to provide interfaces for shareable packages so that consumers can depend on the interface instead of your concrete implementation, which will enable them to mock your package so they can write unit tests for their code that depends on it.

## Add NuGet package info

In the `.csproj` file, in the `PropertyGroup` element, add the following required metadata:
* PackageId - the name of the package, unique to the feed it will be served from
* Version
* Author

{% highlight xml %}
<PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <PackageId>IceCreamLibrary</PackageId>
    <Version>1.0.0</Version>
    <Authors>Zach Esposito</Authors>
    <Description>Serve ice cream with ease</Description>
    <PackageProjectUrl>https://dev.azure.com/zachesposito-devops-example/ice-cream-library</PackageProjectUrl>
    <RepositoryUrl>https://dev.azure.com/zachesposito-devops-example/_git/ice-cream-library</RepositoryUrl>
    <PackageReleaseNotes>Initial release</PackageReleaseNotes>
  </PropertyGroup>
{% endhighlight %}

[See here](https://docs.microsoft.com/en-us/dotnet/core/tools/csproj#nuget-metadata-properties) for a list of other metadata you can add, such as `RepositoryUrl` or `PackageReleaseNotes`. If using Visual Studio, you can also provide metadata via the **Package** tab in the project's **Properties** screen:

![Ice cream package metadata](/static/img/ice-cream-package-metadata.png)

## Create DevOps project and push code

Create a new project in Azure DevOps, then in the **Repos** tab click **Clone**:

![Ice cream package repo](/static/img/ice-cream-repo.png)

[Add the clone URL as a remote](https://help.github.com/en/articles/adding-a-remote) in the Ice Cream Library's local repo and [push the code to it](https://www.atlassian.com/git/tutorials/syncing/git-push).

## Verify Artifacts feed exists
In DevOps, under the **Artifacts** tab, a default feed named after your organizaztion should already exist. If not, click **+ New Feed** at the top to create a feed.

![DevOps feed](/static/img/ice-cream-feed.png)

## Create publish pipeline

In DevOps, under the **Pipelines** tab, click **Builds**, then **New pipeline**.

On the **Where is your code?** step, choose **Azure Repos Git**:

![DevOps new pipeline step 1](/static/img/ice-cream-pipeline-1.png)

Then select the Ice Cream Library's repo:

![DevOps new pipeline step 2](/static/img/ice-cream-pipeline-2.png)

On the **Configure your pipeline** step, choose **Starter pipeline**:

![DevOps new pipeline step 3](/static/img/ice-cream-pipeline-3.png)

On the **Review your pipeline YAML** step, replace the existing YAML with the following:

{% highlight yaml %}
trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: DotNetCoreCLI@2
  displayName: 'dotnet build'
  inputs:
    projects: '**/*.csproj'
    
- task: DotNetCoreCLI@2
  inputs:
    command: 'test'
    projects: '**/*[Tt]ests/*.csproj'
    arguments: '--configuration $(BuildConfiguration) --no-restore'

- task: DotNetCoreCLI@2
  displayName: 'dotnet pack'
  inputs:
    command: pack
    packagesToPack: source/IceCreamLibrary/IceCreamLibrary.csproj

- task: DotNetCoreCLI@2
  displayName: 'dotnet push'
  inputs:
    command: push
    publishVstsFeed: 'zachesposito-devops-example'
{% endhighlight %}

Let's walk through each part of the build definition:

### [trigger](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema#push-trigger)

Says to run this pipeline whenever commits are pushed to the master branch.

### [pool](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema#pool)

Configures the environment in which the pipeline will run. In this case the operating system will be Ubuntu.

### [steps](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema#steps)

Defines a series of .NET Core CLI tasks to run:

#### [dotnet build](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/dotnet-core-cli?view=azure-devops#build)

Builds all projects, specified via wildcards.

#### [dotnet test](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/dotnet-core-cli?view=azure-devops#test)

Runs all tests, assuming that any test projects' names end in "Tests". The `--no-restore` option is used to disable restoring NuGet packages again, since they were restored in the previous step during `dotnet build`.

#### [dotnet pack](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/dotnet-core-cli?view=azure-devops#pack)

Create a NuGet package from the specified project.

#### [dotnet push](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/dotnet-core-cli?view=azure-devops#push)

Upload the NuGet package created in the previous step to the Artifacts feed.

//yaml - assume version by csproj
//undo bad yaml commits
//had to make sure project-specific build service account was in Project Collection Build Service Accounts, and that that group had Contributor on feed

## Consume feed

## Use package