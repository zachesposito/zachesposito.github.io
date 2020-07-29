---
layout: post
title:  ".NET Database Project Lessons"
date:   2020-06-20
categories: [technical]
tags: [c#, .net framework]
---
Recently I needed to add a new [SQL Server Database Project](https://docs.microsoft.com/en-us/visualstudio/data-tools/creating-and-managing-databases-and-data-tier-applications-in-visual-studio?view=vs-2019) to an existing .NET Core API. Between creating the project itself and also setting it up in the API's continuous integration pipeline, I encountered a few unexpected gotchas:

## Explicitly name all constraints
A coworker pointed out that if you don't give a table's `PRIMARY KEY` constraint a name, then every DACPAC deployment will generate a new name for the constraint. This causes the table to be recreated which can waste a lot of time due to copying data, especially for very large tables.

Solution: instead of specifying `PRIMARY KEY` in a column definition like this:
```sql
CREATE TABLE [dbo.MyTable]
(
    [MyTableId] INT NOT NULL PRIMARY KEY IDENTITY
)
```

Define the constraint seperately and give it a name, like this:
```sql
CREATE TABLE [dbo.MyTable]
(
    [MyTableId] INT NOT NULL IDENTITY --**Don't specify PRIMARY KEY here**
    CONSTRAINT [PK_MyTable] PRIMARY KEY CLUSTERED ([MyTableId] ASC) --**Make explicit constraint instead**
)
```

## Make sure login's default database is correct
Discovered the hard way that if you misconfigure a login by setting its default database to a database for which the login's user doesn't have the `CONNECT` grant, the user will be unable to log in at all, even though they might have the `CONNECT` grant on a different database on the same server. Here's an example of my mistake:
```sql
CREATE LOGIN [MyLogin] WITH PASSWORD = 'password', DEFAULT_DATABASE=InaccessibleDatabase
GO

CREATE USER [MyUser] FOR LOGIN [MyLogin];
GO

GRANT CONNECT TO [MyUser]
GO
```

Setting the login's default database to the one they actually have access to fixes the problem.

## Build database project separately
In the API project's CI pipeline, the build configuration used the .NET Core CLI to build the entire solution. Initially I assumed that this would build the database project automatically, since the database project was part of the solution. However, I discovered that [you can't use the .NET Core CLI to build a database project](https://github.com/dotnet/sdk/issues/10441)(yet). Similarly, you instead build the solution with MSBuild, but use `dotnet publish` to output the build artifacts, the database project will be built but no database artifacts will be published because `dotnet publish` only affects .NET Core projects.

To fix, I simply had to add separate build and publish steps for the database project to the pipeline.