---
layout: post
title: Convention Tests v2 Released
metaTitle: Convention Tests v2 Released
description: ConventionTests is a TestStack project allowing you to easily validate and document your project conventions
revised: 2013-09-19
date: 2013-09-19
categories: [TestStack]
migrated: true
comments: true
sharing: true
footer: true
permalink: /conventiontests-v2-released/
summary: | 
  ConventionTests is a TestStack project allowing you to easily validate and document your project conventions

---
## ConventionTests v2 Released
[Krzysztof Koźmic](https://github.com/kkozmic) first spoke about ConventionTests at NDC 2012. You can find the video of that talk [here](http://vimeo.com/43676874), slides [here](http://kozmic.pl/presentations/) and the introductory blog post [here](http://kozmic.pl/2012/06/14/using-conventiontests/).

In v2, we have rewritten convention tests from the ground up to make it easier to get started, bundle some default conventions and also decouple it from a specific unit testing framework.

There is still plenty we can make better, so please raise issues on github with suggestions!

## What is ConventionTests?

Convention over Configuration is a great way to cut down repetitive boilerplate code. But how do you validate that your code adheres to your conventions? Convention Tests is a code-only NuGet that provides a simple API to build validation rules for convention validation tests.
<!-- more -->
## Getting Started
It is really easy to get started with Convention Tests, we have included a bunch of conventions out of the box. The included conventions are:

 - All Classes Have Default Constructor 
 - All Methods are Virtual
 - Class type has specific namespace (for example, all dtos must live in the ProjectName.Dtos namespace)
 - Files are Embedded Resources
 - Project does not reference dlls from Bin or Obj directory
 - Plus others will be added!

### Writing your first Convention test
#### 1. Using your favourite testing framework, create a new test. Lets call it `nhibernate_entities_must_have_default_constructor`

#### 2. Define some data
At the moment there is minimal support for type scanning, but better support will be added soon!

    var itemsToVerify = typeof (SampleDomainClass).Assembly.GetTypes();
    var nhibernateEntities = new Types("nHibernate Entitites")
    {
        TypesToVerify = itemsToVerify
    };
    
#### 3. Assert the convention
`Convention.Is(new AllClassesHaveDefaultConstructor(), nhibernateEntities);`

#### That's it!
When you run this convention, if it fails an exception will be thrown, which will look something like this:

    ConventionFailedException
    Message = Failed: 'Types must have a default constructor' for 'nHibernate Entitites'
    --------------------------------------------------------------------------
	
    TestAssembly.ClassWithNoDefaultCtor
    TestAssembly.ClassWithPrivateDefaultCtor

How cool is that!

### Reporting
If you would like to use ConventionTests reporting features, you just have to opt in by specifying the reporter you want. This makes it easy to add your own reporters, for example a WikiReporter may be better than the `HtmlReporter`

In your `Properties\AssemblyInfo.cs` file add the reporters you want. This are global reporters which will report the results of all conventions.

    [assembly: ConventionReporter(typeof(HtmlConventionResultsReporter))]
    [assembly: ConventionReporter(typeof(MarkdownConventionResultsReporter))]

Then if you look in the directory where your test assembly is, there will be an html report called `Conventions.htm`, serving as living documentation!