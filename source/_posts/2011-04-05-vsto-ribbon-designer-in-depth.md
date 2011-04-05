---
layout: post
title: Ribbon XML and Ribbon Designer in Depth
metaTitle: Ribbon XML and Ribbon Designer in Depth
description: Ever wanted to know a bit more about how VSTO gives you the Ribbon designer. This post will explain Ribbon XML, then how the designer gives you more.
revised: 2011-04-05
date: 2011-04-05
categories: [vsto]
migrated: true
comments: true
sharing: true
footer: true
permalink: /vsto-ribbon-designer-in-depth/
summary: | 
  

---
#Ribbon Designer
One of the goals when VSTO 'RAD' or Rapid Application Development, where you get your draggy/droppy style of development. Which is exactly what the Ribbon designer gives you, a really simple, winforms like way of building up your ribbons. It gives you:

 - Ability to easily target specific ribbon types, for example Microsoft.Outlook.Mail.Read or Microsoft.Excel.Workbook
 - Context of the Ribbon, which is the Document, or Inspector etc.
 - Familiar programming model, and IDE support. Simply handle click event through designer, it generates the callback. Super Simple.

Which is great, the reason I have used it in the past is for the Context, Ribbon XML doesn't give you context, and it is really hard to get at the current context other than using the Globals.ThisAddIn.Application.ActiveExplorer properties.

#How Ribbons work in Office
It all starts with `Microsoft.Office.Core.IRibbonExtensibility` this simple interface has a single method on it:  
`string GetCustomUI(string RibbonID)`

In your ThisAddIn.cs you can override the `CreateRibbonExtensibilityObject` method to provide a custom implementation of `IRibbonExtensibility`. By default it is the VSTO RibbonFactory.

When office starts up before any ribbon is displayed, it will call this method ONCE for each ribbon type. For example, when Outlook starts up it will call it passing `RibbonId = 'Microsoft.Outlook.Explorer'`, then when you open an email to read, it will be called again with `RibbonId = 'Microsoft.Outlook.Mail.Read'`.

Your job when implementing this interface, is to return the ribbon xml for the requested ribbon, and null if you don't have anything useful.

#Ribbon XML
Ribbon XML itself is really simple, you define the ribbon structure through xml, it is quite logical and I have no issues as I write xaml all the time which is basically the same thing. This is what it looks like:

    <?xml version="1.0" encoding="UTF-8"?>
    <customUI onLoad="Ribbon_Load" xmlns="http://schemas.microsoft.com/office/2006/01/customui">
        <ribbon>
            <tabs>
                <tab idMso="TabAddIns">
                    <group id="group1" label="group1">
                        <button id="button1" label="button1" showImage="false" />
                    </group>
                </tab>
            </tabs>
        </ribbon>
    </customUI>

You can read more about the details of the format at [http://msdn.microsoft.com/en-us/library/aa942866.aspx][1]

##Callbacks
This is where Ribbon XML becomes a bit ugly, if you have Ribbon XML for two different ribbons more, all callbacks must be on your IRibbonExtensibility class, which can make this a god object and the root of your entire application.

#How the Ribbon Designer works
You might now be wondering, if you can only specify a ribbon extensibility class, and all callbacks must be on the same class, how can I have multiple ribbon designers in a single file?

Under the hood the Ribbon Designer is actually extremely complex. When you run your add-in a few things happen.

 1. The VSTO Ribbon Factory scans all assemblies for any classes implementing `Microsoft.Office.Tools.Ribbon.IRibbonExtension` (4.0) or `Microsoft.Office.Tools.Ribbon.OfficeRibbon` (3.5). This can be expensive for large add-ins, or add-ins that reference large assemblies.

 2. As far as I can tell (based off spending some time in reflector), is that the RibbonFactory will reflect over each of your ribbons, then generate RibbonXML for your ribbon designer, this will be returned to Office when it requests that ribbon type.

 3. When a new Context is created (new document, inspector etc) then the RibbonFactory will new up the appropriate ribbon designer class, and set the context for you.

 4. When Office calls back to the RibbonFactory (for a button click etc.) then it will figure our which ribbon designer object the callback is for, then invoke the appropriate event.

A bit of this is speculation, but working through this problem myself, it cannot be that far off the way things actually work.

Point 3 is the most interesting, and the one I had the most issues around when creating my Ribbon Factory for VSTO Contrib.

You can read about the [VSTO Contrib Ribbon Factory][2] as well if you are interested.


  [1]: http://msdn.microsoft.com/en-us/library/aa942866.aspx
  [2]: vsto-contrib/ribbon-factory