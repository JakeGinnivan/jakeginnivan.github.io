---
layout: post
title: On Async and Sync Contexts
date: 2014-01-10 14:12:49 +0000
comments: true
categories: [Async]
---
This week has been a really heavy learning experience for me in terms of async/await, .ConfigureAwait() and how it interacts with Synchronisation Contexts. Quite a few of my assumptions were wrong, or the behaviour is different between the .NET 4.0 implementation (CTP3 and RTM) and what happens in .NET 4.5.

In this post I will cover:

 - What .ConfigureAwait(false) does in .net 4.0 and in .net 4.5, and why they act differently
 - Some guidance around using await in a rich client app (i.e await for *offloading*, not scalability) .net 4.0 in particular, .net 4.5 behaves better and the guidance is not as relevant
 - How you can use custom awaiters to switch contexts

<!-- more -->
## Quick SynchronizationContext Intro
If you do not know what synchronization context's are, then I will quickly cover them here. If you know what they do, skip to the next section.

A SynchronizationContext allows you to queue a unit of work on another *context*, a context can be a specific thread or it may be shared between multiple threads. For example you can get the SynchronizationContext of the current thread by going `var uiSyncContext = SynchronizationContext.Current`, you can use it to run a delegate on the UI thread (or whatever context the SynchronizationContext represents) with `uiSyncContext.Send(state => { MyProperty = state; }, state: "bar")`. This will pass the string "bar" through as state (without implicitly capturing the outside closure) then assign to MyProperty on the UI thread.

SynchronizationContext's also get notified when asynchronous work is started, which is how frameworks like ([nUnit](http://www.nunit.org/), [xUnit 2.0](http://xunit.codeplex.com/), [BDDfy](https://github.com/TestStack/TestStack.BDDfy) and others supports tests which have a method signature of `async void`

## How await uses SynchronisationContext's
Async/await is just compiler magic to make it easier to compose asynchronous stuff, it does not make your code run asynchronously all it does is when it sees an await, it splits your method up into a state machine then when the thing you are awaiting finishes executing, the state machine is resumed and your code continues running. Resuming execution is called the *continuation*.  
A great feature of the `await` keyword is that it captures the current SynchronizationContext before it runs the asynchronous operation, then it will post the continuation to that SynchronizationContext, meaning if you are on the UI Thread when you `await Foo()` once `Foo()` finishes running your code will continue execution on the UI Thread. 

This scenario here is called *offloading*, because the UI thread is an important thread you want to run as little code as possible on the UI thread. 

### Controlling await with .ConfigureAwait()
To change the default behaviour TPL gives us the `.ConfigureAwait(bool continueOnCapturedContext)`, this allows you to write code like this:

    public async Task<ObservableCollection<DataViewModel>> DoSomeStuff() {
        var results = await _service.GetSomeData().ConfigureAwait(false);
        var mappedData = MapDataToViewModels(results); // Just pretend this is CPU bound work
        return mappedData;
    }

Because of the `.ConfigureAwait(false)` our continuation and the `MapDataToViewModels()` method call will be run on a background thread (unless the awaited task is already complete, then the continuation will be executed in-line). Lets take this one step further

    public async Task<ObservableCollection<DataViewModel>> DoSomeStuff() {
        var results = await _service.GetSomeData().ConfigureAwait(false);
        var mappedData = MapDataToViewModels(results);
        var moreData = await _service.GetMoreData();
        UpdateViewModelsWithAdditionalData(moreData);
        return mappedData;
    }

In which context does `UpdateViewModelsWithAdditionalData(moreData);` run?

The answer is *it depends*, on .net 4.0 `UpdateViewModelsWithAdditionalData(moreData);` runs on the UI thread, on .net 4.5 it will run on the *threadpool*. Why is this? To answer that we need to dive into something called the execution context.

