---
layout: post
published: false
title: "GitVersion: GitFlow version numbers"
comments: true
---

So it turns out this is quite a complex topic with different people having different goals. In GitVersion we wanted to come up with a strategy which solves most of the normal things people want to do in a sensible way and guide people in a direction which should *just work*.

This blog post covers GitFlow, I will follow up with a GitHubFlow version which has it's own set of issues.

This is a list of things people normally want to do when using GitFlow:

 - Have a CI build feed off develop which can be consumed to acheive continuous integration across packages
 - Be able to hotfix old versions, and create pre-release of that hotfix
 - Be able to create releases, and have beta's/rc's of that release
 - Release stable packages

So given those basic requirements, lets see how hard it can be. {major}, {minor} and {patch} are the parts of the *last tag on master*

If you are unfamiliar with GitVersion, it is an exe which calculates your applications semantic version from your git branches! It will expose multiple variables in different formats, but a complete SemVer will be `{major}.{minor}.{patch}-{tag}+{buildmetadata}`

<!-- more -->

## develop branch
The develop branch will be versioned as {major}.{minor+1}.0 automatically because it is at least a minor version bump, but may be a major version bump. GitFlow uses hotfix branches for patch releases.

Our first requirement was to allow a CI feed off develop, this means SemVer does not fully work for develop because SemVer requires only a single increment for each release. This means develop must always sort higher than all other packages of the same semantic version (including hotfixes, releases and features).

The way we solve this is to use the `unstable` tag and promote the build metadata build number to the 4th digit of the version. So if the calculated SemVer is `1.2.0-unstable+10` the version produced by GitVersion will be `1.2.0.10-unstable`.

### But how do I test my releases
The recommended approach is to use a separate MyGet feed or the Build Server feed to expose your CI packages.
This means consumers can opt into the firehost of packages being produced by CI which can have a cleanup policy. If you would rather consume beta/rc/stable packages then they should be pushed to the official feeds (nuget.org or similar).

## Hotfix branch
The hotfix branch only works for the last release off master, if you need to hotfix a previous release check out [Support branch](#support-branch)

To hotfix a release, checkout a hotfix branch from that releases tag (say 1.0.0) with the naming convention of `hotfix/{major}.{minor}.{patch+1}` (hotfix/1.0.1). Git version will then calculate the version of that branch is `{major}.{minor}.{patch}-beta.1` (1.0.1-beta.1). This allows you to have beta packages of your hotfix. When you release beta.1 you tag the hotfix branch and the calculated version will be bumped to beta.2.

When you are ready to release your hotfix, merge the hotfix branch into master. The stable version will be produced. We can push this package to NuGet or whatever and then tag master with the release version.
Once that is done we delete the hotfix branch.

## Support branch
When we need to support old versions we use support branches. For this example lets say we have tagged master with 1.0.0 and 2.0.0, now we need to hotfix 1.0.0.

First we take a branch off the 1.0.0 tag called `support/1.0.0`. This will build as 1.0.1 automatically.

We then branch off our support branch to a hotfix branch using the same conventions above. So in this case we branch from `support/1.0.0` -> `hotfix/1.0.1`. This will build us `1.0.1-beta.1` just like the example above.

Support branches are not deleted for as long as you need to support that release. Treat support branches like master, and are versioned as such. When you merge your hotfix branch into the support branch the stable version will start being built.

### What about supporting minor releases
One of the points of Semantic Versioning is allowing safe upgrades between minor versions, this means that you should only need a support branch for each *major* version because the hotfix can target the last released minor version for that major.

## Feature branch
Feature branches are versioned like develop. Except the branch name will be used instead of `unstable`. This means if you name your feature branch `feature/xMyFeature` the tag will be `-xMyFeature` which will sort **higher** than develop. This gives you the option to force a feature branch to be sorted higher than develop if you need to.

## NuGet
NuGet does not support semantic versioning properly, which sucks. We have the `LegacySemVer` and `LegacyPaddedSemVer` variables mainly for NuGet, but we did not tie the name to NuGet as there may be other systems which do not support SemVer properly.
Use the padded variable if you are liklely to need more than 10 betas/pre-releases because in nuget 1.0.0-beta2 sorts higher than 1.0.0-beta10 =(

## Summary
Hopefully this gives you some idea how we envisioned GitVersion to be used with GitFlow. Feedback is welcome.
