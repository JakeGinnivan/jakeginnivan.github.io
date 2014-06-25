---
layout: post
title: "Updating Chocolateys release notes with GitReleaseNotes"
date: 2014-06-25 22:51:20 +0100
comments: true
categories:  [Git, Release Notes, Open Source]
---

Today I saw a tweet from Rob Reynolds today that [Chocolatey 0.9.8.24 RC1 was released](https://twitter.com/ferventcoder/status/481816379340120064) so I clicked the link which was straight to the closed issues list on GitHub.

I also noticed that many of the RCs and betas were not tagged in Git so you can't see what was fixed each beta.

I have been working on a little utility to solve exactly this problem called **GitReleaseNotes**, the idea is that you install it via Chocolatey, then run `GitReleaseNotes /outputFile ReleaseNotes.md /allTags` and it will connect to your issue tracker (if the issue tracker is a REMOTE in your Git repository) fetch all the closed issues since it was last run and append them into your release notes. For public GitHub repos using GitHub issues these are the same so it just works, for Jira and YouTrack you will need to specify additional command line parameters

You then can manually edit, group and do whatever you want. All your modifications will not be changed when you run GitReleaseNotes again.

<!-- more -->

The first step was to change the formatting slightly, currently GitVersion assumes all release titles will start with `# <Release>`, so I changed all of the `##1.2.3 (release date)` to `# 1.2.3 (release date)`.

The next step was to tell GitReleaseNotes where to start from, to do this I needed to add the commit range of the last release, the most important one is the last sha. For this I just took when the release notes was updated last, then went back in the history until I found another major release. This gave me:

    Commits: [a32f1fc133...f15a8f3b52](https://github.com/chocolatey/chocolatey/compare/a32f1fc133...f15a8f3b52)

But this would work fine too

    Commits: a32f1fc133...f15a8f3b52

After I had done that I just ran `GitReleaseNotes /o CHANGELOG.md /allTags` and this was appended to the top of `CHANGELOG.md`

```
# vNext

 - [#493](https://github.com/chocolatey/chocolatey/issues/493) - [Enhancement] Chocolatey-Update cleanup
 - [#492](https://github.com/chocolatey/chocolatey/issues/492) - Error messages late in update of chocolatey itself
 - [#487](https://github.com/chocolatey/chocolatey/issues/487) - nuget.exe hangs for packages with many dependencies
 - [#486](https://github.com/chocolatey/chocolatey/pull/486) - Improve Chocolatey setup as administrator and add Test-ProcessAdminRights helper contributed by Jakub Berezanski ([jberezanski](https://github.com/jberezanski))
 - [#424](https://github.com/chocolatey/chocolatey/issues/424) - Update Contributing.md with link to mailing list
 - [#416](https://github.com/chocolatey/chocolatey/pull/416) - [Enhancement] added quiet parameter and forced write-host to honor that param (#411) contributed by Johan Leino ([jole78](https://github.com/jole78))
 - [#411](https://github.com/chocolatey/chocolatey/issues/411) - [Enhancement] absolute "quiet" mode - Allow shutting off "real" Write-Host
 - [#393](https://github.com/chocolatey/chocolatey/pull/393) - Resolve issue with DISM "missing" or with the 32-bit DISM being called on a 64-bit system contributed by Julian Easterling ([dcjulian29](https://github.com/dcjulian29))
 - [#379](https://github.com/chocolatey/chocolatey/issues/379) - [Enhancement] Update NuGet.exe to 2.8+ in Chocolatey install

Commits: [c1ab0a6473...f541d8ca31](https://github.com/chocolatey/chocolatey/compare/c1ab0a6473...f541d8ca31)
```

That was pretty easy, I submitted this as a pull request and Rob can just edit and next time generate the new release notes with easy. Including all the beta's.
You can view the pull request at [https://github.com/chocolatey/chocolatey/pull/496](https://github.com/chocolatey/chocolatey/pull/496) - it may not be merged, but I figured this was a good guide of how you can start using GitReleaseNotes on your own project. 
Feel free to post on issue on GitHub if you have issues with your project, it is still a work in progress tool after all!

You can also run `GitReleaseNotes /o releasenotes.md /allTags` to generate a complete new set of release notes since the start of the project with easy. Give that a go yourself and see what the output is.  