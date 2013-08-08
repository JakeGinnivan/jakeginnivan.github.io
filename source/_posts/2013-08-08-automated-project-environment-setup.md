---
layout: post
title: Automated Project Environment Setup
metaTitle: Automated Project Environment Setup
description: Create a SetupEnvironment.ps1 for your project now, it will save you time in the long run
revised: 2013-08-08
date: 2013-08-08
categories: []
migrated: true
comments: true
sharing: true
footer: true
permalink: /automated-project-environment-setup/
summary: | 
  Create a SetupEnvironment.ps1 for your project now, it will save you time in the long run

---
## Web Project Dev Environment
A common problem projects have is when a new dev joins the team is they hopefully have to follow a bunch of instructions to set everything up. Most of the time, those instructions are out of date so another member of the team ends up setting everything up.

Depending on the complexity of the project, this can be quite time consuming.

The last project I was on was greenfields, so from day 1 we had a `Setup Dev Environment.ps1` file in the root of the project. This powershell script did the following:

 - Installed IIS, and all the components we needed (windows auth, .net etc)
 - Registered asp.net with IIS
 - Created Websites with host headers for each of the sites in the project (we ended up having 3 different websites)
 - Opened the HOSTS file in notepad (elevated) and printed out the lines you needed to paste into your HOSTS file

Over time, this script gained more and more features. But when a new team member joined the team, they just ran this script which installed/configured everything.

Why IIS you may ask, instead of say IIS Express. Well this approach means each site had it's own domain, so fiddler works out of the box. It also means that our UI Test suite doesn't have to fire up IIS Express to run it's tests.

