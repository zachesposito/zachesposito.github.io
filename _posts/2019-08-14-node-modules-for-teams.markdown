---
layout: post
title:  "Creating and sharing Node modules in a team"
date: 2019-08-14
---

In my [guide to creating NuGet packages for teams][nuget-post], I showed how to create a NuGet package and host it on a private NuGet feed using Azure DevOps, with the goal of enabling greater code reuse. Here's how to do the same thing with Node.js modules:

## Contents
* [Create a Node module](#create-a-node-module)
* [Create DevOps project and push code](#create-devops-project-and-push-code)
* [Create publish pipeline](#create-publish-pipeline)
* [Consume feed](#consume-feed)
* [Use module](#use-module)
* [Releasing updates](#releasing-updates)
* [Conclusion](#conclusion)

## Create a Node module
![Module files](/static/img/ice-cream-library-node-files.png)

As an example I re-implemented the Ice Cream Library from [the NuGet guide][nuget-post-create-module] as a Node module. You can [clone it here](https://dev.azure.com/zachesposito-devops-example/_git/ice-cream-library-node).

## Create a DevOps project and push code

[Follow the same steps as shown in the NuGet guide][nuget-post-create-devops].

## Create publish pipeline

[Follow the same steps as shown in the NuGet guide][nuget-post-create-pipeline] to select the `ice-cream-library-node` Git repository and the **Starter pipeline** template. On the **Review your pipeline YAML** step, replace the existing YAML with the following:

{% highlight yaml %}
trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: NodeTool@0
  inputs:
    versionSpec: '10.x'
  displayName: 'Install Node.js'

- script: |
    npm install
  displayName: 'npm install'

- script: |
    npm test
  displayName: 'npm test'

- task: Npm@1
  displayName: 'npm publish'
  inputs:
    command: publish
    publishRegistry: useFeed
    publishFeed: 'zachesposito-devops-example'
{% endhighlight %}

The `trigger` and `pool` definitions are the same as [the YAML from the Nuget guide][nuget-post-yaml]. Here's an outline of the steps in the above YAML:

### Install Node.js

Install Node on the build machine using the [Node installer task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/tool/node-js?view=azure-devops).

### npm install

Uses the [Command Line task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/command-line?view=azure-devops&tabs=yaml) to install dependencies.

### npm test

Also uses the Command Line task to run tests (test script defined in `package.json`).

### npm publish

Publishes this module to the private Artifacts feed. Note that this is the same feed that was used in the [NuGet guide][nuget-post]. An Azure Artifacts feed can contain multiple types of packages, in this case both NuGet packages and Node modules.

If the publishing step fails, try giving the project's build account access to the Artifacts feed as described in the [Troubleshooting section of the NuGet guide][nuget-post-troubleshooting]. 

## Consume feed

To make your development environment aware of the Artifacts feed so that you can `npm install` packages from it, follow these steps:

### Add feed URL to .npmrc

On the Artifacts tab in the DevOps project, click **Connect to feed** and select **npm** on the left side:

![npm connection instructions](/static/img/ice-cream-library-node-auth.png)

Copy the contents of the first textbox and paste them into the `.npmrc` file located in your user folder. If `.npmrc` does not exist, create it.

### Authenticate to feed

If developing on Windows, install the VSTS npm authentication helper tool: `npm install -g vsts-npm-auth`. Then, run the authentication helper to automatically add credentials to the .npmrc file: `vsts-npm-auth -config path-to-your\.npmrc`

If developing on another OS, [generate a token](https://docs.microsoft.com/en-us/azure/devops/artifacts/npm/npmrc?view=azure-devops&tabs=windows#linux-or-mac) to use in `.npmrc`.

## Use module

Create another Node module to use the `ice-cream-library-node` module. In my case I reimplementated the very brief [DessertBar project from the NuGet guide][nuget-post-dessert-bar]. After creating `package.json`, use `npm install ice-cream-library-node --save` to get the ice cream library module from the DevOps Artifacts feed.

After installing the ice cream library module, you can use it in code. Here's `index.js` from the Node version of DessertBar:

{% highlight javascript %}
const IceCreamServer = require('ice-cream-library-node');

const iceCreamService = new IceCreamServer();
const serving = iceCreamService.serve(2, 'chocolate');
console.log(`Served ${serving.scoops} scoops of ${serving.flavor} ice cream.`);
{% endhighlight %}

Running `node index.js` should output `Served 2 scoops of chocolate ice cream.`

## Releasing updates

[Just like with NuGet packages][nuget-post-updates], to release new version, simply update the `version` property in `package.json` and then push a commit to the master branch in the DevOps repo. That will trigger the pipeline again and release a new version with the number you specified.

## Conclusion

The above example showed how to create a simple Node module and publish it to a private Azure Artifacts feed using an automatic pipeline, and how to consume the module in another Node module. Now you can follow this pattern to make small, reusable Node modules for your team.


[nuget-post]:{{ site.baseurl }}{% post_url 2019-07-23-nuget-packages-for-teams %}
[nuget-post-create-module]:{{ site.baseurl }}{% post_url 2019-07-23-nuget-packages-for-teams %}#create-a-net-standard-class-library
[nuget-post-create-devops]:{{ site.baseurl }}{% post_url 2019-07-23-nuget-packages-for-teams %}#create-devops-project-and-push-code
[nuget-post-create-pipeline]:{{ site.baseurl }}{% post_url 2019-07-23-nuget-packages-for-teams %}#create-publish-pipeline
[nuget-post-yaml]:{{ site.baseurl }}{% post_url 2019-07-23-nuget-packages-for-teams %}#trigger
[nuget-post-troubleshooting]:{{ site.baseurl }}{% post_url 2019-07-23-nuget-packages-for-teams %}#troubleshooting
[nuget-post-dessert-bar]:{{ site.baseurl }}{% post_url 2019-07-23-nuget-packages-for-teams %}#use-package
[nuget-post-updates]:{{ site.baseurl }}{% post_url 2019-07-23-nuget-packages-for-teams %}#releasing-updates
