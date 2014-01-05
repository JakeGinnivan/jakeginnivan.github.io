---
layout: post
title: Setting up Git
metaTitle: Setting up Git
description: This is how I setup git on windows
revised: 2013-07-27
date: 2013-07-27
categories: [git]
migrated: true
comments: true
sharing: true
footer: true
permalink: /setting-up-git/
summary: | 
  

---
I have setup git many times for myself, and also team members. I thought I would just share the way I install and setup my Git environment on Windows.

I use [Git Extensions](https://code.google.com/p/gitextensions/) as my Gui when I am not using the Command Line (which is my preference). It also is bundled with KDiff3 and MsysGit which means you only have to download one things.
<!-- more -->
## Required Software
Tick both boxes (MsysGit and KDiff3)

## Feature selection
I tend to disable Visual Studio integration. With VS 2013 you get native git support which will continue to get better over time. And I don't need more menus...

## Select SSH Agent
I prefer to go with OpenSSH, it is more work to setup, but once you have generated your ssh keys, put the .ssh folder into your dropbox or back it up, then you can just drop it back into your user profile when you reinstall windows or move to another computer.

# KDiff Installation
Next Next Next Finish etc. :)

# Git Installation
## Select components
I leave this as the default

## Adjust your PATH environment
I go for option 2 (Run git from the Windows Command Prompt), this sets up a reasonable default so you can access git from the command line anywhere

## Line Endings
Option 3, Checkout as is, commit as is. Nobody likes it when you mess with their line endings.

# Install PoshGit
Now you have git installed, open up a PowerShell console as Administrator and change directories to somewhere sensible that you want to put your code.

Then run: 

 - `Set-ExecutionPolicy RemoteSigned`, this will allow you to run powershell scripts
 - `git clone https://github.com/dahlbyk/posh-git`
 - `cd posh-git`
 - `.\install.ps1` - this will install posh-git into your profile.
 - `. $PROFILE` - this will run your profile, now you should see [master] in blue on the command line

You may have noticed a warning when you ran your profile
**WARNING: Could not find ssh-agent**

To fix this, we need to change our PATH settings. 
Run `control sysdm.cpl` on the command line, go to the Advanced tab, click Environmental Variables
The edit your PATH variable. When you click edit, you should see `C:\Program Files (x86)\Git\cmd` right at the end. Change this to `C:\Program Files (x86)\Git\bin`

Now when you restart your powershell console, you shouldn't get a warning.

# Creating your SSH Key
[https://help.github.com/articles/generating-ssh-keys](https://help.github.com/articles/generating-ssh-keys) is a great resource for creating your SSH key, after you are done you should have a folder in your user profile called .ssh, and two files in that folder, id_rsa and id_rsa.pub.

Simply copy the contents of id_rsa.pub to your GitHub keys under your profile, now you can push/pull from github easily!

# Git Config
And finally, here is my .gitconfig

    [core]
	autocrlf = false
	editor = 'C:/Program Files (x86)/Notepad++/notepad++.exe'
    [diff]
        tool = kdiff3
        guitool = kdiff3
    [merge]
        tool = kdiff3
    [mergetool "kdiff3"]
        path = C:/Program Files (x86)/KDiff3/kdiff3.exe
        keepBackup = false
        trustExitCode = false
    [difftool]
        prompt = false
    [mergetool]
        keepBackup = false
    [difftool "kdiff3"]
        path = c:/Program Files (x86)/KDiff3/kdiff3.exe
    [alias]
        st = status
        rc = rebase --continue

Hope that helps someone get started with Git