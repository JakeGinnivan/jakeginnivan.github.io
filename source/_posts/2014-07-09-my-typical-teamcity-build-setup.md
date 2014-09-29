---
layout: post
published: true
title: My Typical TeamCity build Setup
comments: true
---

I posted [Simple Versioning and Release Notes](http://jake.ginnivan.net/blog/2014/05/25/simple-versioning-and-release-notes/) a few weeks ago talking about how to simplify release notes and versioning. This post is a bit of a cheat sheet for how I set up my builds.

## Structure
I normally have two but sometimes three builds for each project. The structure is something like this:

 - TeamCity Project
   1. CI
   2. Acceptance/UI Tests (optional)
   3. Release

`1. CI` builds the solution with correctly versioned assemblies (update assembly info files with version before build), runs all unit tests then creates any packages which are required. This includes NuGet packages, Chocolatey packages, zipped binaries, clickonce installers etc.
This build monitors pull requests and is triggered automatically when there are new commits/branches.

`2. Acceptance/UI Tests` is an extra I use when I have long running or UI tests, for example [TestStack.White on TeamCity (sign in as guest)](http://teamcity.ginnivan.net/project.html?projectId=TestStack_White&tab=projectOverview) has this build setup and it runs Whites UI tests.
The reason I separate it is for speed reasons, I want my CI build to fail fast and only if it is successful do I run this slow build.
This build triggers whenever `1. CI` succeeds

`3. Release` (or 2. Release if there is no acceptance/ui test build) is run manually and it releases the artifacts build by `1. CI`, if that is a NuGet package it is pushed to NuGet.org, chocolatey packages get pushed to chocolatey.org, zip files get pushed as a GitHub release etc.
Once this build succeeds it should tag the VCS root and push that tag.

<!-- more -->

## Build Configuration
### VCS Root
First step is to create our VCS root.  
**Type: ** Git  
**VCS Root Name: ** Project Name (all branches + pull requests)  
**Default branch: ** master  
**Branch specifications: **
```
+:refs/pull/*/merge
+:refs/heads/*
```
If you are using stash it would be
```
+:refs/pull-requests/*/merge-clean
+:refs/heads/*
```
You should also put a username/password in to authenticate with the VCS root so we can tag in the release build

### 1. CI
This has two build steps, for this to work you should install GitVersion on your build server via chocolatey.

#### Build steps
 - **GitVersion**  
**Runner Type:** Command Line  
**Step Name:** GitVersion  
**Run:** Executable with parameters  
**Command executable:** GitVersion  
**Command parameters:** . /updateAssemblyInfo /output buildserver

 - **Build project**  
This should be setup however you used to do it, build your solution, run your build scripts etc.

#### Triggers
Add trigger `VCS Trigger` with the default settings which has a branch spec of `+:*`

#### Build Features
I have the report status to GitHub plugin installed, this plugin tells github the build status of pull requests so it can display warnings if a pull request will break the build

### 2. Acceptance test
This build varies greatly, if you need this build it has the same triggers and dependencies setup as 3. Release. I won't repeat those settings here.

### 3. Release
#### VCS Root
The same VCS root which is referenced by 1. CI should also be referenced by this build
#### Build steps
Normally for me this just a NuGet push step. But whatever you normally do, use the artifact dependencies to get at the artifacts from your CI build to publish
#### Build Features
Add feature `VCS Labelling`  
**Labeling pattern:** %system.build.number%  
**Label builds in branches:** `+:*` (we want to label any release)  
Check *Label successful builds only*
#### Dependencies
This is the important part, we want to setup a proper TeamCity build chain.
1. Add snapshot dependency for CI build and previous build in chain, so if you have 2. Acceptance tests then add a snapshot dependency for both CI and Acceptance test builds  
Tick *Do not run new build if there is a suitable one* and *Only use successful builds from suitable ones*  
The snapshot dependency is really important, it means that you release the artifacts associated with a specific git commit. If you don't you can deploy artifacts which came from another branch =/
2. Add Artifact Dependency to CI Build with these settings:  
**Get artifacts from: ** Build from the same chain  
**Artifacts rules: ** MyPackage.\*.nupkg (or whatever artifacts you need to publish
#### General Settings
I put this last because you need the dependencies setup before configuring this page. 
**Build Number Format:** %dep.MyProject_Ci.build.number% (MyProject_Ci is the project ID of the CI build)

## Wrap up
Hopefully this helps you get your TeamCity builds setup, I have found this setup works quite well and is easy to setup and keep running.