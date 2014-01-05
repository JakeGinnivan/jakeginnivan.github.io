---
layout: post
title: WP7 MVC Debug and Tracing
metaTitle: WP7 MVC Debug and Tracing
description: Now that WP MVC is maturing, it is important that when you have performance issues or any issues, you can find the problem easily!
revised: 2011-09-23
date: 2011-09-22
categories: [Open Source, Windows Phone]
migrated: true
comments: true
sharing: true
footer: true
permalink: /wp7-mvc-trace-logging-and-debugging/
summary: | 
  Now that WP MVC is maturing, it is important that when you have performance issues or any issues, you can find the problem easily!

---
## WP7 Essentials Tracing
A new feature coming in the next release is a super easy to use Trace class to allow you to debug your app really easily.
<!-- more -->
In your App.xaml.cs constructor simply put:

    #if DEBUG
        Trace.Appenders.Add(s=>Debug.WriteLine(s));
        Trace.TraceLevel = TraceLevel.Debug;
    #endif

Then the following code will output `9/23/2011 8:10 PM - Debug   [MyClass] - Some  Trace Message` in your Debug output window. You can add your own trace appenders too if you want to implement logging in your app.

    // this.GetType() == typeof(MyClass)
    Trace.WriteInfo(this, TraceLevel.Debug, ()=>"Some Trace Message");

Notice that we pass a lambda into the WriteInfo class, this is so the Trace class can be super lazy and if the message is not going to be written to the Trace appenders, it will not even construct that string (useful if you are doing more complex logging like checking memory usage etc).

