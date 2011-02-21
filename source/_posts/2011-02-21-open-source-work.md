---
layout: post
title: Open Source Work
metaTitle: Open Source Work
description: A collection of the open source projects I am currently contributing to, or have created.
revised: 2011-02-21
date: 2011-02-21
categories: [open-source]
migrated: true
comments: true
sharing: true
footer: true
permalink: /open-source-work/
summary: | 
  

---
# My Projects

###[VSTO Contrib][1]
A Utility Library for either Office Automation/COM Interop or VSTO. One of the main goals is to make working with COM Interop easier, and add support for IoC containers and DI into VSTO Add-ins.

###[Fibre Async Library][2]
The purpose for this library is to make it simple to coordinate and schedule async work. It allows multiple async pattern (BeginGet/EndGet style) or Any delegate (Action/Func etc) calls to be run simultaneously and coordinated with a callback when everything is completed.

It has a much lower barrier to entry than Rx or TPL, and runs on .NET, Silverlight and WP7

###[SqlMetalInclude][3]
A basic utility which allows you to enter a database, select the tables you want to include, or exclude. It will then generate you a .cmd file which will regenerate your Linq To Sql dbml file and associated code generated models.
I quickly needed access to a CRM database which had a huge amount of tables, i just wanted a subset. Quite old, not maintained, but still useful if you use LinqToSql.

# Contributing to

###[MahTweets for Windows Phone 7][4]
The popular open source twitter client, now on Windows Phone 7. 


###[Columbus MVC Framework for WP7][5]
MVC Framework for WP7

###[FunnelWeb Blog][6]
A awesome MVC3 based blog engine using Razor as the view engine. Also is powering my blog =)

###[AutoMapper][7]
I maintain a fork of automapper which adds mapping inheritance to automapper, making it far more natural to use. It can also significantly cut down mapping code in a project with lot's of inheritance.

 
  [1]: http://vstocontrib.codeplex.com/
  [2]: http://fibre.codeplex.com/
  [3]: http://sqlmetalinclude.codeplex.com/
  [4]: http://mahtweetswp7.codeplex.com/
  [5]: http://columbus.codeplex.com/
  [6]: http://code.google.com/p/funnelweb/
  [7]: https://github.com/JakeGinnivan/AutoMapper/tree/NET35