---
layout: post
title: Disable Narrator in Windows 8
metaTitle: Disable Narrator in Windows 8
description: Use resharper and accidently hit win+enter rather than alt+enter and Narrator pops up? This is how you disable it
revised: 2012-12-11
date: 2012-12-11
categories: [windows8]
migrated: true
comments: true
sharing: true
footer: true
permalink: /diable-narrator-in-windows-8/
summary: | 
  Use resharper and accidently hit win+enter rather than alt+enter and Narrator pops up? This is how you disable it

---
I get rather annoyed with the narrator. Using [this](http://blog.ostebaronen.dk/2012/08/disable-narrator-in-windows-8.html) as inspiration here is how you do it

 - Open `C:\Windows\System32`
 - Open Properties of Narrator.exe, Click advanced  
![Narrator 1](/assets/posts/2012-12-11-diable-narrator-in-windows-8/Narrator1.png)
 - Change the owner to yourself  
![Narrator 2](/assets/posts/2012-12-11-diable-narrator-in-windows-8/Narrator2.png)
 - Click ok, and accept the warning
 - Now back on the properties, click edit  
![Narrator 3](/assets/posts/2012-12-11-diable-narrator-in-windows-8/Narrator3.png)
 - Add yourself, then explicitly deny `Read & execute` and `Read`. Click yes on warning about modifying security on system files  
![Narrator 4](/assets/posts/2012-12-11-diable-narrator-in-windows-8/Narrator4.png)
 - Change owner to `SYSTEM`

Win+Enter now won't do anything :D