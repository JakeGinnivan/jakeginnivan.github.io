---
layout: post
title: Pack URI in Unit Tests
metaTitle: Pack URI in Unit Tests
description: Some exceptions and solutions when you try to unit test around pack Uri's
revised: 2013-09-16
date: 2013-09-16
categories: [WPF]
migrated: true
comments: true
sharing: true
footer: true
permalink: /pack-uri-in-unit-tests/
summary: | 
  Some exceptions and solutions when you try to unit test around pack Uri's

---
I had a unit tests which constructed a pack uri, and I didn't want to abstract it (needless abstraction) so here is how I solved a few issues.

## Exception 1
### System.UriFormatException : Invalid URI: Invalid port specified
This one is pretty easy to fix, you can use the `PackUriHelper` which registers a few things in it's static ctor

    PackUriHelper.Create(new Uri("reliable://0"))

## Exception 2
### System.NotSupportedException : The URI prefix is not recognized
This one is fixed by giving WPF the default resource assembly.

    System.Windows.Application.ResourceAssembly = typeof(App).Assembly;

Now you should be able to unit tests around pack uri's