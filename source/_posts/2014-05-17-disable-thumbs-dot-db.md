---
layout: post
title: "Disable thumbs.db"
date: 2014-05-17 11:43:14 +0100
comments: true
categories: 
---
Thumbs.db is created by windows whenever a folder has images in it as a cache for the thumbnails for the images. But it gets locked and stops me switching branches in git and I have to restart my PC sometimes.

Save this as `KillThumbsDb.reg` and run it as administrator:

    Windows Registry Editor Version 5.00

    [HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Explorer]
    "DisableThumbsDBOnNetworkFolders"=dword:00000001
    
I have no idea why thumbs.db is being created in local folders, but this fixes it for me.