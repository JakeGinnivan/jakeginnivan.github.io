---
layout: post
title: Enterprise VSTO
metaTitle: VSTO Add-ins which are maintainable and well designed
description: VSTO Add-ins which are maintainable and well designed
revised: 2011-02-19
date: 2010-04-23
categories: [vsto,.net,c#]
migrated: true
comments: true
sharing: true
footer: true
permalink: /enterprise-vsto/
summary: | 
  

---
Its been a while since I have blogged, since starting with VSTO I haven’t posted much. That is about to change.

I have been playing around with VSTO a lot lately, mainly Outlook but I will move into Word and Excel very soon because I see a lot of business value in the VSTO platform.

I have started an open source library called Outlook.Utility, which I will probably break into Office.Utility as well to separate Office and Outlook functionality. There is a very good but old article on MSDN ([http://msdn.microsoft.com/en-us/library/aa479345.aspx][1]). In the code example in that article they have a Outlook.Utility project which has some really handy classes in it. I have cleaned up and enhanced a few of the classes, as well as adding my own.

The main issue with the code is it doesn’t release any COM references and actually leaks, as I invest more time into my Outlook.Utility project I will clean up all the leaks as well as add my own helper classes and extensions. I am going to start a Enterprise VSTO series that will use this library and cover ways you can create a good maintainable add-in.

Over a few posts I will cover IoC, Patterns to use, COM interop, performance issues/solutions, WPF integration (using MVVM) and a few other things. I then will probably put it all in a webcast at the end.

<h1>Enterprise VSTO Series</h1>

[Project Structure][2] <br />
[IoC, DI and VSTO][3] <br />
[Outlook Items, Repositories & Data Access][4] <br />
[VSTO and COM Interop][5]

More to come..


  [1]: http://msdn.microsoft.com/en-us/library/aa479345.aspx
  [2]: /vsto-project-structure
  [3]: /ioc-dependencyinjection-vsto
  [4]: /vsto-data-access-repositories
  [5]: /vsto-com-interop
