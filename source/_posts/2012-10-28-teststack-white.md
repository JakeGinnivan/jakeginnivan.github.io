---
layout: post
title: Announcing TestStack.White
metaTitle: Announcing TestStack.White
description: White is a great rich client UI automation framework which has been inactive for a while. We are restarting the project!
revised: 2012-10-28
date: 2012-10-28
categories: [UI Automation, White, Open Source]
migrated: true
comments: true
sharing: true
footer: true
permalink: /teststack-white/
summary: | 
  White is a great rich client UI automation framework which has been inactive for a while. We are restarting the project!

---
# Project White
For those that don't know, project white has been around for ages. It has gone from Google Code ([https://code.google.com/p/white-project/](https://code.google.com/p/white-project/)) to Codeplex ([http://white.codeplex.com/](http://white.codeplex.com/)) then GitHub ([https://github.com/petmongrels/white](https://github.com/petmongrels/white)). 

The goal of White was to create a nice abstraction over Microsoft's UI Automation framework in a consistent and object orientated way.

White reached a nice maturity level and active development stopped about 2 years ago, since then things have moved on. 

[Vivek Singh](https://github.com/petmongrels) has given me permission to take this project over and try and increase the activity on this project again.

## Why I'm taking on White
Personally, I have been on a rather large WPF project on and off for the past 2.5 years and rely on UI Automation very heavily. When we started the project, White was the most mature UI Automation framework.

This is a snapshot of some of our builds, every CI build that succeeds with kick-off many UI automation test runs for different configurations.  
![TeamCity](/assets/posts/2012-10-28-teststack-white/UIAutomationTests.png)

Over the last few years, we have wrapped White and have been running a custom builds and have learnt a lot about UI automation. 

I would like to put some of the learnings and improvements back into White to make it easier for everyone to use UI automation without paying for one of the commercial offerings.  

One of my colleagues at [Readify](http://www.readify.net), [Mehdi Khalili](http://www.mehdi-khalili.com/) did a presentation at Perth .NET usergroup on UI Automation, he was suggesting many things that we were already doing on this project except he was focusing on Web Automation.
Mehdi created BBDify a while ago, which got renamed to BDDfy when Mehdi, [Michael Whelan](https://github.com/mwhelan) and [Krzysztof Koźmic](http://kozmic.pl/) formed [TestStack](http://teststack.github.com/).

After talking with Mehdi in the pub, he convinced me that I should restart White, and also join TestStack so we can keep some great testing related projects together!

## TestStack.White
I have been doing some updates and cleanups on White over the past month or two in a private repo, but I have now opened it up at [https://github.com/TestStack/White](https://github.com/TestStack/White).

I am looking forward to working on White some more, and I welcome pull requests :P