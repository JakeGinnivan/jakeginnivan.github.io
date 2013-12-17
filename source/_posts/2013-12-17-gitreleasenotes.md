---
layout: post
title: GitReleaseNotes Initial Release!
metaTitle: GitReleaseNotes Initial Release!
description: Need release notes for your GitHub project? It is easy with GitReleaseNotes
revised: 2013-12-17
date: 2013-12-17
categories: [git,github,jira]
migrated: true
comments: true
sharing: true
footer: true
permalink: /gitreleasenotes/
summary: | 
  Need release notes for your GitHub project? It is easy with GitReleaseNotes

---
I have just used GitReleaseNotes to publish a release of GitReleaseNotes on GitHub [https://github.com/JakeGinnivan/GitReleaseNotes/releases/tag/0.2.0](https://github.com/JakeGinnivan/GitReleaseNotes/releases/tag/0.2.0)!

I am excited about this project because it will save me a heap of time and effort with managing the open source projects I do releases for.

GitReleaseNotes was another project I kicked off about the same time that [Simon Cropp](https://github.com/simoncropp) and [Andreas Öhlund](https://github.com/andreasohlund) were kicking off similar projects, see [https://github.com/Particular/ReleaseNotesCompiler](https://github.com/Particular/ReleaseNotesCompiler). We decided to start the projects off down different roads to start with, then maybe merge later once we both could experiment with ideas.

## What does it do
The concept is quite simple, you can optionally specify a Git tag (it will select the newest tag if you do not specify one). It will then scan all commits for references to issues.

Once it has all the referenced commits, it will connect to either GitHub or Jira (and soon YouTrack, TFS and BitBucket) fetching all the closed issues which have been referenced.
It will then output your release notes in [Semantic Release Notes](http://www.semanticreleasenotes.org/) format (which is also markdown) to a file you specify.

It can also publish a release on GitHub, including all the closed issues since your last release.

Here are some examples of the types of release notes it can generate:

[Simple single issue release](https://github.com/JakeGinnivan/GitReleaseNotes/blob/master/src/GitReleaseNotes.Tests/ReleaseNotesGeneratorTests.ApproveSimpleTests.approved.txt]  
[Multiple releases](https://github.com/JakeGinnivan/GitReleaseNotes/blob/master/src/GitReleaseNotes.Tests/ReleaseNotesGeneratorTests.MultipleReleases.approved.txt)

## How to use it
GitReleaseNotes is a .NET exe, but can be used on ALL project types (JavaScript, Ruby, Java, whatever) because it simply works with Git and whatever issue tracker you use.

    GitReleaseNotes.exe /IssueTracker Github /Repo JakeGinnivan/GitReleaseNotes /Token ######################### /OutputFile ReleaseNotes.md

To Publish, just add the `/Publish` switch and specify the version you want to publish with the `/Version` switch. Like so

    GitReleaseNotes.exe /IssueTracker Github /Repo JakeGinnivan/GitReleaseNotes /Token ######################### /Publish /Version 0.2.0

## How to setup on TeamCity
TeamCity is not required, but I like being able to press a button and my latest CI build gets published. 

Well, I am using [GitHubFlowVersion](https://github.com/JakeGinnivan/GitHubFlowVersion) which helps me do Semantic Versioning with ease, but if you have another versioning strategy most of the instructions are the same.

Assuming you have a CI build which creates the artifacts, or you have an existing build you will be adding to.

1. Head to [https://github.com/settings/applications](https://github.com/settings/applications) and create an access token
1. Create a new configuration parameter in teamcity called `GitHubToken`, set the value to the access token you have just created and edit the spec field and paste in `password display='hidden'`, which will mean your GitHub access token will not be published into any build logs and will just be ####ed out.
1. Your VCS root must be set to checkout `Automatically on Agent`, otherwise GitReleaseNotes will not run
1. Make sure you check GitReleaseNotes.exe into source control so you can access it
1. Create a new build step which is running a command line application, and make it look something like this  
    `GitReleaseNotes\GitReleaseNotes.exe /IssueTracker GitHub /Publish /Token %GitHubToken% /Repo JakeGinnivan/GitReleaseNotes /Version %dep.OpenSourceProjects_GitReleaseNotes_CI.system.GitHubFlowVersion.SemVer%`

My version number is the SemVer of my CI build, if you are using GitHubFlowVersion you will have to create a dummy system.GitHubFlowVersion.SemVer variable, otherwise you cannot reference it across builds (doesn't exist at configure time, it is created when you run GitHubFlowVersion).

And thats it, you can have a build publishing your release notes to GitHub and this will also tag master with the version you have just published.

There are still plenty of issues with this project and heaps of work to do, but I like the way it is shaping up. Feedback/contributions are welcome.
