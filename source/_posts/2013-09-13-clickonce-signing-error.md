---
layout: post
title: ClickOnce Signing Error
metaTitle: ClickOnce Signing Error
description: I had an error signing clickonce installers, thought I would blog about it
revised: 2013-09-13
date: 2013-09-13
categories: [ClickOnce]
migrated: true
comments: true
sharing: true
footer: true
permalink: /clickonce-signing-error/
summary: | 
  I had an error signing clickonce installers, thought I would blog about it

---
On my current project we use ClickOnce, and I am setting up the build server to sign using a proper cert rather than a self generated one.

The command: 

    mage.exe -New Application -ToFile <path>\App.exe.manifest -name "<Name>" -Version 0.1.1.1 -FromDirectory <path>\0.1.1.1\ -IconFile App.ico -CertHash "‏ca5da5a1f7c57411111111a79cbf50c4432ed949"

And was getting `This certificate cannot be used for signing - "ca5da5a1f7c57411111111a79cbf50c4432ed949"`

I was searching for what extended attributes are required, checking if I got the right thumbprint and wasted a bunch of time.

The fix was simple, **remove the quotes** around the thumbprint