---
layout: post
title: Async warning with nSubstitute
metaTitle: New post
description: 
revised: 2012-10-16
date: 2012-10-16
categories: [Async, Testing]
migrated: true
comments: true
sharing: true
footer: true
permalink: /async-warning-with-nsubstitute/
summary: | 
  

---
I got sick of a heap of warnings in VS which look like this:

    Warning	1	Because this call is not awaited, execution of the current method continues before the call is completed. Consider applying the 'await' operator to the result of the call.

The code which was causing this error was a received call: `_service.Received().CancelAsync()`

My solution to the problem is creating a simple extension

    public static class NSubstituteHelper
    {
        public static void IgnoreAwaitForNSubstituteAssertion(this Task task)
        {

        }
    }

Then my code turns into `_service.Received().CancelAsync().IgnoreAwaitForNSubstituteAssertion()` and my issue goes away