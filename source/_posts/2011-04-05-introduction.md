---
layout: post
title: Introduction to VSTO Contrib
metaTitle: Introduction to VSTO Contrib
description: The release of VSTO Contrib is imminent, this will be a series covering various parts of VSTO Contrib.
revised: 2011-04-20
date: 2011-04-05
categories: [open-source,vsto,vsto-contrib,wpf]
migrated: true
comments: true
sharing: true
footer: true
permalink: /vsto-contrib/introduction/
summary: | 
  

---
# Goal of VSTO Contrib
When I first started on this library, it was a very simple set of classes designed to help me do trivial tasks in VSTO, such as simplifying listening to certain events, wrappers, and it was really a dumping ground for code that solved problems I had while developing VSTO solutions.

Just before TechEd last year (August), I started trying to tackle some more fundamental architectural issues, such as no ability to use DI, or IoC, which often causes VSTO solutions to have Ribbons with hundreds of lines of code behind, this issue is made worse with Ribbon XML which only lets you have a single callback file for ALL ribbons.
<!-- more -->
#Version Support
I am targeting both Office 2007 & 2010 using either .NET 3.5 or .NET 4.0, and currently support Outlook, Word, Excel and PowerPoint. This means there are 16 different packages you can download.

#Features

##Ribbon Factory
The Ribbon Factory, apart from needed a better name, is the most compelling feature of VSTO Contrib.  

It is a ribbon factory, similar to the one VSTO uses to bring the Ribbon Designer to life ([read more][1]), except mine targets Ribbon XML, and brings all the nice features that the ribbon designer has, like the current context (Contact, Word Document, Powerpoint presentation etc) plus a whole lot more like:

 - Creates a new instance of a 'ViewModel' class you define for each context (document etc), and will notify you when the current view changes (multiple windows displaying same document). This view model is managed, and will be cleaned up when the context is closed.

 - ViewModels are for a single, or multiple Ribbon types, Word/Excel/PowerPoint are simply the Document, Workbook and Presentation ribbons, but outlook you can create a view model for an AppointmentItems, another viewmodel for ContactItem's, another view model for MailItems

 - IoC container support, currently takes a Func<Type, IRibbonViewModel> during initialisation to support custom resolution. I will be creating a IRibbonViewModelFactory at a later stage.  

 - Synchronised TaskPanes, one of the most common questions on StackOverflow is how do I wire up a button on the ribbon to my custom task pane. It is actually quite hard out of the box, with VSTO Contrib, just inherit from IRegisterCustomTaskPane on your ViewModel and the Ribbon Factory will manage the registration of custom task panes on new windows, and a few other things.  

 - WPF like binding support, RibbonXML requires you to know the method syntax of all callbacks. You can now go <toggleButton onAction="PanelShown" getPressed="PanelShown" /> where PanelShown is a **property**, it will even listen to PropertyChanged events and invalidate the ribbon control!

I won't go into detail about how the Ribbon Factory works, and what the code looks like here. But if you would like to read more, I will write all about it at [the ribbon factory][2]

## Code Driven Click-Once updates
Due to a different security model, and VSTO having a custom ClickOnce installer if you try and get the Deployment information, then call ApplicationDeployment.Update you can find yourself with a broken add-in.

VSTO Contrib has a helper class to make updating your add-in super easy. Under the covers it is finding the location of the VSTOInstaller.exe, first through registry, then falling back to file system. It then sets up the application trusts needed to make the update process work nicely.

    new VstoClickOnceUpdater()
                .CheckForUpdateAsync(
                    r =>
                    {
                        if (r.Updated)
                        {
                            MessageBox.Show("My awesome add-in was updated");
                        }
                    });

##WPF Integration
VSTO Contrib has a few helper classes to make WPF development inside VSTO much easier.

Firstly it has a WpfPanelHost, which is registered correctly for COM interop, and works around some issues where WPF controls would not draw correctly until the window is moved. 

