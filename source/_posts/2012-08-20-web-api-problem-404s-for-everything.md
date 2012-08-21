---
layout: post
title: Web Api Problem - 404s for everything
metaTitle: Web Api Problem - 404s for everything
description: 
revised: 2012-08-21
date: 2012-08-20
categories: [WebApi]
migrated: true
comments: true
sharing: true
footer: true
permalink: /web-api-problem-404s-for-everything/
summary: | 
  

---
# Scenario
So the scenario is that we are in the middle of migrating all our services into Azure, and there will be a time when our services will be running on premise and also in Azure.

We added this class to our project and added a reference to `Microsoft.WindowsAzure.ServiceRuntime` (from the GAC):

    public class WebRole : RoleEntryPoint
    {
        public override bool OnStart()
        {
            //Start code
            return base.OnStart();
        }
    }

Our WebApi site starts 404'ing. There was also a heap of other work which was brought in at the same time, so tracking this down was rather hard.

## The issue
So I started work this morning investigating why our UI automation tests were all hanging/broken. I noticed our message processor was logging errors 'Not Found', restarting everything I managed to get something which looked like:

    {"Message":"No HTTP resource was found that matches the request URI 'http://localhost/TestSite/api/values'.","MessageDetail":"No type was found that matches the controller named 'values'."}

Remember that WebRole class we added, well RoleEntryPoint comes from `Microsoft.WindowsAzure.ServiceRuntime` is referenced from the GAC, and it seems that WebApi reflects over all types in the solution which would have caused a TypeLoadException to be thrown. This apparently causes Web Api to not work at all.... I also have no idea where this issue comes from, as I cannot see an exception at all when debugging.

## To Reproduce
1. Create a new Web API project
2. Add a new class called MyDbContext, inheriting from DbContext:

	public void MyDbContext : DbContext {}
	
3. Change the entity framework dll to CopyLocal=False

F5, and try and hit your api's. 

Any insights as to why this is happening and how to log the error would be awesome....