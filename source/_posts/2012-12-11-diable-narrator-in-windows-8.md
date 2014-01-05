---
layout: post
title: Disable Narrator in Windows 8
metaTitle: Disable Narrator in Windows 8
description: Use ReSharper and accidentally hit win+enter rather than alt+enter and Narrator pops up? This is how you disable it
revised: 2012-12-12
date: 2012-12-11
categories: []
migrated: true
comments: true
sharing: true
footer: true
permalink: /diable-narrator-in-windows-8/
summary: | 
  Use resharper and accidently hit win+enter rather than alt+enter and Narrator pops up? This is how you disable it

---
I get rather annoyed with the narrator. Using [this](http://blog.ostebaronen.dk/2012/08/disable-narrator-in-windows-8.html) as inspiration here is how you do it.

**Update:**  Another solution is available at [http://www.hmemcpy.com/blog/2012/12/how-to-disable-windows-narrator-appearing-on-win-enter-in-windows-8/](http://www.hmemcpy.com/blog/2012/12/how-to-disable-windows-narrator-appearing-on-win-enter-in-windows-8/) which uses Image File Execution Options instead.
<!-- more -->
Note: You can just delete it, but windows will restore it every time an update is installed

 - Open `C:\Windows\System32`
 - Open Properties of Narrator.exe, goto Security Tab, Click advanced  
![Narrator 1](/assets/posts/2012-12-11-diable-narrator-in-windows-8/Narrator1.png)
 - Change the owner to yourself  
![Narrator 2](/assets/posts/2012-12-11-diable-narrator-in-windows-8/Narrator2.png)
 - Click ok, and accept the warning
 - Now back on the properties, click edit  
![Narrator 3](/assets/posts/2012-12-11-diable-narrator-in-windows-8/Narrator3.png)
 - Add yourself, then explicitly deny `Read & execute` and `Read`. Click yes on warning about modifying security on system files  
![Narrator 4](/assets/posts/2012-12-11-diable-narrator-in-windows-8/Narrator4.png)
 - Change owner to `SYSTEM`  
![Narrator 5](/assets/posts/2012-12-11-diable-narrator-in-windows-8/Narrator2.png)

Win+Enter now won't do anything :D


  [1]: http://www.hmemcpy.com/blog/2012/12/how-to-disable-windows-narrator-appearing-on-win-enter-in-windows-8/