---
layout: post
title: Azure and msshrtmi.dll
metaTitle: Azure and msshrtmi.dll
description: When adding Azure support to FunnelWeb, I ran into issues with Azure and msshrtmi.dll. Here is my solutions
revised: 2012-06-30
date: 2012-06-30
categories: [Azure]
migrated: true
comments: true
sharing: true
footer: true
permalink: /azure-and-msshrtmi/
summary: | 
  

---
## The Problem
A while back we received some Azure pull requests which added an Azure project and fixed all our SQL scripts so they were compatible with SQL Azure and that was pretty good, FunnelWeb would run in Azure.

The problem though is updates to the my.config required a full new deployment, so I wanted to support loading the configuration from the Role Environment config files instead when running in Azure.

So in our configuration settings class I put something like this:

    public string Get(string name)
    {
        return (RoleEnvironment.IsAvailable)
                    ? RoleEnvironment.GetConfigurationSettingValue(name)
                    : myConfigSettings.Get(name);
    }

All good, local testing works beautifully. So I start with deploying to Azure Websites via git using the same script we use to deploy to AppHarbor. And boom:

    ---> System.TypeInitializationException: The type initializer for 'Microsoft.WindowsAzure.ServiceRuntime.RoleEnvironment' threw an exception. ---> System.IO.FileNotFoundException: Could not load file or assembly 'msshrtmi, Version=1.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35' or one of its dependencies. The system cannot find the file specified.
       at Microsoft.WindowsAzure.ServiceRuntime.RoleEnvironment.InitializeEnvironment()
       at Microsoft.WindowsAzure.ServiceRuntime.RoleEnvironment..cctor()
       --- End of inner exception stack trace ---
       at Microsoft.WindowsAzure.ServiceRuntime.RoleEnvironment.get_IsAvailable()
<!-- more -->
It seems the static initialiser for RoleEnvironment relies on a native dll which is installed into the GAC by the Azure SDK... Isn't the point of IsAvailable to check if you are running in Azure... =(

So I start searching for that assembly, turns out it lives at `"C:\Program Files\Microsoft SDKs\Windows Azure\.NET SDK\2012-06\bin\runtimes\base\x64\msshrtmi.dll"` and `"C:\Program Files\Microsoft SDKs\Windows Azure\.NET SDK\2012-06\bin\runtimes\base\x86\msshrtmi.dll"` for the two different CPU architectures.

So I start off by putting the x64 version in the bin directory, which then blew up with

    Could not load file or assembly 'msshrtmi' or one of its dependencies. An attempt was made to load a program with an incorrect format.

So I put the x86 version into the bin directory, and huzzah we were up and running. I then updated my blog to the latest build (anyone on here yesterday would have seen it going up and down for about an hour :P) and boom, back to `Could not load file or assembly 'msshrtmi' or one of its dependencies. An attempt was made to load a program with an incorrect format.`

Obviously this is not a good solution, and FunnelWeb now can only run on a x86 app pool, which my host is running an x64 app pool.

## My Solution
I have removed all the other FunnelWeb initialisation code, and just left the important parts.

    public class MvcApplication : HttpApplication
    {
        private static string basePath;

        private void Application_Start()
        {
            basePath = Server.MapPath("/");
            AppDomain.CurrentDomain.AssemblyResolve += CurrentDomain_AssemblyResolve;
        }

        // Unfortunately this unmanaged dll is required by azure to check if we are running in azure
        // There is also a x86 and x64 version which is why we have to take this approach
        static Assembly CurrentDomain_AssemblyResolve(object sender, ResolveEventArgs args)
        {
            if (args.Name.StartsWith("msshrtmi,", StringComparison.OrdinalIgnoreCase))
            {
                var fileName = Path.Combine(basePath, "bin", ((IntPtr.Size == 4) ? "x86" : "amd64"), "msshrtmi.dll");

                AppDomain.CurrentDomain.AssemblyResolve -= CurrentDomain_AssemblyResolve;

                return Assembly.LoadFile(fileName);
            }

            return null;
        }
    }

I then create a amd64 and x86 folder in the `_bin_deployableAssemblies` folder so they get copied into the bin folder during the build.
When the AppDomain tries to resolve the native file, I dynamically load the correct version.

Comments on my approach would be great.