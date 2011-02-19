---
layout: post
title: ClickOnce Deployment in TeamCity
metaTitle: ClickOnce Deployment in TeamCity Issues
description: "How to fix MSB3147: Could not find required file 'setup.bin' in 'ProjectFolder\\Engine' error"
revised: 2011-02-19
date: 2009-08-29
categories: [.net,clickonce,teamcity]
migrated: true
comments: true
sharing: true
footer: true
permalink: /clickonce-deployment-in-teamcity/
summary: | 
  

---
I have been trying to get a ClickOnce VSTO add-in publishing automatically during a build and do not want visual studio on my build server.

This blog post was very useful for getting the initial build working [http://abdullin.com/journal/2009/2/17/building-vsto-solutions-without-visual-studio.html][1] it also mentions how to fix the error MSB3147: Could not find required file 'setup.bin' in 'ProjectFolder\Engine' error I was getting.

According to the post we just have to copy the Generic Bootstrapper across to our build machine and create a registry key to let .net know where to find it.

What all the resources (I have seen the same fix posted in a lot of places) fail to mention is the registry key that you need to modify if you are running a x64 system is HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\Microsoft\GenericBootstrapper\3.5.

This Wow6432Node key has got me a few times before…

  [1]: http://abdullin.com/journal/2009/2/17/building-vsto-solutions-without-visual-studio.html
