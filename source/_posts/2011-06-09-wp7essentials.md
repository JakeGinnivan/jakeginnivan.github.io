---
layout: post
title: WP7 Essentials Released!
metaTitle: WP7 Essentials Released!
description: While at MobileCampOz, Brendan Kowitz and I were discussing the need for a essentials library for WP7, this library is focused on code issues, not UI
revised: 2011-06-10
date: 2011-06-09
categories: [open-source,wp7,wp7essentials]
migrated: true
comments: true
sharing: true
footer: true
permalink: /wp7essentials/
summary: | 
  While at MobileCampOz, Brendan Kowitz and I were discussing the need for a essentials library for WP7, this library is focused on code issues, not UI

---
Windows Phone Essentials tries to fill a gap that many of the other WP7 frameworks don't fill. Most of the libraries out there are Controls and UI related libraries, wp7essentials tries to fill that gap.

# Background
[Brendan Kowitz][1] has been writing [Windows Phone MVP][2] and I have started on [Windows Phone MVC][3] which was inspired from [Columbus][4], but the changes I was planning to make to Columbus fundamentally changed the way it worked.
At Mobile Camp Oz, Brendan and I discussed many of the problems we are trying to solve with Windows Phone MVC and MVP. So we have taken the best approaches from each of the frameworks, put them into a common essentials library (which both frameworks rely on) reducing the amount of extra work we both have to do, make the API's for both frameworks similar. Now the decision is simply do I prefer to work using the MVC pattern, or the MVP pattern!

# Windows Phone Essentials
One of the primary goals for wp7essentials is to make phone apps testable, so we have two packages, the first is the `WindowsPhoneEssentials` package, the second is `WindowsPhoneEssentials.Testing` which has additional helpers for writing unit tests against WP7 apps.

# Get started
Either look for `WindowsPhoneEssentials` on NuGet, or head to [http://wp7essentials.codeplex.com/documentation][5] to check out the documentation. 

Please give feedback, we are keen to hear what you would want out of a library like this.


  [1]: http://www.kowitz.net/
  [2]: http://windowsphonemvp.codeplex.com/
  [3]: http://windowsphonemvc.codeplex.com/
  [4]: http://columbus.codeplex.com/
  [5]: http://wp7essentials.codeplex.com/documentation