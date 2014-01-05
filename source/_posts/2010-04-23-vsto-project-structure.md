---
layout: post
title: How to Structure a new VSTO Project
metaTitle: How to Structure a new VSTO Project
description: How to Structure a new VSTO Project to make it easier to maintain and test.
revised: 2011-02-19
date: 2010-04-23
categories: [vsto,patternsandpractices]
migrated: true
comments: true
sharing: true
footer: true
permalink: /vsto-project-structure/
summary: | 
  

---
Without a good project structure you will find you cannot test any of your code and will possibly run into maintainability problems. For my user group talk last month I built a Outlook add-in that synchronises contacts and events from Facebook into outlook. I chose Facebook because it has a easy to access public API and most people are familiar with it. Over the next few weeks I will go through the process of building it, demonstrating many concepts as I go. At the end I will release the completed project.
<!-- more -->
#File -> New

![New Outlook Add-in][1]

I am targeting .NET 4.0 (Full, not client profile) and Outlook 2010, but all the concepts can easily be transferred back to 3.5 and Office 2003/2007.

![Embed Interop Types][2]

.NET 4.0 has a new feature to embed Interop types into the assembly, I disable this because it has caused me issues resolving COM types through my IoC container (not all IoC containers recognise type equivalence yet), and I also expect the user to have the Office PIA’s installed. Read more about the feature here.

Next create a Core library, this project contains all your IoC registration, type mappings (Automapper, use it, I will cover why in another post =D). Then rename Class1.cs to AddinCore.cs, AddinCore.cs will contain all of our bootstrapping code and it keeps ThisAddin.cs very small.

There are two main reasons for keeping ThisAddin.cs small.

 1. If we want to target Office 2003/2007 and 2010 and take advantage of the UI on each, we need different add-ins, because all our code is in a library we do not have code duplication.
 2. You cannot add references to VSTO projects, which makes any code within your VSTO project untestable.

So now we have:
![Solution Explorer][3]

From here you should add all application specific code into the .Core project, and the only code you should add to the actual add-in itself is UI related items like Ribbons, Custom Task Panes, Toolbars (for Office 2007/2003) and Form regions.

Now just create a test project to test our .Core project and we are off and racing. For my next post I will add IoC support into our application and show you how you can inject dependencies into Ribbons.


  [1]: /get/screenshots/NewOutlookAddin.png
  [2]: /get/screenshots/EmbedInteropTypes.png
  [3]: /get/screenshots/AddinSolutionExplorer.png