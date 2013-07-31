---
layout: post
title: TeamCity UI Test Agent
metaTitle: TeamCity UI Test Agent
description: Instructions to setup a TeamCity build agent, in Azure to run UI Automation Tests
revised: 2013-07-31
date: 2013-07-31
categories: [azure,teamcity,ui-automation]
migrated: true
comments: true
sharing: true
footer: true
permalink: /teamcity-ui-test-agent/
summary: | 
  Instructions to setup a TeamCity build agent, in Azure to run UI Automation Tests

---
# Setting up a UI Test build agent
Many UI automation frameworks automate not only using automation patterns, but also automate your mouse and keyboard. 
This means that you need a fully unlocked desktop for things to work correctly. This blog post will show you how to setup a UI Test agent on Azure VM's, but you can use your own vm infrastructure. 

I recommend using a VM, because otherwise you are leaving a desktop unlocked where anyone can come and use it. At least VM's run on a locked desktop, or on the cloud and you need to remote in.

## Create our VM on Azure

![NewDocument](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent_635109042213761250.png)

![NewDocument1](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent1_635109042218761250.png)
I choose Windows Server 2008 R2 as the Operating System, it is preferable to use a client operating system, but server OS's are all that are available in Azure. This also means you don't have to work around the fact that the 2012 start screen shows first, and we need to be on the desktop. 

![NewDocument2](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent2_635109042222198750.png)

Now we are ready to go, lets remote desktop into our VM

![NewDocument3](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent3_635109042225792500.png)

Enter your remote desktop credentials which we you setup when creating your VM
![NewDocument4](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent4_635109042229230000.png)

## Setting up your VM
On first login, make sure you tell the initial configuration and server manager to not open on start
![NewDocument6](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent6_635109042232667500.png)
![NewDocument7](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent7_635109042236105000.png)

Before you close server manager you want to click on `Configure IE ESC`, then turn it off. Otherwise downloading everything will be rather painful (unless you just want to download firefox or chrome, then don't worry.

Next, we need our VM to login automatically, if our VM restarts, it needs to come straight back up and logs in.

To do this, download Sysinternals Autologon for Windows from [http://technet.microsoft.com/en-us/sysinternals/bb963905.aspx](http://technet.microsoft.com/en-us/sysinternals/bb963905.aspx). Once you have downloaded, extracted, run and accepted the EULA you can enter the credentials to login with.
The advantage of using this tool rather than just putting it in the registry, is that your password will be encrypted rather than plain text :)
![NewDocument8](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent8_635109042239542500.png)
![NewDocument9](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent9_635109042242980000.png)

Next we need to install VNC onto the server, we cannot use remote desktop because after you disconnect the desktop will lock, and your tests will start failing.
TeamViewer will also work.

Personally I use TightVNC.

### Installing/Configuring TightVNC
1. Download from [http://www.tightvnc.com/download.php](http://www.tightvnc.com/download.php)
1. Do a complete install, leave all options ticked when presented with them.
1. Set your passwords, I am happy to not have a separate administration password.  
![NewDocument10](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent10_635109042246417500.png)
1. Install the DFMirage driver, available from the TightVNC download page
1. If using azure you need to open up the port (Manage VM, EndPoints, Add, Next)  
![NewDocument11](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent11_635109042249855000.png)

### Finishing VM Setup
Reconnect using something other than remote desktop  
![NewDocument12](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent12_635109042253292500.png)

Now you are logged in, bump the screen resolution up to 1280x1024 (or whatever suites you).

#### Disable WER
Windows error reporting causes issues when running UI automation, if you app crashes (which is why we have UI automation, to find that sort fo thing) then you want it to exit straight away, not popup the Windows Error Reporting Dialog
![NewDocument13](/assets/posts/2013-07-31-teamcity-ui-test-agent/SettingupUITestAgent13_635109042256730000.png)

Save the following text into a .reg file i.e DisableWER.reg then run

	Windows Registry Editor Version 5.00
	 
	[HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Windows]
	"ErrorMode"="2"
	 
	[HKEY_CURRENT_USER\Software\Microsoft\Windows\Windows Error Reporting]
	"DontShowUI"="1"

### Installing the teamcity build agent
![UITestAgent](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent_635109042263761250.png)  
![UITestAgent1](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent1_635109042267198750.png)  
![UITestAgent2](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent2_635109042273605000.png)  
![UITestAgent3](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent3_635109042290480000.png)  

Fill in your teamcity server URL, and note the port number the agent is running on
![UITestAgent4](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent4_635109042304073750.png)  

Now we go back into Azure Management, and add the port  
![UITestAgent5](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent5_635109042307511250.png)  

#### Open Filewall Ports
Once we have added the port on azure, we need to open the windows firewall for that port on the VM itself  
![UITestAgent8](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent8_635109042326886250.png)

![UITestAgent9](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent9_635109042330480000.png)  
Put in port 9090, or whatever you set your teamcity server to  
![UITestAgent10](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent10_635109042341573750.png)  
Next, Next, Next, give it a good name 'TeamCity Build Agent', Finish

#### Set to automatic startup
Now open explorer, and go to `%userprofile%\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup` and create a new shortcut  
![UITestAgent6](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent6_635109042366730000.png)

Browse to your TeamCity build agent folder, and select `agent.bat`

![UITestAgent7](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent7_635109042381730000.png)  
Then add the parameter `start` onto the path. You should end up with
`"C:\UITestsBuildAgent\bin\agent.bat" start`
Then give it a good name like 'Start UI Test Agent', click finish. Then Run the shortcut. Your TeamCity build agent should startup and connect to TeamCity
![UITestAgent11](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent11_635109042385167500.png)

Authorise the build agent, then the agent should update itself and restart, after a few minutes you should have another build controller online!
![UITestAgent12](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent12_635109042398448750.png)

![UITestAgent13](/assets/posts/2013-07-31-teamcity-ui-test-agent/UITestAgent13_635109042408292500.png)

## Conclusion
There you have it, a build agent that runs UI Tests on an unlocked desktop in Azure

### NOTE: TeamCity is not setup for SSL, so everything is unencrypted. This would be another blog post in itself, please leave a comment if that would be useful for you?