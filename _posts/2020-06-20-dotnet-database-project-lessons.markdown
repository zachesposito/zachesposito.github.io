---
layout: post
title:  ".NET Database Project Lessons"
date:   2020-06-20
categories: [technical]
tags: [c#, .net framework]
---
Recently I needed to add a new (SQL Server Database Project)[https://docs.microsoft.com/en-us/visualstudio/data-tools/creating-and-managing-databases-and-data-tier-applications-in-visual-studio?view=vs-2019] to an existing .NET Core API. Between creating the project itself and also setting it up in the API's continuous integration pipeline, I encountered a few unexpected gotchas:

## Explicitly name all constraints

## Make sure login's default database is correct

## Build database project separately

## Parameterize password in deployment pipeline
