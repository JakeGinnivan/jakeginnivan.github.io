---
layout: post
title: Phone Test Project Template
metaTitle: Phone Test Project Template
description: I have released a phone test project to the visual studio gallery allowing you to File -> New Windows Phone Test Project
revised: 2011-08-21
date: 2011-08-20
categories: [Open Source, Windows Phone, Testing]
migrated: true
comments: true
sharing: true
footer: true
permalink: /phone-test-project/
summary: | 
  I have released a phone test project to the visual studio gallery allowing you to File -> New Windows Phone Test Project

---
Last week I put together a project template for a windows phone 7 test project. At the moment there isn't a really good story for unit testing on the phone. If you have the mango tools you have to grab the Mango Silverlight Unit Test assemblies from Jeff Wincox's blog. 

The Project Template available at [http://visualstudiogallery.msdn.microsoft.com/6819514d-4bd6-4f31-a231-48c6530ed03b](http://visualstudiogallery.msdn.microsoft.com/6819514d-4bd6-4f31-a231-48c6530ed03b) is really basic, and you then have to add a reference via NuGet of either the Silverlight Unit Testing Framework (doesn't work with Mango), or WindowsPhoneEssentials.Testing.

The advantage of the WindowsPhoneEssentials.Testing project is it contains the Mango compatible versions, sets everything up and has a collection of really useful testing related helpers/abstractions for WP7. Check out [http://windowsphonefoundations.net/windowsphoneessentials](http://windowsphonefoundations.net/windowsphoneessentials) or the source at [http://wp7essentials.codeplex.com/](http://wp7essentials.codeplex.com/) for more information.

But what you get after you install it is:

![New Phone Test Project](/assets/posts/2011-08-20-phone-test-project/NewTestProject.png)

Then you add the NuGet reference to WindowsPhoneEssentials.Testing

![Add NuGet reference to WindowsPhoneEssentials.Testing](/assets/posts/2011-08-20-phone-test-project/TestingNuGet.png)

Enjoy!