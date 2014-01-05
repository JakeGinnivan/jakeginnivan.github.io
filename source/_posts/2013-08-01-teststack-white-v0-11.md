---
layout: post
title: TestStack.White v0.11 Released!
metaTitle: TestStack.White v0.11 Released!
description: White is pushing towards v1, this is a bugfix release of White
revised: 2013-08-01
date: 2013-08-01
categories: [TestStack, Testing, White, UI Automation]
migrated: true
comments: true
sharing: true
footer: true
permalink: /teststack-white-v0-11/
summary: | 
  White is pushing towards v1, this is a bugfix release of White

---
I have just pushed the button for TestStack.White v0.11.

The main focus between v0.10 and v0.11 is converting the old test suite into a new test suite which is easier to maintain and can reliably run on the build server.

The previous test suite was often red, which meant that it was hard to know if there were regressions as other issues were fixed.

You can see the CI status at [http://teamcity.ginnivan.net/project.html?projectId=TestStack_White](http://teamcity.ginnivan.net/project.html?projectId=TestStack_White&branch_TestStack_White=__all_branches__)
<!-- more -->
## TestStack.White.ScreenObjects
Also released is [https://www.nuget.org/packages/TestStack.White.ScreenObjects](https://www.nuget.org/packages/TestStack.White.ScreenObjects)

This is the old White.Repository project, finally released on NuGet. Hopefully there should be some updates to this project coming up as well!

## Namespace Change
Being part of TestStack now, we wanted to bring White's namespace into line with the other TestStack projects.

White's namespace has changed from `White.Core` to `TestStack.White`. Once you upgrade just run `Fix-WhiteNamespaces` from your NuGet console and we will fix all your namespace references for you!

## Change Log
The change log is available at [http://teststack.azurewebsites.net/White/ChangeLog.html](http://teststack.azurewebsites.net/White/ChangeLog.html)

You may notice the website (which we will have a domain for very shortly), this is the new TestStack documentation site/wiki.

Please have a look around, post comments, contribute and give us feedback! 

## Pull Requests
Whenever you submit a pull request for White, we will automatically do a CI build, then a full UI Test run. 
This means if you do not want to wait for the test suite to run on your machine, you can just submit your pull request, then wait for the status to be reported back (which sometimes fails for some reason, but you should be able to see it on the build server).

## Whats next
The next step for White is upgrading to v3 of the UIA Library, this will likely break a few things (which is why it was important to get the tests running properly).

## Reporting Issues
If you find an issue in White, create an issue on github, and even better, create a failing UI tests. I do not mind pull requests with a failing test, I can fix the underlying issue without you getting involved in the inner workings of White.