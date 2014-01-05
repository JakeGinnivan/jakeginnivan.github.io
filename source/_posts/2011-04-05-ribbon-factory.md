---
layout: post
title: VSTO Contrib RibbonFactory
metaTitle: VSTO Contrib RibbonFactory
description: The Ribbon Factory in VSTO Contrib lets you create a view model for each Ribbon, with wpf like binding, context and DI/IoC support!
revised: 2011-04-06
date: 2011-04-05
categories: [vsto,vsto-contrib,IoC,open-source]
migrated: true
comments: true
sharing: true
footer: true
permalink: /vsto-contrib/ribbon-factory/
summary: | 
  

---
# Why?
When I started creating a RibbonFactory, I hadn't actually looked into the way VSTO does it, I just wanted to use Ribbon XML, because it has more flexibility, but have a single class to represent a single ribbon and also support an IoC container, and DI for my ribbons.

Once I got that working, I started tweaking, improving, adding in context awareness and a view other things. Recently my reasons for trying to finish this have been:

 - Lack of IoC/DI support in VSTO
 - Cost of brute force reflection when using ribbon designer
 - Wanting to use Ribbon XML over Ribbon Designer, but not really viable due to missing context for ribbon xml.
 - Ribbon XML having a single callback file
 - Making custom task panes, which are associated with a ribbon (i.e. button to show/hide custom task pane) really simple.

The VSTO tooling is pretty good, especially around the designer, but I want the best of both worlds, and to be able to fall into `the pit of success` as [Paul Stovell][1] often says.
<!-- more -->
#View Model's in Office?
I have broken all the differences between each of the Office applications up into terminology which is common to all of them, and abstracted the behaviours of each. The way I have broken it down is into

`View` - This is the Outlook Explorer, Word Window, Appointment Inspector. Whatever the actual window is that is displaying something in office, this is the view.
`Context` - This is what the window is displaying, in word the Window is displaying a document, in PowerPoint that is a Presentation, in Outlook it is the Contact, or Appointment.
`ViewModel` - Your entry point into VSTO now is view models. You define a few model for a particular ribbon type, then whenever a new context is created which has that ribbon type, you get a new view model instance, with the context set. You will also be notified when the current active view changes (switching between two windows showing the same document).

#Show us the code
You start off with the IRibbonViewModel interface which looks like:

    public interface IRibbonViewModel
    {
        IRibbonUI RibbonUi { get; set; }
        void Initialised(object context);
        void CurrentViewChanged(object currentView);
        void Cleanup();
    }

`RibbonUi` will be set for you, this allows you to invalidate ribbon controls and activate different tabs on the ribbon. 
`Initialised(object context)` will be called when the context is available, allowing you to hook into events, populate viewmodel data or anything you want. `CurrentViewChanged` and `Cleanup` are called when the current active window changes, or the context is closing respectively.

To get started all you have to do is create a class and make it inherit from IRibbonViewModel, then decorate it with the `RibbonViewModelAttribute`, which will look like this:

    [RibbonViewModel(OutlookRibbonType.OutlookContact)]
    public class ContactFeed : OfficeViewModelBase, IRibbonViewModel, IRegisterCustomTaskPane
    {

    }

OfficeViewModelBase is a helper base class which inherits from INotifyPropertyChanged, and has helpers for raising property changed events with compile time safety.

The IRegisterCustomerTaskPane is the next interesting interface you have access to.

    public interface IRegisterCustomTaskPane
    {
        void RegisterTaskPanes(Register register);
    }

    public delegate ICustomTaskPaneWrapper Register(Func<UserControl> controlFactory, string title);

An example implementation is:

    public void RegisterTaskPanes(Register register)
    {
        _twitterTaskPane = register(() => new WpfPanelHost{Child = new TwitterFeed{DataContext = this}}, "Twitter");
        _twitterTaskPane.Visible = true;
        PanelShown = true;
        _twitterTaskPane.VisibleChanged += TwitterTaskPaneVisibleChanged;
        TwitterTaskPaneVisibleChanged(this, EventArgs.Empty);
    }

Now the RibbonFactory will take care of registering the custom task pane when new views are opened, and keep properties synchronised across the same task panes on different views (visibility etc).

#How does it work?
This was the hard part, it all sounds good in theory, but getting all the pieces together was a lot of hard work.

When `GetCustomIU(string RibbonId)` is called, the RibbonFactory finds a registered ViewModel for that ribbon type, it then fines the appropriate RibbonXml resource through the current `IViewLocator`, so you can specify your own conventions, by default it will look for a Resource with the same name, or try to match without view/viewmodel appended to the end of the names.

Once the RibbonXML has been found, the RibbonFactory rewrites all callbacks and caches them inside the `ViewModelResolver` class, it also generates a tag and inserts that into the control. 

**Before:**  
`<toggleButton id="testTogglePanelButton" onAction="PanelShown" getPressed="PanelShown" label="Show Panel" showImage="false" />`  
**After:**  
`<toggleButton id="testTogglePanelButton" onAction="PressedOnAction" getPressed="GetPressed" label="Show Panel" showImage="false" tag="RibbonType1testTogglePanelButton" />`  

The RibbonFactory has all known callback signatures defined, so when Office invokes one of the callbacks, it invoke a method on the RibbonFactory.  
RibbonXml callbacks have the control that initiated the callback available, and the control has the associated context, so the RibbonFactory will grab the context, then ask the `ViewModelResolver` to locate the current active view for that context. It then uses the Tag to know what the original callback was, and it will invoke that callback for you.

Because it handles all the callbacks manually, it has no problems calling a Property getter or setter instead of a method. It also has quite nice error reporting when the viewmodel has the wrong method signature etc, unlike VSTO.

#Setup/Getting Started
Getting started using the Ribbon Factory is quite easy, in your ThisAddIn.cs you have to override the `IRibbonExtensibility CreateRibbonExtensibilityObject()` and create a instance of the RibbonFactory for your office application. i.e OutlookRibbonFactory, WordRibbonFactory etc. 

Then in the Internal Startup method you need to add the bootstrapping and initialisation code. The TwitterFeed demo project I have looks like this:

    public partial class ThisAddIn
    {
        private AddinBootstrapper _core;

        private static void ThisAddInStartup(object sender, EventArgs e)
        {
            //WPF Support
            if (System.Windows.Application.Current == null)
                new Application { ShutdownMode = ShutdownMode.OnExplicitShutdown };
        }

        protected override IRibbonExtensibility CreateRibbonExtensibilityObject()
        {
            return new OutlookRibbonFactory(typeof(AddinBootstrapper).Assembly);
        }

        private void ThisAddInShutdown(object sender, EventArgs e)
        {
            _core.Dispose();
            System.Windows.Application.Current.Shutdown();
        }

        private void InternalStartup()
        {
            _core = new AddinBootstrapper();
            OutlookRibbonFactory.SetApplication(Application);
            RibbonFactory.Current.InitialiseFactory(
                t => (IRibbonViewModel)_core.Resolve(t),
                CustomTaskPanes);

            Startup += ThisAddInStartup;
            Shutdown += ThisAddInShutdown;
        }
    }

#Summary
The Ribbon Factory in VSTO Contrib I think is quite awesome, it gives a heap of features and makes VSTO development far easier. And it really complements the work the VSTO team has done around custom task panes. I will be exposing more VSTO related stuff as time goes by.

Now go grab the bits from [http://vstocontrib.codeplex.com/releases][2] (very soon, for now source =))


  [1]: http://www.paulstovell.com/
  [2]: http://vstocontrib.codeplex.com/releases