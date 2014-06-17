---
layout: post
published: false
title: GitFlow version numbers
comments: true
---

So it turns out this is quite a complex topic with different people having different goals. In GitVersion we wanted to come up with a strategy which solves most of the normal things people want to do in a sensible way and guide people in a direction which should *just work*.

This blog post covers GitFlow, I will follow up with a GitHubFlow version which has it's own set of issues.

This is a list of things people normally want to do when using GitFlow:

 - Have a CI build feed off develop which can be consumed to acheive continuous integration across packages
 - Be able to hotfix old versions, and create pre-release of that hotfix
 - Be able to create releases, and have beta's/rc's of that release
 - Release stable packages

So given those basic requirements, lets see how hard it can be.

