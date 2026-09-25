---
title: Visual Studio and Package Management
date: 2010-12-27
slug: visual-studio-and-package-management 
summary: "An introduction to NuGet, then known as NuPack, and how package management can simplify the process of adding third-party libraries to .NET applications. Using ELMAH as an example, the post illustrates the manual steps developers previously had to perform and how package management streamlines them."
---

{{< image-align-right >}}![NuGet](/assets/images/nuget.png){{</ image-align-right >}} All of you have used the latest and greatest open source software in your .net based applications. And you already know that it is hard to figure out the best choice, get it, set it up, run it and update it.

Recently I came across the Visual Studio 2010 extension called NuGet (formerly NuPack) which is a package management for the projects. NuGet is a free, open source developer focused package management system for the .NET platform intent on simplifying the process of incorporating third party libraries into a .NET application during development.

Let’s take ELMAH as an example. It’s a fine error logging utility which has no dependencies on other libraries, but is still a challenge to integrate into a project. These are the steps it takes:

1. Find ELMAH
2. Download the correct zip package.
3. “Unblock” the package.
4. Verify its hash against the one provided by the hosting environment.
5. Unzip the package contents into a specific location in the solution.
6. Add an assembly reference to the assembly.
7. Update web.config with the correct settings which a developer needs to search for.

NuGet automates all these common and tedious tasks for a package as well as its dependencies. It removes nearly all of the challenges of incorporating a third party open source library into a project’s source tree. This will help developers to reduce such exercise for downloading, installing and configuring the third party software in to the application.

In upcoming posts, I’ll show how to utilize NuGet to manage open source packages in to project.
