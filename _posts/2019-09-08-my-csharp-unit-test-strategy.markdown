---
layout: post
title:  "My C# Unit Test Strategy"
date:   2019-09-06
categories: [technical]
tags: [c#, .net core]
---

//intro

//xunit and moq, why vs competitors

//project structure
  //got one file per method from dlemstra

//cover everything that could happen in a method
  //expected case
  //empty/wrong params
  //cyclomatic complexity - conditionals, etc

//run in vs or with dotnet test