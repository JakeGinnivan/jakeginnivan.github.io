---
layout: post
title: Release NuGet SemVer packages from Teamcity
metaTitle: Release NuGet SemVer packages from Teamcity
description: I am involved in releasing quite a few open source projects and up until now I have not been happy with the release process. 
revised: 2013-09-28
date: 2013-09-28
categories: [NuGet,teamcity,github,releases]
migrated: true
comments: true
sharing: true
footer: true
permalink: /release-nuget-semver-packages-from-teamcity/
summary: | 
  I am involved in releasing quite a few open source projects and up until now I have not been happy with the release process. 

---
I have a number of open source projects and I do not really have a **good** release process. So I spend the arvo trying to figure out a good way to do it.

My goals were

 - Use GitHubs releases feature - [https://github.com/blog/1547-release-your-software](https://github.com/blog/1547-release-your-software)
 - I want to release from NuGet
 - Preferably write release notes before I click the button in TeamCity, this way i can add them on github to build up a release
 - Support SemVer, including pre-release packages
 - Assembly versions should be stamped with informational version as well as a version
 - Be able to link to the project GitHub releases from the NuSpec
<!-- more -->
# My Solution
## 1. Setup the VCS Root to be authenticated
![ReleaseNuGetSemVerpackagesfromTeamcity](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/ReleaseNuGetSemVerpackagesfromTeamcity.png)

Then tell TeamCity to Label Successful builds
![Release-NuGet-SemVer-packages-from-Teamcity11](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity11_635160285120295000.png)

## 2. Setup some additional build parameters
![Release-NuGet-SemVer-packages-from-Teamcity](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity_635160285057326250.png)

### 2.1. AssemblyVersion
This is the assembly version, only change this for major releases, this will save people adding binding redirects when different projects rely on different versions
![Release-NuGet-SemVer-packages-from-Teamcity1](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity1_635160285061232500.png)  
![Release-NuGet-SemVer-packages-from-Teamcity2](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity2_635160285065138750.png)

If you want the text version of the spec, it is

    text description='The assembly version which will be stamped (assembly info version/nuget version will be the build number)' display='prompt' label='AssemblyVersion' validationMode='not_empty'

### 2.2. Prerelease
This is a checkbox, when ticked it's value is `-pre` so we can just use it in the version
![Release-NuGet-SemVer-packages-from-Teamcity3](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity3_635160285069201250.png)  
![Release-NuGet-SemVer-packages-from-Teamcity4](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity4_635160285073263750.png)

Once again the text version of the spec is

    checkbox checkedValue='-pre' description='Check this box if you want a pre-release' display='prompt' label='PreRelease?'
    
### 2.3. Version
This is the version which you will pass to NuGet when you are creating your package
![Release-NuGet-SemVer-packages-from-Teamcity5](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity5_635160285077170000.png)  
![Release-NuGet-SemVer-packages-from-Teamcity6](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity6_635160285081076250.png)  

    text description='This is the version number, adjust the major/minor as needed to conform to semver' display='prompt' label='VersionNumber' validationMode='not_empty'

### 2.4. env.Version
This is just so the version gets set as an environmental variable so my build scripts can pick it up

![Release-NuGet-SemVer-packages-from-Teamcity7](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity7_635160285084513750.png)

I am not sure if this is actually needed.

## 3. Build Steps
I always have a `.proj` file checked in for all of my projects which contain all logic to build the solution, I just have to tell teamcity what targets to invoke. If you want to view my build script, check out [https://github.com/TestStack/ConventionTests/blob/master/ConventionTests.proj](https://github.com/TestStack/ConventionTests/blob/master/ConventionTests.proj)

![Release-NuGet-SemVer-packages-from-Teamcity8](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity8_635160285101388750.png)

The Test target builds and runs my unit tests, then the Publish Target builds the NuGet packages, step 2 of my TeamCity build is Publish to NuGet

![Release-NuGet-SemVer-packages-from-Teamcity9](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity9_635160285112638750.png)

Finally, the assembly info stamping build feature. Click **Add build feature**

We want to use two different versions from our config, remember from above we have 

![Release-NuGet-SemVer-packages-from-Teamcity10](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity10_635160285116545000.png)

## 4. Update the build version
Finally go to the General Settings of your build, and update the build number format to be: `%Version%` and we set the build counter to 0

# Releasing a new version
So now, we have to stick to SemVer, let us run through a scenario of releasing a minor release (new non-breaking feature) of TestStack.ConventionTests. The current release is v2.0.0. I first need to reset the build counter back to 0, if you don't want to mess with the build counter, just change the VersionNumber variable to have a manually updated patch version. This actually is probably better because then the version numbers are more predictable.

## Create release definition in GitHub
![Release-NuGet-SemVer-packages-from-Teamcity16](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity16_635160285172951250.png)
![Release-NuGet-SemVer-packages-from-Teamcity17](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity17_635160285176857500.png)

Specify the release notes for this release
![Release-NuGet-SemVer-packages-from-Teamcity18](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity18_635160285180763750.png)

Do not have the Publish Release box ticked, otherwise GitHub will create the tag for you. 

![Release-NuGet-SemVer-packages-from-Teamcity19](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity19_635160285184670000.png)

## Release from TeamCity
I click *Run* in TeamCity, and I will be prompted to confirm everything

![Release-NuGet-SemVer-packages-from-Teamcity14](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity14_635160285165138750.png)

Because this is a minor release, I increment the minor version number
![Release-NuGet-SemVer-packages-from-Teamcity15](/assets/posts/2013-09-28-release-nuget-semver-packages-from-teamcity/Release-NuGet-SemVer-packages-from-Teamcity15_635160285169045000.png)

Now hit run, this will publish to NuGet and create the tag in Git. The final step is to go back to GitHub and publish the release (this is not done automatically, and I'm sure it could be automated through the GitHub API).


This is my first attempt at making it easy for me to release my open source projects with SemVer and GitHub Releases