## WP7 MVC Tracing
I have been using this tracing ability in vNext of Windows Phone MVC to measure performance and start optimising. Here is an example of the full debug log when running


    9/23/2011 8:46 PM - Debug   [AutofacNavigationApplication] - Scanning for and Registering Autofac Modules
    9/23/2011 8:46 PM - Debug   [AutofacNavigationApplication] - {
    9/23/2011 8:46 PM - Debug   [AutofacNavigationApplication] -     Registering module ApplicationModule... 1ms
    9/23/2011 8:46 PM - Debug   [AutofacNavigationApplication] - } 46ms
    9/23/2011 8:46 PM - Debug   [AutofacNavigationApplication] - Building Container... 72ms
    9/23/2011 8:46 PM - Debug   [NavigationApplicationActivator] - Handling WP7 Navigation Event to /Shell.xaml
    9/23/2011 8:46 PM - Debug   [NavigationApplicationActivator] - {
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Memory usage before navigation: 8.54296875
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Navigating Forward to Home.MainPage
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     {
    9/23/2011 8:46 PM - Debug   [DefaultControllerLocator      ] -         Locating controller Home
    9/23/2011 8:46 PM - Debug   [DefaultControllerLocator      ] -         {
    9/23/2011 8:46 PM - Debug   [DefaultControllerLocator      ] -             Scanning assemblies for controllers and building lookup... 2ms
    9/23/2011 8:46 PM - Debug   [DefaultControllerLocator      ] -         } 10ms
    9/23/2011 8:46 PM - Debug   [NavigationApplicationActivator] -     } 45ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Resolving best matching action MainPage on Home... 28ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Invoking controller action for Home.MainPage... 52ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult for Home.MainPage
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     {
    9/23/2011 8:46 PM - Debug   [DefaultViewLocator            ] -         Scanning assemblies for views and building lookup... 3ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     } 214ms
    9/23/2011 8:46 PM - Debug   [Journal                       ] -     Home.MainPage added to Journal
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult completion steps... 1ms
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Memory usage after navigation: 9.15234375
    9/23/2011 8:46 PM - Info    [Navigator                     ] - } 526ms
    9/23/2011 8:46 PM - Debug   [ControllerActions`1           ] - Parsing Navigation Expression
    9/23/2011 8:46 PM - Debug   [ControllerActions`1           ] - {
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Memory usage before navigation: 9.23046875
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Navigating Forward to AnotherController.Example
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     {
    9/23/2011 8:46 PM - Debug   [DefaultControllerLocator      ] -         Locating controller AnotherController... 0ms
    9/23/2011 8:46 PM - Debug   [ControllerActions`1           ] -     } 14ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Resolving best matching action Example on AnotherController... 0ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Invoking controller action for AnotherController.Example... 0ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult for AnotherController.Example... 44ms
    9/23/2011 8:46 PM - Debug   [Journal                       ] -     AnotherController.Example added to Journal
    9/23/2011 8:46 PM - Debug   [PageResult                    ] -     Cleaning up ViewModel MainViewModel... 1ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult completion steps... 0ms
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Memory usage after navigation: 10.90234375
    9/23/2011 8:46 PM - Info    [Navigator                     ] - } 101ms
    9/23/2011 8:46 PM - Debug   [Journal                       ] - AnotherController.Example popped from Journal
    9/23/2011 8:46 PM - Debug   [Journal                       ] - Home.MainPage popped from Journal
    9/23/2011 8:46 PM - Info    [Navigator                     ] - Memory usage before navigation: 9.1640625
    9/23/2011 8:46 PM - Info    [Navigator                     ] - Navigating Backward to Home.MainPage
    9/23/2011 8:46 PM - Info    [Navigator                     ] - {
    9/23/2011 8:46 PM - Debug   [DefaultControllerLocator      ] -     Locating controller Home... 0ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Resolving best matching action MainPage on Home... 0ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Invoking controller action for Home.MainPage... 0ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult for Home.MainPage... 22ms
    9/23/2011 8:46 PM - Debug   [Journal                       ] -     Home.MainPage added to Journal
    9/23/2011 8:46 PM - Debug   [PageResult                    ] -     Cleaning up ViewModel ExampleViewModel... 0ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult completion steps... 1ms
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Memory usage after navigation: 10.55859375
    9/23/2011 8:46 PM - Info    [Navigator                     ] - } 82ms
    9/23/2011 8:46 PM - Info    [Navigator                     ] - Memory usage before navigation: 9.8359375
    9/23/2011 8:46 PM - Info    [Navigator                     ] - Navigating Forward to Home.DebugPage
    9/23/2011 8:46 PM - Info    [Navigator                     ] - {
    9/23/2011 8:46 PM - Debug   [DefaultControllerLocator      ] -     Locating controller Home... 0ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Resolving best matching action DebugPage on Home... 0ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Invoking controller action for Home.DebugPage... 4ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult for Home.DebugPage... 104ms
    9/23/2011 8:46 PM - Debug   [Journal                       ] -     Home.DebugPage added to Journal
    9/23/2011 8:46 PM - Debug   [PageResult                    ] -     Cleaning up ViewModel MainViewModel... 0ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult completion steps... 0ms
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Memory usage after navigation: 13.82421875
    9/23/2011 8:46 PM - Info    [Navigator                     ] - } 267ms
    9/23/2011 8:46 PM - Info    [Navigator                     ] - Loading partial view Debug.DemoPartial
    9/23/2011 8:46 PM - Info    [Navigator                     ] - {
    9/23/2011 8:46 PM - Debug   [DefaultControllerLocator      ] -     Locating controller Debug... 0ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Resolving best matching action DemoPartial on Debug... 0ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Invoking controller action for Debug.DemoPartial... 2002ms
    Partial Activated
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PartialViewResult completion steps... 2ms
    9/23/2011 8:46 PM - Info    [Navigator                     ] - } 2040ms
    9/23/2011 8:46 PM - Debug   [ControllerActions`1           ] - Parsing Navigation Expression
    9/23/2011 8:46 PM - Debug   [ControllerActions`1           ] - {
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Memory usage before navigation: 15.98828125
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Navigating Forward to DebugController.PageWithResult
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     {
    9/23/2011 8:46 PM - Debug   [DefaultControllerLocator      ] -         Locating controller DebugController... 0ms
    9/23/2011 8:46 PM - Debug   [ControllerActions`1           ] -     } 12ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Resolving best matching action PageWithResult on DebugController... 0ms
    9/23/2011 8:46 PM - Debug   [DefaultActionInvoker          ] -     Invoking controller action for DebugController.PageWithResult... 0ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult for DebugController.PageWithResult... 35ms
    9/23/2011 8:46 PM - Debug   [Navigator                     ] -     Executing WindowsPhoneMVC.ActionResults.PageResult completion steps... 0ms
    9/23/2011 8:46 PM - Info    [Navigator                     ] -     Memory usage after navigation: 17
    9/23/2011 8:46 PM - Info    [Navigator                     ] - } 103ms


Now you may be saying one of two things, OMG that is way better than the OOTB profiler or you will be saying now my output window will be spammed!

Say we want to silence the Navigator because it logs a lot, we simply do:

    Trace.Filters.Add(typeof(Navigator));

And all Trace events from the Navigator class will be ignored. So you can find out useful information when you need it, and ignore it other times!

Keen to know what everyone thinks.

## Check it out at [http://windowsphonemvc.codeplex.com/](http://windowsphonemvc.codeplex.com/)