---
layout: post
title: TeamCity Build Results in TFS
metaTitle: TeamCity Build Results in TFS
description: 
revised: 2013-04-13
date: 2013-04-13
categories: []
migrated: true
comments: true
sharing: true
footer: true
permalink: /teamcity-build-results-in-tfs/
summary: | 
  

---
# Why?
Recently I have been working with testers on our team, and trying to bring testing closer to our sprints. We use Microsoft Test Manager for our test tool, TFS for work items, GitHub for source control and TeamCity for our build server.

All our tests are written using xUnit.net. When you start working with testers it becomes important to surface your automated and manual testing together, so testers can work more efficiently and have more confidence in the software the team is producing.

Enter TFS. TFS is a great product that starts really value adding when you use different components together, the testing tools is an example of this. When you use Team Build with the testing tools, a heap of features and workflows start working way better! 

# What?
My goal is to have my automated regression tests (both UI and Integration tests) which are written in xUnit to report their results back into TFS (Builds) and Test Manager (Test Runs).

This allows the Testers to use the planned automation features in TFS to mark certain Test Cases as 'Planned' for automation, it also lights up a whole lot of reporting around test case readiness and other things.

# How?
There are a few parts to automatically marking test cases as passed/failed.

1. Associated Automation  
If you open a test case up in Visual Studio, you can assign an MSTest test to a test case  
![TeamCityBuildResultsinTFS1](/assets/posts/2013-04-13-teamcity-build-results-in-tfs/TeamCityBuildResultsinTFS1_635014894026562500.png)
2. Get a .trx file which can be published to TFS (By using VSTest.Console with the TeamCityAndTrx logger)
![TeamCityBuildResultsinTFS2](/assets/posts/2013-04-13-teamcity-build-results-in-tfs/TeamCityBuildResultsinTFS2_635014894036562500.png)  
![TeamCityBuildResultsinTFS3](/assets/posts/2013-04-13-teamcity-build-results-in-tfs/TeamCityBuildResultsinTFS3_635014894039375000.png)
3. Publish the build using TfsBuildResultPublisher (see [BuildAndPublish.msbuild](https://github.com/JakeGinnivan/TfsBuildResultPublisher/blob/master/BuildAndPublish.msbuild) for an example)
4. A test run created in Test Manager which can be linked to a test suite, this will automatically pass/fail a test. 

I have built a few projects which glue this all together.

## 1. VSTest.TeamCityLogger
The first step is I have xUnit tests, luckily Visual Studio 2012 has a new test runner, this introduces [Unit Test Adapters](http://msdn.microsoft.com/en-us/library/hh598952.aspx) to support third party test frameworks. 

The new test runner has a command line version too called VSTest.Console.exe which allows you to specify which logger to use, out of the box there is a `trxLogger` to output a .trx file for any framework.

VSTest.TeamCityLogger introduces two new loggers `TeamCityLogger` and `TeamCityAndTrxLogger`, the first simply exposes VSTest results to TeamCity, the second also outputs a .trx file.

Now we have a TeamCity build, which exports a .trx file!

More information available at [https://github.com/JakeGinnivan/VSTest.TeamCityLogger](https://github.com/JakeGinnivan/VSTest.TeamCityLogger)

## 2. TfsBuildResultPublisher 
The next part of the puzzle is getting the build into TFS, this is where TfsBuildResultPublisher comes in. 
It is based off Based off [http://blogs.msdn.com/b/jpricket/archive/2010/02/23/creating-fake-builds-in-tfs-build-2010.aspx](http://blogs.msdn.com/b/jpricket/archive/2010/02/23/creating-fake-builds-in-tfs-build-2010.aspx) and [http://msmvps.com/blogs/vstsblog/archive/2011/04/26/creating-fake-builds-in-tfs-build-2010-using-the-command-line.aspx](http://msmvps.com/blogs/vstsblog/archive/2011/04/26/creating-fake-builds-in-tfs-build-2010-using-the-command-line.aspx) but it provides a heap of additional options, like:

 - Publish Test Results, if you simply want to publish a .trx into your build, TfsCreateBuild will take care of that for you.
 - Publish Test Run to MTM (Microsoft Test Manager)
    - In addition to this, VSTest.Console.exe generates different testId's to MSTest, this means that MTM will not recognise the associated tests (I am coming to this soon!). TfsCreateBuild has a flag to fix your .trx file and correct the testId's so the automation is matched.

More information available at [https://github.com/JakeGinnivan/TfsBuildResultPublisher](https://github.com/JakeGinnivan/TfsBuildResultPublisher)

## TestCaseAutomationAssigner
I mentioned earlier my tests are xUnit.net and that visual studio only lets you associate MSTest tests with Test Cases. 

This is where [https://github.com/JakeGinnivan/TestCaseAutomationAssigner](https://github.com/JakeGinnivan/TestCaseAutomationAssigner) comes in. It is a simple WPF application which allows you to associate either xUnit.net or NUnit tests with Test Cases.

# Summary
Using TestCaseAutomationAssigner, TfsCreateBuild and VSTest.TeamCityLogger you can use TeamCity as your build server, xUnit or NUnit as your unit test framework and report the results back into Test Manager and TFS giving you all the reportability and value that comes with TFS when you use all the features!