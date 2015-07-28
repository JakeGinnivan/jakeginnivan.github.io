---
layout: post
published: true
title: GitVersion v3.0.0!
comments: true
---

Today GitVersion v3.0.0 has been released. For me it has been a long time coming, with 4 beta releases and incorporating a lot of feedback between each release. It all started with [this pull request](https://github.com/GitTools/GitVersion/pull/338) which replaced the existing approach of each git workflow being hard coded with a class responsible for calculating each branch type.

This meant that GitFlow and GitHubFlow had completely separate code paths with very little shared, it also meant that things like commit counting varied on different branches. v3 changed all that and now is entirely driven by configuration. This initial pull request weighed in at 78 commits, 196 files, changed 2,500 lines of code and deleted another 700. 106 pull requests later we are shipping.

This configuration based approach has been really important for a number of reasons:

1. Understanding/documenting what GitVersion is doing.
  - When we had bug reports of a version being incorrectly calculated there was never a generic fix.
  - The logic for every branch was different and inconsistent. Then GitFlow and GitHubFlow worked quite differently in some respects
2. It was hard to get GitVersion to work with nightlies and publishing CI builds to NuGet
  - This meant that getting GitVersion working with Octopus deploy was not great

A number of long standing bugs were fixed with the changes made in v3, some of the highlights of the new features

 - `GitVersion init` is a configuration tool which allows you to make GitVersion work the way you want it to
 - Decent documentation!
   - Check it out at [http://gitversion.readthedocs.org/en/latest](http://gitversion.readthedocs.org/en/latest)
 - Configurable version incrementing
   - [Continuous Delivery](http://gitversion.readthedocs.org/en/latest/reference/continuous-delivery/) (default) means GitVersion will only increment the version when you release and tag, this works great for some scenarios and not well for otherwise
   - [Continuous Deployment](http://gitversion.readthedocs.org/en/latest/reference/continuous-deployment/), meaning each commit will produce a *different* SemVer
   - Read more at [http://gitversion.readthedocs.org/en/latest/more-info/version-increments/]
 - Better error reporting - when things go wrong (which should be a lot less often now) we try to give you decent information including:
   - Dumping out a cleaned git history graph which you can paste into an issue so we can reproduce the issue
   - Logging out all decisions and sources for versions

## Whats next
There are a few features which will be released in 3.1 which will make GitVersion nicer to get started. v4 hopefully will be not too far down the road when we rethink the command line usage and fix a few other things which have got a bit complex over time.

## Give it a go
All the links you need are on our GitHub page at [https://github.com/GitTools/GitVersion#gitversion](https://github.com/GitTools/GitVersion#gitversion)
