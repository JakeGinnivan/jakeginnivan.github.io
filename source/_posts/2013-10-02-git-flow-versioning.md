---
layout: post
title: Git Flow Versioning
metaTitle: Git Flow Versioning
description: I just tried using git-flow to manage releases, I quite like it.
revised: 2013-10-02
date: 2013-10-02
categories: [gitflow,git,github]
migrated: true
comments: true
sharing: true
footer: true
permalink: /git-flow-versioning/
summary: | 
  I just tried using git-flow to manage releases, I quite like it.

---
As a follow up to my last post [http://jake.ginnivan.net/release-nuget-semver-packages-from-teamcity](http://jake.ginnivan.net/release-nuget-semver-packages-from-teamcity) I have been investigating more into different ways to achieve semantic versioning and being able to release in an easy way.

Next stop on my trip was looking into Git-Flow and how it manages releases, initially it seemed very waterfally and too heavy to use on an open source project, but I thought it may fit at different clients. I was instantly drawn to the fact that releases have an explicit step to version at the start of the release process, this is great, I can manage my project, merge pull requests, add features then when I am ready, I can decide to release, put together release notes and figure out if this is a major, minor or patch release.

Much to my surpise, it is actually very light-weight and you can drop much of it if your process is simpler (say for open source).

[https://github.com/TestStack/ConventionTests](https://github.com/TestStack/ConventionTests) is my guinea pig!

To get started, I decided to try and implement git-flow manually to really understand what is going on. There are plenty of explainations out there, so I will just be running through what I did, and how it works from my point of view. I always have two remotes setup for my projects, 'upstream' is the main repository, 'origin' is my fork.

## 1. Convert the repo over to git-flow
From your git command line

    git checkout master
    git fetch upstream
    git merge upstream/master
    git checkout develop
    git push upstream develop
    
This pushes the `develop` branch into your repo, then head to the project settings in github and change the default branch to `develop`.

## 2. Install GitFlowVersion
I am a big fan on Simon Cropp's ([https://twitter.com/SimonCropp](https://twitter.com/SimonCropp)) work. He has been working on [https://github.com/Particular/GitFlowVersion](https://github.com/Particular/GitFlowVersion) with Andreas Öhlund.

Basically GitFlowVersion uses the conventions in place in git-flow to make it really easy to version your software.

Once up and running, tags off `master` get a normal version, say `v2.1.0` (assume this is the LAST tagged release), develop CI builds get the version `v2.2.0-unstable20` where the minor is LASTMINOR + 1 and there has been 20 commits since the last tag on master. Or `v<major>.<minor+1>.0-unstable<#commitssincelastrelease>`
Release branches have a version of `v2.2.0-beta` where 2.2.0 is the version you have put in the release branch name (in this case `release-2.2.0`)
You can also tag a release branch as `rc1` or `rc2` and the version will become `v2.2.0-<tag>`. 

If this sounds confusing, its not, and it is all pretty automatic, read on to see what it actually means for you maintaining a project.

## 3. Contributing/Pull Requests
I always use feature branches, but not for an entire feature. I use very short lived branches, which I put up as a pull request as soon as I am done. I am known to submit 3+ pull requests all within the space of an hour when I work on a project because I just fix a bunch of small things.

Nothing much changes here, except you take you branch from `develop`. So

    git checkout develop
    git fetch upstream
    git merge upstream/develop
    git checkout -b FixingSomething

Pull requests now target `develop`, so I do my commits, push to origin like normal and submit my pull request targeting `develop`

Your CI build (assuming you are building pull requests) will trigger with build number `v2.2.0-PullRequest50`, where the version is the last release, with the minor bumped, just like develop version numbers, except the semver tag is PullRequest<PR#>. 
I think that is pretty neat.

## 4. Releasing a new version
When you decide you want to release, or start preparing a release you can either take a release branch, or merge develop straight into master and tag master with the release (afaik this doesn't cause any issues :P)

In my mind there are two styles to run a project, 

1. every checkin builds, then if tests pass autodeploys. These projects cannot adhere to semantic versioning, but the changes are often so tiny that upgrades pose little risk.
1. At some point in time, the decision to release is made, this could be as soon as a pull request is merged, or after a particular milestone is reached. All the projects I am currently contributing to work in this way.

### 4.1 Using a release branch
Using a release branch allows you to make the decision I am going to release, there may be a few things I know I have to do before releasing (DbUp is a great example of this). But I don't want to stop being able to merge pull requests. To release using a branch (assume develop is up to date), and I am only fixing a bug in the current v2.1.0 release

    git checkout -b release-2.1.1
    
or

    git flow release start 2.1.1
    
I can now leave this branch open for a bit, do the work I need to do to release 2.1.1 or send it to someone else on the team to OK. I could even release the 2.1.1 build as a pre-release package on NuGet. It would have version 2.1.1-beta1 remember.

Once I am happy, either the pre-release package has got the feedback I wanted, or I have made the additional changes I need I simply merge the branch to master (with the --no-ff option), tag, then delete the remote branchs. Remember this is if you are doing it manually, you can also just go `git flow release finish` if you have the git-flow extensions installed, or use source tree and click a button :P The first two commands are not needed if you haven't committed anything to the release branch

    git checkout develop
    git merge release-2.1.1 --no-ff
    git checkout master
    git merge release-2.1.1 --no-ff
    git tag 2.1.1
    
or

    git flow release finish 2.1.1
    
Then publish

    git push upstream master
    git push --tags
    git branch -d release-2.1.1
    git push upstream :release-2.1.1

or

    git flow release publish 2.1.1

### 4.2 Skip release branch
If you don't want to bother with the release branch, you could also just go

    git checkout master
    git merge develop --no-ff
    git tag 2.1.1
    git push upstream master
    git push --tags

## 5. Publish the build
Now you have the release build from your CI you can progress through your build pipeline and release to NuGet or publish to a test environment or whatever the first step is in your Continuous Delivery pipeline.

## That's it
I think Git-Flow is actually not too bad for releasing projects using semantic versioning, most of the time in this is writing the release notes, seeing what has changed since the last release (which is really easy due to the conventions in place!) and checking to see if you have any breaking changes which would mean a major version bump.

As a summary. Here is me fixing and releasing a feature as a pre-release package. The current version on NuGet is currently v2.2.0

    git fetch upstream
    git checkout develop
    git merge upstream/develop
    git checkout -b SomeFeature
    
    # Work..
    git commit -am "Some experimental feature"
    git push origin SomeFeature
    
    # Submit pull request, so NOTHING has changed yet, this PR is merged
    git fetch upstream
    git checkout develop
    git merge upstream/develop # to get my changes into my local develop branch
    
    # So now I am starting the release, nothing before here is new if you are using GitHub flow
    git checkout -b release-2.3.0
    git push upstream release-2.3.0
    
Now my CI builds, and I click the button to promote the last CI build as a pre-release NuGet package. Once I am happy, I have got my feedback

    git checkout master
    git merge release-2.3.0 --no-ff
    git tag 2.3.0
    git push upstream master
    git push upstream --tags

My CI will build, and I can click the publish button on teamcity to release to NuGet.

Thoughts? Do you know of an easier way. I am REALLY open to suggestions and willing to try stuff out at the moment :)