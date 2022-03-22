---
layout: post
title: "Easy npm Package Pipeline with Workspaces"
date: 2022-03-22
categories: [technical]
tags: [c#, npm]
---

Previously I showed [how to create a pipeline for publishing a single npm package][old-npm-post]. If you have many packages to publish, it can be more efficient to publish them all together, as opposed to creating a separate pipeline for each. Here's the approach my current team took to publish all of our npm packages at once:

## Step 1: create repository

The first step is to collect all modules into one repository ("monorepo"). Here's the file structure of our repo:

- package.json
- modules folder
  - some-module-1 folder
  - some-module-2 folder
  - some-module-3 folder

### Workspaces

We use [workspaces](https://docs.npmjs.com/cli/v8/using-npm/workspaces) to simplify installing dependencies and testing. Workspaces requires npm v7 or newer (tested on npm v8.3.1, node v16.4.0). With workspaces we can run `npm install` from the repo root to install all dependencies for all modules in one node_modules folder. In addition we define a test script in the package.json in the root to allow us to run all tests for all modules via `npm run test`.

To use workspaces, add a workspaces property to package.json like this:

{% highlight javascript %}
{
  "workspaces": [
    "modules/*"
  ],
  ...
}
{% endhighlight %}

Our test script looks like this:

{% highlight javascript %}
{
  "scripts": {
    "test": "npm run test --workspaces --if-present"
  },
  ...
}
{% endhighlight %}

### Defining workspace modules

Modules are defined by adding a subfolder containing a package.json under `./modules`.

To test a module locally before publishing, you can use [`npm link`](https://docs.npmjs.com/cli/v8/commands/npm-link) to consume a local module in another local project.

## Step 2: build and test

Our build and test script is super simple thanks to workspaces:

{% highlight powershell %}
npm ci # install dependencies for all modules
npm run test # run tests for all modules
{% endhighlight %}

## Step 3: publish

Publishing is slightly more complex because of the need to authenticate to our internal package registry. Essentially we have to send a request to the registry to get an access token which then gets saved in a temporary `.npmrc` file.

In addition we need to only publish modules that have changed to avoid errors caused by version number collisions. Some solutions rely on auto-generated version numbers to get around that problem, but we chose to manually manage version numbers in [SemVer](https://semver.org/) to keep them as meaningful as possible. Here is our solution:

{% highlight powershell %}
Write-Host "Publishing"
Get-ChildItem modules -Directory | ForEach-Object {
  cd $_.FullName
  $packageName = (node -p "require('./package.json').name")
  $currentVersion = (node -p "require('./package.json').version")
  $publishedVersion = (npm view $packageName version --silent --registry $registryURL)
  Write-Host "Package: $packageName $currentVersion"
  if ($publishedVersion){
    Write-Host "Published version is $publishedVersion"
  }
  else {
    Write-Host "No published version found"
  }
  if ($currentVersion -ne $publishedVersion) {  
    Write-Host "Publishing $packageName $currentVersion"
    npm publish --registry $registryURL
  }
}
Write-Host "Done publishing"
{% endhighlight %}

In short we compare the version in each module's package.json to its published version, and if they are not the same we publish the module. Note that if the package does not exist yet in the registry, `npm view $packageName version --silent` will swallow the resulting 404 error and `$publishedVersion` will be empty as a result.

## That's it

After following the steps above you should see your packages in whichever registry you published to, and you should now be able to `npm install <your package name>` locally.

## See also

[Lerna](https://github.com/lerna/lerna) - a tool for managing more complex/larger npm package monorepos

[old-npm-post]:{{ site.baseurl }}{% post_url 2019-08-14-node-modules-for-teams %}
