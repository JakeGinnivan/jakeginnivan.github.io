---
layout: post
title: Cleaning up Office COM Interop Code with Extension Methods
metaTitle: Cleaning up Office COM Interop Code with Extension Methods
description: Office Automation and COM interop code can be really ugly, this post show you how the extension methods in VSTO Contrib can help
revised: 2011-04-05
date: 2011-04-05
categories: [vsto-contrib,com-interop,vsto]
migrated: true
comments: true
sharing: true
footer: true
permalink: /vsto-contrib/com-cleanup-extension-methods/
summary: | 
  

---
Office Automation and COM interop code can be really ugly, this post show you how the extension methods in VSTO Contrib can help
<!-- more -->
#Ugly Office Automation Code

You have been tasked with the simple job of writing a console application which goes through every contact in an Outlook addressbook and replacing Smith, John with John Smith. Because you want Outlook 2003 to close properly when you call quit, you make sure you manage your references properly. The code you come up with looks something like:


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

The above code and way too much code to easily scan and see what is going on.

##Better code
So what does it look like using the simple helpers:

    using (var app = new Application().WithComCleanup())
    {
        using (var session = app.Resource.Session.WithComCleanup())
        {
            var folder = session.Resource.GetDefaultFolder(OlDefaultFolders.olFolderContacts);
            using (var contactsFolder = folder.Resource.WithComCleanup())
            {
                foreach (var contactItem in contactsFolder.Resource.Items.ComLinq<_ContactItem>())
                {
                    if (!string.IsNullOrEmpty(contactItem.FullName))
                    {
                        contactItem.FileAs = contactItem.FullName;
                        //contactItem.Save();
                    }
                }
            }
        }
        app.Quit();
    }

#Introducing Dynamic Proxies
The reason we can't do this, is the COM land doesn't understand IDisposable. So VSTO Contrib has a generated set of interfaces that look like this:

    public interface IApplication : Microsoft.Office.Interop.Outlook.Application, IDisposable { }

The .WithComCleanup actually is also generated, so it looks like this:

    public static IApplication WithComCleanup(this Outlook.Application resource)
    {  return resource.WithComCleanup<Outlook.Application, IApplication>(); }

The generic version will generate a proxy on the fly, and will call Marshal.ReleaseComObject on the proxied resource when Dispose is called. The end result is we now no longer have .Resource scattered all through our code, which again improves readability.

    using (var app = new Application().WithComCleanup())
    {
        using (var session = app.Session.WithComCleanup())
        {
            var folder = session.GetDefaultFolder(OlDefaultFolders.olFolderContacts);
            using (var contactsFolder = folder.WithComCleanup())
            {
                foreach (var contactItem in contactsFolder.Items.ComLinq<_ContactItem>())
                {
                    if (!string.IsNullOrEmpty(contactItem.FullName))
                    {
                        contactItem.FileAs = contactItem.FullName;
                        //contactItem.Save();
                    }
                }
            }
        }
        app.Quit();
    }

I think you would agree that this code is far better than the first example. And the difference becomes more dramatic as the complexity grows.