---
layout: post
title: ClickOnce Bootstrapping Errors
metaTitle: ClickOnce Bootstrapping Errors
description: Once again I have wasted a heap of time trying to get ClickOnce publishing on a build server. Here is how to get it working.
revised: 2011-07-21
date: 2011-07-20
categories: [TeamCity, MSBuild, .NET]
migrated: true
comments: true
sharing: true
footer: true
permalink: /clickonce-bootstrapping-errors/
summary: | 
  Once again I have wasted a heap of time trying to get ClickOnce publishing on a build server. Here is how to get it working.

---
I have hit this before, but it was VSTO related and posted about it [http://jake.ginnivan.net/clickonce-deployment-in-teamcity](http://jake.ginnivan.net/clickonce-deployment-in-teamcity)

When trying to publish my clickonce installer I am getting the error:

    [14:55:16]: [_DeploymentGenerateBootstrapper] GenerateBootstrapper
    [14:55:16]: [GenerateBootstrapper] c:\Windows\Microsoft.NET\Framework\v4.0.30319\Microsoft.Common.targets(3939, 9): warning MSB3155: Item 'Microsoft.Windows.Installer.3.1' could not be located in 'C:\TeamCity\buildAgent\work\63190f273e745a25\TrainersAdmin'.
    [14:55:16]: [GenerateBootstrapper] c:\Windows\Microsoft.NET\Framework\v4.0.30319\Microsoft.Common.targets(3939, 9): warning MSB3155: Item '.NETFramework,Version=v4.0' could not be located in 'C:\TeamCity\buildAgent\work\63190f273e745a25\TrainersAdmin'.
    [14:55:16]: [GenerateBootstrapper] c:\Windows\Microsoft.NET\Framework\v4.0.30319\Microsoft.Common.targets(3939, 9): error MSB3147: Could not find required file 'setup.bin' in 'C:\TeamCity\buildAgent\work\63190f273e745a25\ProjectName\Engine'.

If your build server is a **x64** operating system copy `'C:\Program Files (x86)\Microsoft SDKs\Windows\v7.0A\Bootstrapper\'` to the same directory on the build server, then copy the blow code into a file called FixBootstrapper.reg, then run it.

    Windows Registry Editor Version 5.00

    [HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\Microsoft\GenericBootstrapper\4.0]
    "Path"="C:\\Program Files (x86)\\Microsoft SDKs\\Windows\\v7.0A\\Bootstrapper\\"



If you have a **x86** build server, upgrade. Otherwise copy '`C:\Program Files\Microsoft SDKs\Windows\v7.0A\Bootstrapper\`' to the same directory, then copy the blow into a registry file

    Windows Registry Editor Version 5.00

    [HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\GenericBootstrapper\4.0]
    "Path"="C:\\Program Files\\Microsoft SDKs\\Windows\\v7.0A\\Bootstrapper\\"