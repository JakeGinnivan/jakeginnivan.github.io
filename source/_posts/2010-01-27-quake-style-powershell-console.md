---
layout: post
title: Quake Style Powershell Console
metaTitle: Quake Style Powershell Console
description: Press the windows key + tilde (`) to open a powershell console
revised: 2011-02-19
date: 2010-01-27
categories: [powershell]
migrated: true
comments: true
sharing: true
footer: true
permalink: /quake-style-powershell-console/
summary: | 
  

---
I decided today that I wanted a quake style powershell console (with the classic win+` to activate or hide), and here is the end result:
![Awesome powershell console][1]

I used [http://www.instructables.com/id/"Drop-Down",-Quake-style-command-prompt-for-Window/][2] as a template to get it all working, but wanted to use Console2 instead of v1.5 like this article uses.
<!-- more -->
<h1>What you need?</h1>

Download [AutoHotKey][3] <br />
Download [Console2][4]

<h1>Instructions</h1>

 1. Extract the Console2 folder somewhere (I extract to %userprofile%\Console2).
 2. Add the directory to your %path% (http://support.microsoft.com/kb/310519) or create a shortcut to Console.exe and place it in your windows directory.
  - In windows 7/Vista you can use the command 'setx PATH "%PATH%;%userprofile%\Console2\" /M'
 3. Download [console.xml][5] and [QuakeMode.ahk][6] and overwrite the original console.xml. Then edit console.xml and set the startup directory.
 4. Right click on QuakeMode.ahk and run the script.

Now you can press win + ` to open the console, esc or win+` to close again. Also right click on the console to access the settings to modify things like colour, window size etc etc.


  [1]: /get/screenshots/PowershellConsole.png
  [2]: http://www.instructables.com/id/"Drop-Down",-Quake-style-command-prompt-for-Window/
  [3]: http://www.autohotkey.com/download/
  [4]: http://sourceforge.net/projects/console/
  [5]: /get/downloads/console.xml
  [6]: /get/downloads/QuakeMode.ahk