---
layout: post
title: VSTO and COM Interop
metaTitle: VSTO and COM Interop
description: VSTO and COM Interop are two different things entirely, but you can't use VSTO without COM Interop. This post will help you do COM interop right.
revised: 2011-02-19
date: 2010-05-14
categories: [vsto,com-interop,patternsandpractices,.net,c#]
migrated: true
comments: true
sharing: true
footer: true
permalink: /vsto-com-interop/
summary: | 
  

---
In this post I will give an introduction to COM Interop and covering some of the basic concepts you need to understand when dealing with VSTO and the Office Object Model. By understanding the way COM interop works and the potential impact of not deterministically cleaning up your references you will build much more reliable VSTO add-ins.

After covering the basics of COM Interop I will write about a helper library I have written and have code examples of how it can make your life much easier.

#COM Interop Overview
Hopefully you have a bit of a understanding about how it works, but I really want to explain my experiences and pull some of the high level concepts that will help you develop against the Office Object Model..

For any .NET application to talk to unmanaged code we need a .NET Interop Assembly which contains meta data about the types exposed in the COM component. When you add a reference to a COM component through the Add References dialog, visual studio will call the command line tool Tlbimp.exe and generate us a type library for that COM component, then the reference will be added to that Interop Assembly that we have generated. Publishers will often provide Primary Inerop Assemblies (PIA’s) which are signed Interop Assemblies by the publisher which prevent conflicts between different applications having their own generated Interop assemblies. I will refer to both as PIA’s from now on.

Once we have our PIA’s .NET makes it very easy for us to interact with the COM types. Whenever a COM object crosses into our .NET application the CLR creates a runtime callable wrapper (RCW) which consumes the IUnknown (which provides object identity, type coercion and lifetime management) and IDispatch (used for late binding through reflection) and gives us an instance of the RCW which implements the interface of the type we requested. See below where the .NET client is given a RCW implementing the INew interface.
![RCW][1]

There is a RCW created for every instance of a particular COM object. This means we can have references to a single RCW in multiple area’s of our application.

##Memory Models
So all sounds really easy for the moment, we add a reference to our PIA’s, then .NET will do its best to make it seem like we are talking to a managed library and try to take care of all the memory management itself, after all that is what we are used to as .NET developers.

The problem is .NET memory management is non-deterministic, meaning we do not know when our memory will be cleaned up by the garbage collector. COM on the other hand is unmanaged and follows a deterministic memory cleanup model. These models do not work well together.
![Memory Models][2]

What is actually happening in the above diagram is when the garbage collector runs, it finds a RCW that has no references inside the managed process the garbage collector proceeds to clean up the RCW. The RCW actually implements a finaliser which releases all the associated COM references.

By relying on the finaliser of the RCW it means that it requires at least two garbage collections to clean up our COM references. If you want to read about this in more detail check out [http://msdn.microsoft.com/en-us/magazine/bb985010.aspx][3].

Microsoft have provided us with a method to clean up our COM references in a deterministic way. When you call Marshal.ReleaseComObject on a COM object it decrements the RCW’s reference counter, when this counter reaches 0 the underlying COM references are released. By calling ReleaseComObject on every COM object we request when we are finished with it we bring the two memory models closer together and can avoid hard to diagnose ‘ghost’ inspectors in Outlook (will explain later in the post).

##How to achieve deterministic cleanup in .NET

Now that I have covered the basics of COM interop, we can look at ways to code in a deterministic manner.
###Only ever use a single . (period) in lines of code
**Example:** xlApp.Workbooks.Add()

What is happening behind the scenes is we have a RCW for the _ExcelApplication interface, .Workbooks gives us a RCW implementing _WorkBooks interface, then we call the Add method which returns a _Workbook RCW.

We have just lost the reference to a WorkBooks COM object. You can never force the release of those resources now, and Excel 2003 will not exit without killing the process (2007 has code that detects leaked objects and will clean up after you on exit).

<h3>Do not call Marshal.ReleaseComObject on an object that has left the current scope</h3>
f you let a COM object leave the scope it was instantiated in you probably can no longer guarantee that you have the only reference in your application. If you call Marshal.ReleaseComObject, the next time another area of your app makes a call to that COM object you will get a InvalidComObjectException thrown.
![InvalidComObjectException][4]

The Visual Studio team posted on their blog a really interesting post titled “Marshal.ReleaseComObject Considered Dangerous“. Have a read at [http://blogs.msdn.com/visualstudio/archive/2010/03/01/marshal-releasecomobject-considered-dangerous.aspx][5]

###Call Marshal.ReleaseComObject on every COM object you bring into the managed process. (That does not leave that method scope)

Every COM object you get by accessing a property or calling a method you should release. For techniques on how to not let the COM object leave the current method scope check out my last blog post covering [Outlook data repositories][6]. Your goal should be to not let those COM objects leave the current scope, because once they do you shouldn’t release them explicitly anymore.

###Release each item that is enumerated in a foreach loop
Foreach loops are a very big trap. Take: 
foreach (var item in folder.Items) { }

That simple line of code leaks an Items collection, and every item that is enumerated is also leaked. I have a helper extension that makes it much easier. I will cover it closer to the end of the post.

##Issues with not releasing
It is not just memory or ‘best practise’ that has made me look into this so much, there are many REAL problems that can be caused if you do not release your COM resources.

###Application doesn’t exit
If you have ever done excel automation and used COM interop to generate excel documents on a server (I don’t recommend this by the way  ) then you probably have found that even though you call .Exit() on the excel API the process still hangs around. This is because you have missed releasing a resource, or released them in the wrong order.

###Ghost Inspectors
If you have used Outlook a lot, you probably would have experienced this and just put it down to something strange happened. Here is what happens, I open a contact.

Then I press Save & Close and am left with

![Ghost Inspector][7]

The ribbon gets greyed out. What has happened is the Inspector (the window) is no longer associated to the item it was displaying, but because of a leaked reference the Inspector did not close correctly. You CANNOT close this window through code (unless you find the window and close it through WIN32). I will cover this in more detail in another post, I still have not figured out how to not cause ghost inspectors sometimes when the inspector is opened modally.

###Unable to resize appointments in calendar

One add-in I wrote synchronised calendar items. After a synchronisation you could no longer resize items in your calendar. This was due to not releasing each Appointment and all the child properties I accessed (user properties).

As you can see, by not releasing the resources you access you can hit very hard to diagnose issues. You are much better off enforcing the I must release everything I access. (It is also a good idea to check Marshal.IsComObject() before you call release. Maybe put it in an extension method or a static helper method. Check out how I have done it in Outlook.Utility.

##How to make this easier
I have spent some time creating some extension methods which makes dealing with COM objects much cleaner, and safer.

The first problem I wanted to reduce the amount of boiler plate code to make sure you deterministically clean up your COM references.

Enter the WithComCleanup extension method. Take this code:

    void Main()
    {
        _Application app;
        _NameSpace session;
        _MAPIFolder folder;
        Items contactItems;
        try
        {
            app = new Application();
            session = app.Session;
            folder = session.GetDefaultFolder(OlDefaultFolders.olFolderContacts);
            contactItems = folder.Items;
            foreach (_ContactItem contactItem in folder.Items)
            {
                try
                {
                    if (!string.IsNullOrEmpty(contactItem.FullName))
                    {
                        contactItem.FileAs = contactItem.FullName;
                        contactItem.Save();
                    }
                }
                finally
                {
                    if (Marshal.IsComObject(contactItem))
                        Marshal.ReleaseComObject(contactItem);
                }
            }
        }
        finally
        {
            if (contactItems != null && Marshal.IsComObject(contactItems))
                Marshal.ReleaseComObject(contactItems);
            if (folder != null && Marshal.IsComObject(folder))
                Marshal.ReleaseComObject(folder);
            if (session != null && Marshal.IsComObject(session))
                Marshal.ReleaseComObject(session);

            if (app != null)
            {
                app.Quit();
                if (Marshal.IsComObject(app))
                    Marshal.ReleaseComObject(app);
            }
        }
    }

Each COM variable must be declared out of the try/finally scope, and for each COM object we have to check if its null, and that it is a COM object (not only can you use COM objects in .NET, you can also implement a COM interface in .NET and pass it into the unmanaged application, you can read about COM Callable Wrappers at [http://msdn.microsoft.com/en-us/library/f07c8z1c(VS.71).aspx][8]), but this means you can get a .NET object when you are expecting a COM object, as more parts of Office become managed this will mean you code will not break.

Now lets use the WithComCleanup extension method:

    void Main()
    {
        using (var app = new Application().WithComCleanup())
        {
            using (var session = app.Resource.Session.WithComCleanup())
            {
                var folder = session.Resource.GetDefaultFolder(OlDefaultFolders.olFolderContacts);
                using (var contactsFold = folder.WithComCleanup())
                {
                    foreach (var contactItem in contactsFold.Resource.Items.ComLinq<_ContactItem>())
                    {
                        if (!string.IsNullOrEmpty(contactItem.FullName))
                        {
                            contactItem.FileAs = contactItem.FullName;
                            contactItem.Save();
                        }
                    }
                }
            }
            app.Resource.Quit();
        }
    }

This code is functionally identical to the last snippet. What is happening is our COM resource is being wrapped in a AutoCleanup<T> object which implements IDisposable, which allows you to use a using statement to wrap each com object. It means more indentation, but I think it results in much cleaner code.

Another thing you may notice is you now must access your COM object through the .Resource property. Lets have a look at another example. In this example I simply want to access a custom property on each Contact in Outlook.

    void Main()
    {
        _Application app;
        _NameSpace session;
        _MAPIFolder folder;
        Items contactItems;
        try
        {
            app = new Application();
            session = app.Session;
            folder = session.GetDefaultFolder(OlDefaultFolders.olFolderContacts);
            contactItems = folder.Items;
            foreach (_ContactItem contactItem in folder.Items)
            {
                UserProperties userProperties;
                UserProperty property;
                try
                {
                    //Create a user property on every contact
                    userProperties = contactItem.UserProperties;
                    property = userProperties.Find("CustomProperty", true);
                    if (property == null)
                        property = userProperties.Add(name, OlUserPropertyType.olText, false, Type.Missing);

                    property.Value = "Value";
                    //Dont save, just a demo
                }
                finally
                {
                    if (Marshal.IsComObject(property))
                        Marshal.ReleaseComObject(property);
                    if (Marshal.IsComObject(userProperties))
                        Marshal.ReleaseComObject(userProperties);
                    if (Marshal.IsComObject(contactItem))
                        Marshal.ReleaseComObject(contactItem);
                }
            }
        }
        finally
        {
            if (contactItems != null && Marshal.IsComObject(contactItems))
                Marshal.ReleaseComObject(contactItems);
            if (folder != null && Marshal.IsComObject(folder))
                Marshal.ReleaseComObject(folder);
            if (session != null && Marshal.IsComObject(session))
                Marshal.ReleaseComObject(session);

            if (app != null)
            {
                app.Quit();
                if (Marshal.IsComObject(app))
                    Marshal.ReleaseComObject(app);
            }
        }
    }

This is still a REALLY simple app, I think there is probably 80% boiler plate code to ensure all our COM references are cleaned up and Outlook shuts down gracefully.

Enter the ComLinq and the Get/SetUserProperty extension methods.

    void Main()
    {
        using (var app = new Application().WithComCleanup())
        {
            using (var session = app.Resource.Session.WithComCleanup())
            {
                var folder = session.Resource.GetDefaultFolder(OlDefaultFolders.olFolderContacts);
                using (var contactsFolder = folder.WithComCleanup())
                using (var contactItems = contactsFolder.Resource.Items.WithComCleanup())
                {
                    foreach (var contactItem in contactItems.ComLinq<_ContactItem>())
                    {
                        contactItem.SetPropertyValue("CustomProperty", OlUserPropertyType.olText, "Value", addToFolder: false);
                    }
                }
            }
            app.Resource.Quit();
        }
    }

That is a massive reduction in code, lets look at what is actually happening here.

First lets look at our loop:

    foreach (var contactItem in contactItems.ComLinq<_ContactItem>())

Under the covers the ComLinq extension will wrap any IEnumerable (not the generic version) in custom generic IEnumerable, which has a IEnumerator that disposes the COM objects it enumerates as it enumerates the collection.

The key thing to remember is that the items MUST NOT leave the scope of the foreach loop because the RCW’s will be poisoned as soon as the enumerator moves to the next item.

    contactItem.SetPropertyValue("CustomProperty", OlUserPropertyType.olText, "Value", addToFolder: false);


Now lets have a look at the SetPropertyValue extension method (there is also a GetPropertyValue extension method as well).

It takes 4 arguments, first is the name of the Property, second is the type, third is the value we want to set, then 4th is a flag to add the UserProperty to the parent folder, this allows you to search your custom property through the Outlook UI. For example in my FacebookToOutlook demo application I use it to filter appointments to facebook events.
![Custom Folder Property][9]

The signature of the GetUserProperty extension method looks like this:

    T GetPropertyValue<T>(UserProperties userProperties, string name, OlUserPropertyType type, bool create, Func<object, T> converter, T defaultValue)

And you can call it like this:

    _appointmentItem.GetPropertyValue(FacebookeventidProperty, OlUserPropertyType.olText, false, Convert.ToInt64, -1)

As you can see it simplifies the syntax greatly, by using type intference with the converter and the default value we get a strongly typed way to get custom properties that cleans up all the COM resources it accesses deterministically.

Check out these extensions plus other helpers in my Outlook.Utility project at http://vstocontrib.codeplex.com/ to get the source code and see what other useful helpers are in that library.

#Wrap up

This has been a pretty big post, which I hope is useful to people. I have not covered events and quite a lot of other things, but starting with knowing how COM interop basically works and deterministically cleaning up as many COM resources are you can make your life much easier.

If you have feedback, please contact me.

##Resources
If you would like to read more about this check out these links:

[http://www.guidanceshare.com/wiki/Interop_(.NET_1.1)_Performance_Guidelines_-_Marshal.ReleaseComObject][10] – DotNet 1.1, but still good

[http://blogs.msdn.com/mbend/archive/2007/04/18/the-mapping-between-interface-pointers-and-runtime-callable-wrappers-rcws.aspx][11]

[http://www.add-in-express.com/creating-addins-blog/2008/10/30/releasing-office-objects-net/][12] – I don’t use add-in express. But the content on this page is good.

[http://msdn.microsoft.com/en-us/magazine/cc163316.aspx][13] - Managing Object Lifetime, good stuff, talks about deterministic finalisation in .NET

[http://samteknik.blogspot.com/][14] – It looks like Samir only started blogging late last year, but the two posts he has on his blog are a good read.

[http://msdn.microsoft.com/en-us/library/8023ct8s(VS.100).aspx][15] – COM interfaces explanation

[http://msdn.microsoft.com/en-us/library/8bwh56xe(VS.100).aspx][16] – RCW description

[http://blogs.msdn.com/vcblog/archive/2006/09/20/762884.aspx][17] - Mixing deterministic and non-deterministic cleanup


  [1]: /get/screenshots/RCW.png
  [2]: /get/screenshots/MemoryModels.png
  [3]: http://msdn.microsoft.com/en-us/magazine/bb985010.aspx
  [4]: /get/screenshots/InvalidComObjectException.png
  [5]: http://blogs.msdn.com/visualstudio/archive/2010/03/01/marshal-releasecomobject-considered-dangerous.aspx
  [6]: /vsto-data-access-repositories
  [7]: /get/screenshots/GhostInspector.png
  [8]: http://msdn.microsoft.com/en-us/library/f07c8z1c(VS.71).aspx
  [9]: /get/screenshots/OutlookCustomerFolderProperty.png
  [10]: http://www.guidanceshare.com/wiki/Interop_(.NET_1.1)_Performance_Guidelines_-_Marshal.ReleaseComObject
  [11]: http://blogs.msdn.com/mbend/archive/2007/04/18/the-mapping-between-interface-pointers-and-runtime-callable-wrappers-rcws.aspx
  [12]: http://www.add-in-express.com/creating-addins-blog/2008/10/30/releasing-office-objects-net/
  [13]: http://msdn.microsoft.com/en-us/magazine/cc163316.aspx
  [14]: http://samteknik.blogspot.com/
  [15]: http://msdn.microsoft.com/en-us/library/8023ct8s(VS.100).aspx
  [16]: http://msdn.microsoft.com/en-us/library/8bwh56xe(VS.100).aspx
  [17]: http://blogs.msdn.com/vcblog/archive/2006/09/20/762884.aspx