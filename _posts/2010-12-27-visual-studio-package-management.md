---
layout: post
title: Visual Studio and Package Management
tags: [C#, CODING, NUGET, TOOLS]
category: [blog]
comments: false
#DECEMBER 27, 2010 BY PRAVESH SONI
---

All of you have used the latest and greatest open source software in your .net based applications. And you already know that it is hard to figure out the best choice, get it, set it up, run it and update it.

Recently I came across the Visual Studio 2010 extension called NuGet (formerly NuPack) which is a package management for the projects. NuGet is a free, open source developer focused package management system for the .NET platform intent on simplifying the process of incorporating third party libraries into a .NET application during development.

Let’s take ELMAH as an example. It’s a fine error logging utility which has no dependencies on other libraries, but is still a challenge to integrate into a project. These are the steps it takes:

- Find ELMAH
- Download the correct zip package.
- “Unblock” the package.
- Verify its hash against the one provided by the hosting environment.
- Unzip the package contents into a specific location in the solution.
- Add an assembly reference to the assembly.
- Update web.config with the correct settings which a developer needs to search for.

NuGet automates all these common and tedious tasks for a package as well as its dependencies. It removes nearly all of the challenges of incorporating a third party open source library into a project’s source tree. This will help developers to reduce such exercise for downloading, installing and configuring the third party software in to the application.

In upcoming posts, I’ll show how to utilize NuGet to manage open source packages in to project.
