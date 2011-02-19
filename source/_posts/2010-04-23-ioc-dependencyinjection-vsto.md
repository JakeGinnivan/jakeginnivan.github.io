---
layout: post
title: IoC, Dependency Injection and VSTO
metaTitle: Inversion of Control, Dependency Injection and VSTO
description: How can you use DI and IoC containers within VSTO?
revised: 2011-02-19
date: 2010-04-23
categories: [vsto,patternsandpractices,c#,.net]
migrated: true
comments: true
sharing: true
footer: true
permalink: /ioc-dependencyinjection-vsto/
summary: | 
  

---
A really common problem I see with VSTO is getting access to dependencies. Because the framework builds your ribbons and the main point of interaction is in the Load event, developers resort to putting dependencies in their ThisAddin.cs class, then access them through Globals.ThisAddin.MyDependency.

This is bad..

Unfortunately out of the box this is really hard to get around, in this post I will present a few options and a few helper classes that will be included in Outlook.Utility very soon.

I try to use IoC and DI as much as I can inside Outlook add-ins, in some places this is really hard, so I fall back to a ServiceLocator.

<h1>Why IoC <strong>and</strong> ServiceLocator</h1>

If you are not sure about what IoC, DI or a ServiceLocator is I suggest you do some reading, understanding these concepts (even if you do not use them) will make you a better developer. Start at [http://martinfowler.com/articles/injection.html][1].

The main problem is if you create a Ribbon with designer support, VSTO constructs the class for you, then you add your code in the Ribbon_Load event handler that is setup for you. Many parts of VSTO also suffer from this issue. So we can use the service locator in these area’s.

Because I use both a service locator and a IoC container I will use the Common Service Locator to wrap my IoC container so we only have to register our components once with our IoC container, and the ServiceLocator will use our container to resolve the dependencies, and does not tie our code to a specific container.

<h1>What you need</h1>

 - A IoC container (I will be using [Autofac][2])
 - [Common Service Locator][3]
 - Common Service Locator adapter for your IoC container. Autofac has one in the [contrib project][4].

<h1>Registration</h1>
There are two components I see are essential to register in your IoC container, I have found a lot of my Services/Repositories will use them. Application.Session (NameSpace interface) is your current outlook session, you need the session to create or search for Items. And the Dispatcher for the Outlook STA thread.

<h1>Outlook runs in a Single Threaded Apartment</h1>

Outlook runs in what is called a [Single Threaded Apartment (STA)][5], which essentially means Outlook runs on a single thread, all calls to the Outlook Interop library will be marshalled to the STA if you make the call on a background thread, this is expensive, if you are doing heavy interaction with Outlook, don’t do it from a background thread, BUT still make web service and other long calls or heavy work in .NET then do it on a background thread.

<h1>Usage</h1>
I will cover more advanced uses of IoC and how to get it working with Ribbons and other area’s later in this series. For now if you understand IoC you want to use it to register all your services and the .Core project should not have to use the ServiceLocator.

Then in your Ribbon_Load event handler you can simply go:

    var dialogService = ServiceLocator.Current.GetInstance<IDialogService>();

To resolve your dependencies.

Next I will be talking about how to get data out of Outlook safely (no leaking COM objects) by using the Repository pattern.

  [1]: http://martinfowler.com/articles/injection.html
  [2]: http://code.google.com/p/autofac/
  [3]: http://commonservicelocator.codeplex.com/
  [4]: http://code.google.com/p/autofac/downloads/list
  [5]: http://msdn.microsoft.com/en-us/library/ms680112(VS.85).aspx
