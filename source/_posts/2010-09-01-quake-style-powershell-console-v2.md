---
layout: post
title: Quake Style Powershell Console v2
metaTitle: Quake Style Powershell Console, v2
description: I revisited this idea, and have improved it a lot =)
revised: 2011-06-09
date: 2010-09-01
categories: [powershell,command-line]
migrated: true
comments: true
sharing: true
footer: true
permalink: /quake-style-powershell-console-v2/
summary: | 
  

---
Edit: Updated to fix a problem when there are spaces in the path where Console2.exe is.

I decided I would get this idea working again, when I did, it fell short a few times. I needed a few extra features:

 - Elevated Powershell Console
 - x64 Console
 - I wanted it to work without adding anything to the %path% environmental variable

##The goal

win + `

= 

![Quake Mode Powershell][1]
<!-- more -->
##What you need

 - [Console2 x64][2]
 - [Elevate.exe, c version (Blog Post)][3]
 - or [Elevate.exe, c# version (Blog Post)][4]
 - [AutoHotKey script, Console.xml config file, and launch.cmd][5]
 - [AutoHotKey][6]


##Instructions
 1. Download and install AutoHotKey (while you are at it, grab the autocorrect script [here][7]
 2. Download and extract Console2 x64 to a folder. 
 3. Extract my script file into the same folder
 4. Make sure all files are unblocked (right click, properties, unblock if they are).
 5. Create a shortcut to QuakeMode.ahk in the Startup folder in the start menu

##How to use

Press the windows key + ` (tilde, above tab) to activate the console.
You will see a few command windows run, and you will be prompted to put in your password (or press yes) to elevate the process. If you have UAC disabled, turn it back on :P


  [1]: /get/screenshots/powershellConsole.png
  [2]: http://sourceforge.net/projects/console/files/
  [3]: http://jpassing.com/2007/12/08/launch-elevated-processes-from-the-command-line/
  [4]: http://www.wintellect.com/cs/blogs/jrobbins/archive/2007/03/27/elevate-a-process-at-the-command-line-in-vista.aspx
  [5]: get/downloads/quakeconsolescripts.zip
  [6]: http://www.autohotkey.com/download/
  [7]: http://www.autohotkey.com/docs/Hotstrings.htm#AutoCorrect