### Execution Context?
When our .NET code is executing, these is a bunch of additional metadata which floats around with our thread, these are things like security information and the synchronisation context. For more in depth information on the .net execution context have a read of [http://blogs.msdn.com/b/pfxteam/archive/2012/06/15/executioncontext-vs-synchronizationcontext.aspx](http://blogs.msdn.com/b/pfxteam/archive/2012/06/15/executioncontext-vs-synchronizationcontext.aspx).

The important thing to know about the execution context is that the .NET framework flows it around so things work as we would expect, for instance if we start a new thread, out execution context will go with us. Task.Run, ThreadPool.QueueUserWorkItem() etc all will flow the execution context (but will supress the flow of the SynchronizationContext). 

## Differences between .NET 4.0 and 4.5
So we have explained some of the moving parts when we are dealing with async/await. (If you would like me to do a more in depth post on async/await in general and how it works starting from the basics, leave a comment below).

Above I showed this code

    public async Task<ObservableCollection<DataViewModel>> DoSomeStuff() {
        var results = await _service.GetSomeData().ConfigureAwait(false);
        var mappedData = MapDataToViewModels(results);
        var moreData = await _service.GetMoreData();
        UpdateViewModelsWithAdditionalData(moreData);
        return mappedData;
    }

And said that in .NET 4.0 `UpdateViewModelsWithAdditionalData(moreData);` would run on the UI thread and in .NET 4.5 it would run on the ThreadPool. This is because of how the different implementations capture the `ExecutionContext`.

### Breakdown of .NET 4.0
If we break this method down into what will be executed. This is a mixture of the generated state machine, library code and code to just demonstrate what is happening (because there are heaps of moving parts). 

    int state;
    Results results;

    void IAsyncStateMachine.MoveNext()
    {
        switch 1:
            var mappedData = MapDataToViewModels(results);
            // etc...
            break;
        // etc...
        default:
            var ex = ExecutionContext.Capture();
            var task = _service.GetSomeData();
            
            TaskAwaiter.OnCompletedInternal(task, ()=> {
                ex.Run(()=>
                {
                    results = task.Result;
                    state = 1;
                    MoveNext();
                });
            }, continueOnCapturedContext: false);
    }

Now there are a few things to look at in this small bit of code, first is `var ex = ExecutionContext.Capture();`, this captures the current execution context and the ExecutionContext contains the SynchronizationContext so when we execute `MoveNext()` our SynchronizationContext has been restored BUT because the TaskAwaiter.OnCompletedInternal has been told to not continue on the captured context it will be run on the default task scheduler.

The end result of all this is when `var mappedData = MapDataToViewModels(results);` runs we will:

 - Be running on the ThreadPool
 - SynchronizationContext.Current will still be the `DispatcherSynchronizationContext`
 
So on the next line when we await and *do not use* .ConfigureAwait(false) the continuation will run on the captured SynchronizationContext

### .NET 4.5 Behaviour
Lets have a look at what the code looks like in .NET 4.5 land.

    int state;
    Results results;

    void IAsyncStateMachine.MoveNext()
    {
        switch 1:
            var mappedData = MapDataToViewModels(results);
            // etc...
            break;
        // etc...
        default:
            var ex = ExecutionContext.CaptureInternal(ExecutionContext.CaptureOptions.IgnoreSyncCtx); 
            var task = _service.GetSomeData();
            
            TaskAwaiter.OnCompletedInternal(task, ()=> {
                ex.Run(()=>
                {
                    results = task.Result;
                    state = 1;
                    MoveNext();
                });
            }, continueOnCapturedContext: false);
    }

Notice the line where the execution context is captured, in .NET 4.5 the AsyncMethodBuilderCore calls an *internal* method on the ExecutionContext which allows the caller to specify that they do not want the SynchronizationContext to be captured as part of the ExecutionContext.

The reason .NET 4.0 and .NET 4.5 are different is that this internal cannot be called from the Async Targeting Pack because it's a library, in .NET 4.5 the AsyncMethodBuilderCore is part of the framework so it can call this internal methods.

### How does TaskEx.Run not capture SynchronizationContext?
Because TaskEx.Run delegates to `Task.Factory.StartNew()` which is part of the framework, it passes the option to not capture the SynchronizationContext

## Some guidance for .NET 4.0
The general guidance is any library code should use `.ConfigureAwait(false)` so our continuations do not constantly post back to the UI thread, because .NET 4.0 always flows the SynchronizationContext it means that *all* of our await calls should have `.ConfigureAwait(false)` which is pretty ugly. Also because .ConfigureAwait(false) still checks if the task is complete, using `.ConfigureAwait(false)` does not guarantee that we will no longer be executing on the UI thread.

Rather than using .ConfigureAwait() we can explicitly get our code running on the ThreadPool as soon as possible. There are two options for this (that I can see currently):

In ViewModels when you invoke any sort of async service, wrap it in a `await TaskEx.Run(()=>_myService.DoStuffAsync());`, this is quite good because if your service calls other services you do not get the problem where all your services have `TaskEx.Run` scattered everywhere and you are scheduling a heap more things into the ThreadPool. 

Create a app services layer which your viewmodels interact with, in this layer you delegate to your domain or existing app services, but wrap it in a `TaskEx.Run(()=>..);`

    | UI/ViewModels |   ==>   | Service Layer (TaskEx.Run) |   ==>   | Rest of app (never has sync context) |

Both of these options basically try and make sure that only code executing in the ViewModels have a SynchronizationContext, and the rest of your application had no threading concerns.

### Additional Option for .NET 4.5
If you are using .NET 4.5, then another option is to create a custom awaiter (*note* this feature was removed from the async CTP, possibly because it doesn't work properly in .NET 4.0 and also because it can confuse people)

    await TaskHelper.SwitchToThreadPool();

Just awaiting this will cause our async method to switch onto the ThreadPool, personally I think this is quite nice because you do not have to pay the Lambda Tax of `Task.Run(..)`

The code is pretty simple

    public static class TaskHelper
    {
        public static SwitchContextToThreadPoolAwaiter SwitchToThreadPool()
        {
            return new SwitchContextToThreadPoolAwaiter();
        }
    }

    public struct SwitchContextToThreadPoolAwaiter : INotifyCompletion
    {
        public SwitchContextToThreadPoolAwaiter GetAwaiter() { return this; }

        public bool IsCompleted { get { return false; } }

        public void OnCompleted(Action continuation)
        {
            if (!Thread.CurrentThread.IsThreadPoolThread)
                ThreadPool.QueueUserWorkItem(state => ((Action)state)(), continuation);
            else
                continuation();
        }

        public void GetResult() { }
    }

## Summary
I dived into this because of randomly failing tests when I upgraded to nCrunch 2.2 beta, this was due to some false assumptions about how async/await worked in .net 4.0 and also because of some *helper* classes which scheduled work on the threadpool or on the main UI thread. 
Because of the way the SyncContext flowed, the nUnit synchronisation context was present where we thought it should not be. Because of this our code and helpers were not working correctly as the WPF UI DispatcherSynchronizationContext can be captured as a TaskScheduler and has a few other differences.

In short

 - .NET 4.0 flows the SynchronizationContext always, even when you use `.ConfigureAwait(false)`
 - .NET 4.5 does not flow the SynchronizationContext when you use `.ConfigureAwait(false)`
 - Don't try and use the nUnit SynchronizationContext as a task scheduler
 - If you are using .NET 4.0, use TaskEx.Run to get off the UI Thread as fast as possible, then use an `await` in your viewmodel to capture the UI thread and bring you back to it
 - Make sure your application code does not need to run on the UI thread, if you have services (like say an `IDialogService`) then it should post to the UI dispatcher in that service so it is effectively mocked in your unit tests. This was the main source of our issues with tests failing..
 - Don't mix threading concerns and application code
 
 
I Hope that helps a few people understand SynchronizationContexts, ExecutionContexts and how async/await works with either of them.

## Resources
[Stephen Toub on ExecutionContext vs SynchronizationContext (talks about .net 4.5 behaviour)](http://blogs.msdn.com/b/pfxteam/archive/2012/06/15/executioncontext-vs-synchronizationcontext.aspx)