It also provides a OfficeViewModelBase, and DelegateCommand class to enable easy data binding to your Ribbon View Model.

##COM interop helpers
One of the things I found really hard when starting VSTO development was learning about the COM interop side of things, and in particularly the 'right' way of doing it, which is why I wrote the blog post on [vsto-com-interop][3], which explains how the whole COM Interop thing works, any why you should be calling Marshal.ReleaseComObject.

The issue is that code quickly becomes ugly and unmanageable when everything is wrapped in try{} finally{Marshal.ReleaseComObject();}. It also is not tollerant to later versions of Office, which may swap out some of these unmanaged com objects, for managed .net objects. If they do that you will find the call to Marshal.ReleaseComObject throws an exception.

These helpers come in two flavours, simple, and dynamic proxies. 

###.WithComCleanup()
This extension method will return either a `Wrapped<ComType> : IDisposable` (simple) or a `IComType : ComType, IDisposable` (dynamic proxy).

Usage is:
`using (var sheets = workbook.WorkSheets.WithComCleanup())`
`sheets.Resource.Add() //for simple`
or `sheets.Add() //for dynamic proxy version`

The dynamic proxy is slightly nicer, but you have to take on a dependency to Castle.Core if you want that. If you want to see some more code examples of how this can cleanup your VSTO/Office Automation code, [have a look here][4]

The dynamic proxy version simply removes the need to go .Resource to access the wrapped COM object.

###.ComLinq<T>()
Allows you to cleanly write linq against office collections. *Beware* as this will return a IEnumerable<T> with a custom Enumerator which releases the previous item when MoveNext is called. This means it is perfect for code like this:

    using (var someSheet = workbook.Sheets.ComLinq<WorkSheet>().Where(s=>s.Name == "Sheet1"))
    { someSheet.Name = "NewSheetName"; }

But you will get a RCW has been separated from underlying com object exception if you do this:

    var sheetOne = workbook.Sheets.ComLinq<WorkSheet>().SingleOrDefault(s=>s.Name == "Sheet1");
    sheetOne.Name = "NewSheetName"; //Will throw exception, because Enumerator has already released the WorkSheet as SingleOrDefault forces the IEnumerable to be iterated.

Future versions may have a custom collection which only expose specific operations to make this extension method more predictable. For the moment, use with caution.

###Outlook User Properties helpers
You will find some handy extensions called GetPropertyValue<T>, and SetPropertyValue<T>, which are handy wrappers around the UserProperties collection on most outlook items. Has options to automatically create the properties, and specify if they are folder level so they are accessible through search etc. Very handy!


##FolderHomePage [Outlook Only]
Greatly simplifies the process of creating a custom view for a folder. Ever wanted to select a folder in Outlook, and have a fully blown WPF custom view be displayed, well, this class is for you.

Note: Does need outlook to be running as Administrator, as it registers types for com interop and registers the user controls as safe for scripting, which requires the process to be elevated. I may end up making this an extension which will shell out to a elevated process.

##OutlookFolderMonitor
Quite a simple class which will monitor an Outlook MAPIFolder for changes, this includes Add/Modified and deleting. Each event has the item being affected, delete is not trivial, which makes this class particularly handy.

##GenericSynchronisationService
As the name implies, this is a synchronisation helper, which takes care of synchronisation logic, all you have to do is create a source and remote provider. It greatly simplifies your job if you need to synchronise contacts or appointments etc with outlook.


# Where can I get it

I will be releasing it before MIX 2011 at [http://vstocontrib.codeplex.com/releases][5]


#Feedback
I would love feedback, I still have a lot of unit testing to do and test every combination of .net and office. Please if you have an issue, let me know!


 


  [1]: ../vsto-ribbon-designer-in-depth
  [2]: ribbon-factory
  [3]: vsto-com-interop
  [4]: com-cleanup-extension-methods
  [5]: http://vstocontrib.codeplex.com/releases