### The script
This is what our script looked like

	& DISM /Online /Enable-Feature /All `
	/FeatureName:IIS-ApplicationDevelopment `
	/FeatureName:IIS-ASPNET `
	/FeatureName:IIS-BasicAuthentication `
	/FeatureName:IIS-CommonHttpFeatures `
	/FeatureName:IIS-DefaultDocument `
	/FeatureName:IIS-DirectoryBrowsing `
	/FeatureName:IIS-HttpErrors `
	/FeatureName:IIS-HttpLogging `
	/FeatureName:IIS-HttpRedirect `
	/FeatureName:IIS-HttpTracing `
	/FeatureName:IIS-ISAPIFilter `
	/FeatureName:IIS-ISAPIExtensions `
	/FeatureName:IIS-IIS6ManagementCompatibility `
	/FeatureName:IIS-ManagementConsole `
	/FeatureName:IIS-ManagementScriptingTools `
	/FeatureName:IIS-Metabase `
	/FeatureName:IIS-NetFxExtensibility `
	/FeatureName:IIS-ASPNET45 `
	/FeatureName:IIS-NetFxExtensibility45 `
	/FeatureName:NetFx4Extended-ASPNET45 `
	/FeatureName:IIS-Security `
	/FeatureName:IIS-ServerSideIncludes `
	/FeatureName:IIS-StaticContent `
	/FeatureName:IIS-WebServer `
	/FeatureName:IIS-WebServerManagementTools `
	/FeatureName:IIS-WebServerRole `
	/FeatureName:IIS-WindowsAuthentication `
	/FeatureName:IIS-WMICompatibility `
	/FeatureName:WAS-ConfigurationAPI `
	/FeatureName:WAS-NetFxEnvironment `
	/FeatureName:WAS-ProcessModel `
	/FeatureName:WAS-WindowsActivationService
	
	& C:\Windows\Microsoft.NET\Framework\v4.0.30319\aspnet_regiis.exe -i
	
	$invocation = (Get-Variable MyInvocation).Value
	$directorypath = Split-Path $invocation.MyCommand.Path
	$physicalPath = Join-Path $directorypath "src\SampleWebSite"
	$adminWebPhysicalPath = Join-Path $directorypath "src\Project.AdminWeb"
	$guestServicesPhysicalPath = Join-Path $directorypath "src\Project.GuestServices"
	$fakeApiPhysicalPath = Join-Path $directorypath "src\FakeSiteApi"
	$elevate = Join-Path $directorypath "tools\Elevate.exe"
	
	& c:\Windows\system32\inetsrv\AppCmd.exe add apppool /name:SampleWebSiteAppPool /managedRuntimeVersion:v4.0 /managedPipelineMode:Integrated
	& c:\Windows\system32\inetsrv\AppCmd.exe set config /section:applicationPools "/[name='SampleWebSiteAppPool'].processModel.identityType:LocalSystem"
	& c:\Windows\system32\inetsrv\AppCmd.exe add site /name:SampleWebSite /physicalPath:$physicalPath /bindings:http/*:80:samplewebsite.net
	& c:\Windows\system32\inetsrv\AppCmd.exe set app "SampleWebSite/" /applicationPool:"SampleWebSiteAppPool"
	
	& c:\Windows\system32\inetsrv\AppCmd.exe add apppool /name:AdminSiteAppPool /managedRuntimeVersion:v4.0 /managedPipelineMode:Integrated
	& c:\Windows\system32\inetsrv\AppCmd.exe set config /section:applicationPools "/[name='AdminWebAppPool'].processModel.identityType:LocalSystem"
	& c:\Windows\system32\inetsrv\AppCmd.exe add site /name:"AdminSite" /physicalpath:"$adminWebPhysicalPath" /bindings:http/*:80:adminsite.net
	& c:\Windows\system32\inetsrv\AppCmd.exe set app "AdminSite/" /applicationPool:"AdminSiteAppPool"
	
	& c:\Windows\system32\inetsrv\AppCmd.exe add apppool /name:GuestServicesAppPool /managedRuntimeVersion:v4.0 /managedPipelineMode:Integrated
	& c:\Windows\system32\inetsrv\AppCmd.exe set config /section:applicationPools "/[name='GuestServicesAppPool'].processModel.identityType:LocalSystem"
	& c:\Windows\system32\inetsrv\AppCmd.exe add site /name:"GuestServices" /physicalPath:$guestServicesPhysicalPath /bindings:http/*:80:guestservices.net
	& c:\Windows\system32\inetsrv\AppCmd.exe set app "GuestServices/" /applicationPool:"GuestServicesAppPool"
	
	& c:\Windows\system32\inetsrv\AppCmd.exe add apppool /name:FakeApiAppPool /managedRuntimeVersion:v4.0 /managedPipelineMode:Integrated
	& c:\Windows\system32\inetsrv\AppCmd.exe set config /section:applicationPools "/[name='FakeEmbedApiAppPool'].processModel.identityType:LocalSystem"
	& c:\Windows\system32\inetsrv\AppCmd.exe add site /name:"FakeApi" /physicalPath:$fakeEmbedApiPhysicalPath /bindings:http/*:80:siteapi.net
	& c:\Windows\system32\inetsrv\AppCmd.exe set app "FakeApi/" /applicationPool:"FakeApiAppPool"
	
	Write-Host "Add the following lines to your hosts file:" -ForegroundColor red
	Write-Host "127.0.0.1      samplewebsite.net" -ForegroundColor yellow
	Write-Host "127.0.0.1      adminsite.net" -ForegroundColor yellow
	Write-Host "127.0.0.1      guestservices.net" -ForegroundColor yellow
	Write-Host "127.0.0.1      siteapi.net" -ForegroundColor yellow
	Write-Host ""
	
	Start-Process $elevate -ArgumentList "notepad c:\Windows\system32\drivers\etc\hosts"
	
You can also put unattended installs for SQL, or any other dependencies your project has.
	
## Build Server considerations
We also were running UI Tests using [http://teststack.net/TestStack.Seleno/](http://teststack.net/TestStack.Seleno/)

This meant we needed our build server setup to be able to run the project, as well as build it. The great thing about this script, is you can reuse it to set up your build server.

For the build server, we had another script, which would re-map the IIS sites to the correct folder (as teamcity can checkout your code to different folders, we didn't want to set things up over and over).

The script looked like this

	$invocation = (Get-Variable MyInvocation).Value
	$directorypath = Split-Path $invocation.MyCommand.Path
	$physicalPath = Join-Path $directorypath "src\SampleWebSite"
	$adminWebPhysicalPath = Join-Path $directorypath "src\AdminWeb"
	$guestServicesPhysicalPath = Join-Path $directorypath "src\GuestServices"
	$fakeApiPhysicalPath = Join-Path $directorypath "src\FakeApi"
	$elevate = Join-Path $directorypath "tools\Elevate.exe"
	
	& c:\Windows\system32\inetsrv\AppCmd.exe set vdir "SampleWebSite/" /physicalPath:"$physicalPath"
	& c:\Windows\system32\inetsrv\AppCmd.exe set vdir "AdminSite/" /physicalpath:"$adminWebPhysicalPath"
	& c:\Windows\system32\inetsrv\AppCmd.exe set vdir "GuestServices/" /physicalpath:"$guestServicesPhysicalPath"
	& c:\Windows\system32\inetsrv\AppCmd.exe set vdir "FakeApi/" /physicalpath:"$fakeApiPhysicalPath"

Just execute this as part of your build process before you run your UI Automation tests.

## Wrapping up

I encourage you to create your own SetupEnvironment.ps1 script on the project you are on at the moment, it will be well worth the time investment over the lifetime of the project (we found it had saved us time within a few weeks).