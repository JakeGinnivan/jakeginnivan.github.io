---
layout: post
title: "nUnit 2.6.3, async and .net 4.0 broken"
date: 2014-01-14 18:55:09 +0000
comments: true
categories: [async]
---
I am currently working on a **.net 4.0** project which uses async/await quite heavily. We also are using the old AsyncTargeting pack rather than the RTM because we do not have stats on how many clients currently have the [.net 4.0 KB 2468871 patch](http://www.microsoft.com/en-us/download/details.aspx?id=3556) which enables PCL support. These issues affect the RTM version as well.

Recently I have upgraded to nCrunch 2.2 beta and ReSharper 8.1 both which ship with the upgraded nUnit 2.6.3 runner. After I upgraded these tools I noticed tests which should have been failing were passing and passing tests were being reported as failing but showing stack traces from a different test..

Also I was getting different results in nCrunch, R# and nUnits console runner. Something was broken.

**NOTE**: Async and TPL support is *not supported* in nUnit 2.x, but will be officially supported in v3.x and that it was a coincidence that it worked in 2.6.2. My discussions about the issues are at [here on the nunit discussion board](https://groups.google.com/forum/#!topic/nunit-discuss/McE95Cy2DlY).  
As far as I can tell, there is no reason that 4.0 cannot be supported because to offer framework support does not need any new features OR the classes in the Async Targeting Pack or .NET 4.5. At a minimum tests returning `Task` should be supported as TPL was introduced into the CLR for net40.    
Recently I added async void and Task support to [BDDfy](https://github.com/TestStack/TestStack.BDDfy/pull/32) which targets .NET 4.0, also xUnit 1.9.x supports Tasks in the current released version and has backported `async void` support to the 1.9.x codebase from the 2.0 and will be released if there is a need to release another patch release before 2.0 is released.

<!-- more -->

![2014-01-14-nunit-263,asyncandnet4](/assets/posts/2014-01-14-nunit-263,asyncandnet4.png)

When I first saw this it really really confused me, so I have been diving into async, nUnit and trying all different things which is where my blog post on [async and synchronisation contexts](http://jake.ginnivan.net/blog/2014/01/10/on-async-and-sync-contexts/) came from.

## So what's broken?
But in 2.6.3 tests returning `Task` and `async void` tests will not wait for completion. In addition to that, nUnit 2.6.3 will flat out refuse to run tests which return `Task` and the console runner will return an error code. Other runners just silently skip the tests... 

This means that if you use async/await *or* TPL and have upgraded to R# 8.1, nCrunch 2.2 or 2.3 beta, the nUnit.Runners NuGet project or any other tools which have upgraded to use the nUnit 2.6.3 runner internally your async tests will be completing **unobserved** and may be failing without your knowledge.

### Repro Solution
I put together a sample solution showing the issues (and the screenshot above is from) the GitHub repo is available at [https://github.com/JakeGinnivan/nUnit_net4.0AsyncIssues](https://github.com/JakeGinnivan/nUnit_net4.0AsyncIssues)

Here are the test results from 2.6.2:

	Tests run: 3, Errors: 1, Failures: 0, Inconclusive: 0, Time: 1.4863562 seconds
	  Not run: 0, Invalid: 0, Ignored: 0, Skipped: 0
	
	Errors and Failures:
	1) Test Error : ClassLibrary1.Class1Tests.Test1
	   System.InvalidOperationException : Operation is not valid due to the current state of the object.
	
	Server stack trace:
	   at ClassLibrary1.Class1Tests.<Test1>d__0.MoveNext() in c:\Users\Jake\_Code\WpfApplication4\ClassLibrary1\Class1Tests.cs:line 19
	
	Exception rethrown at [0]:
	   at System.Runtime.CompilerServices.AsyncMethodBuilderCore.<ThrowAsync>b__0(Object state)
	   at NUnit.Core.AsyncSynchronizationContext.AsyncOperationQueue.InvokePendingOperations()
	   at NUnit.Core.AsyncSynchronizationContext.AsyncOperationQueue.InvokeAll()
	   at NUnit.Core.NUnitAsyncTestMethod.RunVoidAsyncMethod(TestResult testResult)

And 2.6.3:

	Tests run: 2, Errors: 0, Failures: 0, Inconclusive: 0, Time: 0.676135209417321 seconds
	  Not run: 1, Invalid: 1, Ignored: 0, Skipped: 0
	
	Errors and Failures:
	
	Tests Not Run:
	1) NotRunnable : ClassLibrary1.Class1Tests.Test3
	   Test method has non-void return type, but no result is expected

## What to do about it
Luckily we can fix this and go back to a working version of nUnit (which as far as I can tell works perfectly for both `async void` and `Task` tests).

First you will need to [download nUnit 2.6.2](http://launchpad.net/nunitv2/trunk/2.6.2/+download/NUnit-2.6.2.zip) and extract it to a known location

### Fixing ReSharper
ReSharper is pretty easy as the options dialog allows you to override the runner it uses.

Go into the Resharper menu, then Options. Scroll down to *Unit Testing* under tools and select NUnit.

Then you can point ReSharper at the nUnit lib directory like so:
![2014-01-14-nunit-263,asyncandnet41](/assets/posts/2014-01-14-nunit-263,asyncandnet41.png)

Now ReSharper will be using a non-broken version of nUnit.

### Fixing nCrunch
nCrunch is a little harder, but this fix works fine.

First open up `C:\Program Files (x86)\Microsoft Visual Studio 12.0\Common7\IDE\Extensions\Remco Software\NCrunch for Visual Studio 2013`, then overwrite `nunit.core.dll` and `nunit.core.interfaces.dll` with the ones from the 2.6.2 zip you downloaded earlier.

Now nCrunch will be using the non-broken version. Yay

## Summary
If you are using .NET 4.0, async/await *or* TPL and nUnit, do not upgrade your runners to 2.6.3 and if any tools/build servers you are using upgrade, make sure you set them back to using 2.6.2 manually