---
layout: post
title: VSTO Contrib
metaTitle: Open Source VSTO Contrib project started
description: VSTO Contrib project started with VSTO Helper classes, and Example Projects
revised: 2011-02-19
date: 2010-07-14
categories: [vsto,.net,open-source]
migrated: true
comments: true
sharing: true
footer: true
permalink: /vsto-contrib/
summary: | 
  

---
Over the time with working with VSTO, I have created a heap of reusable classes and extensions for VSTO. So I have decided to start a Contrib project.

[http://vstocontrib.codeplex.com/][1]

The best class (in my opinion) in the Office.Utility project is the RibbonFactory, it allows you to have a ribbon callback file for each xml ribbon, register all the callback files via IoC (or manually), then the factory will locate the associated Ribbon.xml file and wire up all the callbacks for you. I prefer working with the Ribbon XML model over the designer, but the limitation of only having a single callback class is really annoying. The ribbon factory gets around that 

Over the next few months I will write up some more posts about ways to really leverage VSTO, and also write maintainable apps.

  [1]: http://vstocontrib.codeplex.com/