---
layout: post
title: ReSharper Xaml Attribute Ordering Plug-in
metaTitle: ReSharper Xaml Attribute Ordering Plug-in
description: This is something I have wanted for a while, so I started this plugin over the weekend at JetBrains day
revised: 2013-09-09
date: 2013-09-09
categories: [ReSharper, Open Source]
migrated: true
comments: true
sharing: true
footer: true
permalink: /resharper-xaml-attribute-ordering-plugin/
summary: | 
  

---
A while back I got introduced to [https://xamlstyler.codeplex.com/][1] which is a pretty good visual studio plug-in for formatting xaml, but I like ReSharper's formatting options better :)

So over the weekend I was at [JetBrains Day][2] in Malmo, and Matt Ellis did a talk on ReSharper extensions. I figured it would be a good time to try and write a plug-in. 

Here are some screenshots

![Moar Xaml Code Cleanups](/assets/posts/2013-09-09-resharper-xaml-attribute-ordering-plugin/Capture1.PNG)

![Options](/assets/posts/2013-09-09-resharper-xaml-attribute-ordering-plugin/Capture2.PNG)

Now, I don't expect this to be super stable and I know of a few issues (like when you first format the Window tag is not quite formatted right), but I hope to setup a CI build and get some fixes out over the next week or so.

Check out the code, report issues and submit pull requests at [https://github.com/JakeGinnivan/XamlAttributeOrderingCodeCleanup][5] and install from [ReSharper Extensions Site][6]


  [1]: https://xamlstyler.codeplex.com/
  [2]: http://www.jetbrains.com/jetbrainsday/
  [5]: https://github.com/JakeGinnivan/XamlAttributeOrderingCodeCleanup
  [6]: https://resharper-plugins.jetbrains.com/packages/JetBrains.ReSharper.Plugins.XamlAttributeOrdering/