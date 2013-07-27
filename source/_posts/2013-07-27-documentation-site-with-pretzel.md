---
layout: post
title: Documentation Site with Pretzel
metaTitle: Documentation Site with Pretzel
description: How you can get a documentation site up and running really fast with Pretzel and Azure Websites
revised: 2013-07-27
date: 2013-07-27
categories: [open-source,pretzel]
migrated: true
comments: true
sharing: true
footer: true
permalink: /documentation-site-with-pretzel/
summary: | 
  How you can get a documentation site up and running really fast with Pretzel and Azure Websites

---
Over the last day or so, I have been setting up the documentation website for TestStack at [http://teststack.azurewebsites.net/](http://teststack.azurewebsites.net/)

With Pretzel's Wiki template support, it is super easy to get your own site setup.

## Create the site in pretzel
This assumes you already have pretzel and it is in your PATH.

    C:\PretzelDemo>pretzel.exe create -t=Razor --wiki --azure

There are a few things we are saying:
 - Create me a pretzel site
 - We are not specifying a directory, so pretzel will create the site in the current directory
 - We want our templating engine to be Razor, by default pretzel uses the Liquid templating engine (same as Jekyll)
 - The *--wiki* switch says we want a wiki template rather than the default blog setup
 - The *--azure* switch tells pretzel to create a solution which will bake our site when it is pushed to azure websites, this will move our site into a folder called _source

	C:\PretzelDemo> pretzel.exe create -t=razor --wiki --azure
	starting pretzel...
	create - configure a new site
	Using razor Engine
	Pretzel site template has been created
	Shim project added to allow deployment to azure websites
	Press any key to continue...

A file you may be interested in is `_source\_layouts\layout.cshtml`, this file has all the logic for your Wiki, feel free to edit this, make improvements, style changes, structural changes. What pretzel gives you is just a starting point!

## The Wiki
To view your wiki, run 
`C:\PretzelDemo> pretzel taste _source`

This will fire up pretzel's web server, and launch your wiki.

![NewDocument](/assets/posts/2013-07-27-documentation-site-with-pretzel/NewDocument.png)

Lets create a few files

![NewDocument1](/assets/posts/2013-07-27-documentation-site-with-pretzel/NewDocument1.png)

Notice the items circled in blue are new files, each file we create must have what is called yaml front matter (circled in red).

Yaml front matter contains metadata about the file, like the title, permalink (a fixed url) and order among other things.

If you refresh your browser, you will get this

![NewDocument2](/assets/posts/2013-07-27-documentation-site-with-pretzel/NewDocument2.png)

Pretzel has detected changes to files in the site, then rebuilt our site for us.

## Deploying to Azure Wesbites
Create the website
![NewDocument3](/assets/posts/2013-07-27-documentation-site-with-pretzel/NewDocument3.png)
![NewDocument4](/assets/posts/2013-07-27-documentation-site-with-pretzel/NewDocument4.png)
![NewDocument5](/assets/posts/2013-07-27-documentation-site-with-pretzel/NewDocument5.png)

Now our site and deployment is setup, lets deploy. First copy the git deployment url
![NewDocument6](/assets/posts/2013-07-27-documentation-site-with-pretzel/NewDocument6.png)

Now we head back to our powershell console and run the following commands:

    git init
    "_site" | Out-File .gitignore
    git add -A
    git commit -m "Committed wiki"
    git remote add azure <deploymenturl>
    git push azure master

And that should deploy our site, if everything went well, you should have an output looking something like this

    C:\PretzelDemo [master]> git push azure master
    Counting objects: 33, done.
    Delta compression using up to 4 threads.
    Compressing objects: 100% (29/29), done.
    Writing objects: 100% (33/33), 716.65 KiB | 0 bytes/s, done.
    Total 33 (delta 10), reused 0 (delta 0)
    remote: Updating branch 'master'.
    remote: Updating submodules.
    remote: Preparing deployment for commit id '6476560e7e'.
    remote: Generating deployment script.
    remote: .
    remote: info:    Executing command site deploymentscript
    remote: info:    Solution file path: .\Shim.sln
    remote: info:    The site directory path: .\_source\_site
    remote: info:    Generating deployment script for .NET Web Site
    remote: info:    Generated deployment script files
    remote: info:    site deploymentscript command OK
    remote: Running deployment command...
    remote: Handling .NET Web Site deployment.
    remote: C:\DWASFiles\Sites\pretzeldemo\VirtualDirectory0\site\repository\Shim.sln.metaproj : warning MSB4121: The project configuration for project "Sham" was not specified in the solution file for the solution configuration "Release|Any CPU". [C:\DWASFiles\Sites\pretzeldemo\VirtualDirectory0\site\repository\Shim.sln]
    remote:   Shim -> C:\DWASFiles\Sites\pretzeldemo\VirtualDirectory0\site\repository\bin\Release\Shim.dll
    remote:   starting pretzel...
    remote:   bake - transforming content into a website
    remote:   Recommended engine for directory: 'razor'
    remote:   done - took 2943ms
    remote:   Press any key to continue...
    remote: KuduSync.NET from: 'C:\DWASFiles\Sites\pretzeldemo\VirtualDirectory0\site\repository\_source\_site' to: 'C:\DWAS Files\Sites\pretzeldemo\VirtualDirectory0\site\wwwroot'
    remote: Deleting file: 'hostingstart.html'
    remote: Copying file: 'index.html'
    remote: Copying file: 'MainTopic.html'
    remote: Copying file: 'Topic.html'
    remote: Copying file: 'topic2.html'
    remote: Copying file: 'css\default.css'
    remote: Copying file: 'css\style.css'
    remote: Copying file: 'Feature\index - Copy.html'
    remote: Copying file: 'Feature\index.html'
    remote: Copying file: 'Feature\Topic - Copy.html'
    remote: Copying file: 'Feature\Topic 2.html'
    remote: Copying file: 'Feature\Topic.html'
    remote: Copying file: 'img\favicon.ico'
    remote: Finished successfully.
    remote: Deployment successful.
    To https://JakeGinnivan@pretzeldemo.scm.azurewebsites.net/pretzeldemo.git
     * [new branch]      master -> master

Check the azure management console
![NewDocument8](/assets/posts/2013-07-27-documentation-site-with-pretzel/NewDocument8.png)

Head to your website
![NewDocument7](/assets/posts/2013-07-27-documentation-site-with-pretzel/NewDocument7.png)

Job done! You have a new wiki for documentation.