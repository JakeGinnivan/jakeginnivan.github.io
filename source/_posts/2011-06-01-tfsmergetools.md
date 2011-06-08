---
layout: post
title: TFS Merge Tools Configuration
metaTitle: TFS Merge Tools Configuration
description: Every time I format my machine, I have to think of all the different file extensions I work with. Here is my list.
revised: 2011-06-08
date: 2011-06-01
categories: [tfs,.net]
migrated: true
comments: true
sharing: true
footer: true
permalink: /tfsmergetools/
summary: | 
  Every time I format my machine, I have to think of all the different file extensions I work with. Here is my list.

---
When I first install Visual Studio and TFS Explorer, high on my priorities is to configure a decent merge tool.

This blog post [http://blogs.msdn.com/b/jmanning/archive/2006/02/20/diff-merge-configuration-in-team-foundation-common-command-and-argument-values.aspx][1] lists all common merge tools, and the command line arguments for each of them.

I use KDiff, so the Compare arguments are `%1 --fname %6 %2 --fname %7` and Merge arguments are `%3 --fname %8 %2 --fname %7 %1 --fname %6 -o %4`

Great, except I have to register all the extensions that I want to use KDiff for. This is my list:

`.xml,.xaml,.cs,.csproj,.sln,.config,.msbuild,.txt,.cmd,.bat,.ps1,.nuspec,.xsd,.tasks,.xsl,.resx,.vb,.sql`

What other extensions am I missing? 

  [1]: http://blogs.msdn.com/b/jmanning/archive/2006/02/20/diff-merge-configuration-in-team-foundation-common-command-and-argument-values.aspx