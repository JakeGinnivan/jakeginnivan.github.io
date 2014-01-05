---
layout: post
title: No more R# warnings for nHibernate aliases
metaTitle: No more R# warnings for nHibernate aliases
description: When using QueryOver projections with type safe aliases, r# will warn that the value is always null. Here is a trick to remove the warning
revised: 2011-07-22
date: 2011-07-21
categories: [nHibernate,resharper,c#]
migrated: true
comments: true
sharing: true
footer: true
permalink: /nhibernate-alias-warnings/
summary: | 
  When using QueryOver projections with type safe aliases, r# will warn that the value is always null. Here is a trick to remove the warning

---
I dislike having R# warnings in code bases I work on, and try to leave every class I work on having a green status.

Here is a scenario

    var entryAlias = default(Entry);
    var entries = session.QueryOver<Entry>(()=>entryAlias);

In the above scenario R# will warn that entryAlias is always null, which is is, but I don't care, nHibernate is simply getting the type from the expression.
<!-- more -->
So I have created a simple static class:

    /// <summary>
    /// Simple helper class to remove r# warnings when using nHibernate aliases
    /// </summary>
    public static class Alias
    {
        public static T For<T>()
        {
            return default(T);
        }
    }

And we turn the above code into:

    var entryAlias = Alias.For<Entry>();
    var entries = session.QueryOver<Entry>(()=>entryAlias);

Which does not warn.

Happy Resharpering