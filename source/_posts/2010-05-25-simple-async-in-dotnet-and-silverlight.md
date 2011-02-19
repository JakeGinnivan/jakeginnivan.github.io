---
layout: post
title: Simplifiying Asynchronous work in .NET and Silverlight
metaTitle: Simplifiying Asynchronous work in .NET and Silverlight
description: In both .NET and Silverlight we have to work asynchronously, in Silverlight this is forced. I explored how I could make my life easier!
revised: 2011-02-19
date: 2010-05-25
categories: [.net,wcf,silverlight,c#]
migrated: true
comments: true
sharing: true
footer: true
permalink: /simple-async-in-dotnet-and-silverlight/
summary: | 
  

---
When working with silverlight, one of the first things you notice is all services have only Asynchronous methods available. This normally results in chaining completed callbacks to handle data coming back and then starting more work.

For example take this code:

    void Main()
    {
        var webRequest = HttpWebRequest.Create("http://www.google.com");
        IAsyncResult asyncResult = null;
        asyncResult = webRequest.BeginGetResponse(state =>
                        {
                            var response = webRequest.EndGetResponse(asyncResult);
                            Console.WriteLine(new StreamReader(response.GetResponseStream()).ReadToEnd());
                        }, null);
    }

Even if we do not want to get the result, we should ALWAYS call EndInvoke/EndGet to re-throw any exceptions that occurred on the background thread.

Sometimes it seems to much hassle because we have to keep a reference to our IAsyncResult to pass back into EndGet, so then API’s that look like this start appearing:

    myService.GetDataComplete += CompletedCallback;
    myService.BeginGetData(args);

I dislike this API because it means you cannot reuse that instance of the service in multiple classes, or make multiple calls to the same method at the same time. It makes it hard to match up what call the event was raised for. So my solution doesn’t cover this style of asynchronous API’s, sorry guys.

<h1>My Solution</h1>

I started off trying to solve the problem of performing multiple asynchronous tasks at the same time, and getting a single callback once done all of them. So I created a SimultaneousWorker class which you can add work to.

    var webRequest = WebRequest.Create("http://www.google.com");
    var worker = new SimultaneousWorker();

    worker.AddWork(
        new WorkItem
            {
                Work = callback => webRequest.BeginGetResponse(callback, null),
                CompletedEvents =
                    {
                        r =>
                            {
                                ReadResponseStream(webRequest
                                                        .EndGetResponse(r)
                                                        .GetResponseStream());
                            }
                    }
            });

    worker.PerformWork(AllComplete);

So we now at least can synchronise multiple work items to be performed at the same time, and give us a single callback once done.

But I still feel this is too verbose, has too much overhead and it is hard to figure out what the intention is. So I built a fluent interface wrapping my simultaneous worker, and also added support for Action, and Func<T>, which will also be executed asynchronously.

<h1>UnitOfWork class</h1>
The main goal of the unit of work class was to build WorkItem's which the SimultaniousWorker used, but reduce the amount of code and effort required to create those WorkItem's.

Here is the first example rewritten to take advantage of the UnitOfWork:

    var webRequest = WebRequest.Create("http://www.google.com");
    var unitOfWork = new UnitOfWork();

    unitOfWork
        .AddWork(webRequest.BeginGetResponse,
                    r => webRequest.EndGetResponse(r))
        .WhenComplete(c => ReadResponseStream(c.Result.GetResponseStream()))
        .WhenAllComplete(AllComplete)
        .PerformWork();

This is using the IAsync Pattern, why shouldn't we be able to add work that is synchronous, and have it performed asynchronously. 

    new UnitOfWork()
        .AddWork(() =>
                        {
                            Thread.Sleep(1000);
                            Console.WriteLine("Some synchronous work");
                        })
        .WhenAllComplete(AllComplete)
        .PerformWork();

Now lets mix up synchronous work and asynchronous work, and let the unit of work coordinate it all! We can also return an object from the synchronous work

    var webRequest = WebRequest.Create("http://www.google.com");

    new UnitOfWork()
        .AddWork(() =>
        {
            Thread.Sleep(1000);
            Console.WriteLine("Some synchronous work");
            return "My Data";
        })
        .WhenComplete(c => Console.WriteLine(c.Result)) //Prints 'My Data'

        .AddWork(webRequest.BeginGetResponse, r => webRequest.EndGetResponse(r))
        .WhenComplete(c => ReadResponseStream(c.Result.GetResponseStream()))

        .WhenAllComplete(AllComplete)
        .PerformWork();

Simple right? What about error handling? 

    new UnitOfWork()
        .AddWork(() =>
                        {
                            Thread.Sleep(1000);
                            throw new InvalidOperationException("Oh no");
                        })
        .WhenComplete(c =>
                            {
                                if (c.Error != null)
                                    Console.WriteLine("Oops something went wrong: {0}", c.Error.ToString());
                            })
        .PerformWork();

The syntax is exactly the same for Async Pattern calls or Func/Action delegates.

The next goal of the UnitOfWork was being able to chain work. When a work item completes, I want to use the result to fetch further data for instance.

    unitOfWork
        .AddWork(webRequest.BeginGetResponse,
                    r => webRequest.EndGetResponse(r))
        .WhenComplete(c =>
                            {
                                var url = GetFirstResultUrl(c.Result.GetResponseStream());
                                var firstResultRequest = WebRequest.Create(url);
                                c.DoWork(firstResultRequest.BeginGetResponse, //Begin Get
                                        firstResultRequest.EndGetResponse,    //End Get
                                        //Handle complete
                                        complete => ReadResponseStream(complete.Result.GetResponseStream())); 
                            })
        .WhenAllComplete(AllComplete)
        .PerformWork();

The main difference between the AddWork method and the DoWork method is that you must specify the work, and the completed handlers in one method call because the work will start as soon as the method call is being made.

You can also make multiple calls to DoWork in the completion context of some work. This becomes really powerful in data aggregation systems where you are pulling data from multiple data sources.

The library has both a Silverlight and a .NET build, and all callbacks are made on the same synchronisation context as the UnitOfWork was constructed on. So it is recommended to create the UnitOfWork on the UI thread.

<h1>Unit Tests</h1>

I have created a decent suite of unit tests. Unit testing asynchronous processes is not straightforward, especially in silverlight. So if you need to test anything Asynchronous, then have a look at the tests. 

In silverlight the unit testing framework has some really nice Asynchronous helpers. See:

    [TestMethod, Asynchronous]
    public void TestSynchronousWorkItemReturningData()
    {
        var vm = new UnitOfWork();
        vm
            .AddWork(() => "result")
            .WhenComplete(result =>
                                {
                                    Assert.AreEqual("result", result.Result);
                                    TestComplete();
                                })
            .PerformWork();
    }

The Asynchronous attribute on the test instructs the framework to run the test on a background thread, and only complete the test when TestComplete() is called (Method in base WorkItemTest class, which is also provided by the test framework).

<h1>Source code</h1>

[Download the solution][1]

  [1]: /get/downloads/CommonThreading.zip