---
layout: post
title: ApiApprover
metaTitle: ApiApprover
description: 
revised: 2012-02-19
date: 2012-02-19
categories: [open-source,nuget,semanticversioning]
migrated: true
comments: true
sharing: true
footer: true
permalink: /apiapprover/
summary: | 
  

---
Recently I had an issue at work, we wanted to guarantee we had no breaking public API changes, and wanted to start adhering to semantic versioning.

The interesting thing about Semantic versioning is that often people accidently break the semantic version. So I wrote a unit test which solves this problem

        [Fact]
        public void phoenix_has_no_public_api_changes()
        {
            // arrange
            var phoenix = typeof(IPhoenixHost).Assembly;

            // act
            var publicApi = CodeGen.CreatePublicApiForAssembly(phoenix);

            // assert
            var reporter = new DiffReporter();
            Approvals.Approve(new ApprovalTextWriter(publicApi), new XUnitTestFrameworkNamer(), reporter);
        }
        
# What it does
This test is actually quite simple, we grab the assembly, I then have a class which generates the public API for that assembly as a string. I then use Approval Tests to approve any API changes with a diff tool.

So what does that actually look like:  
![ApiChange](/assets/posts/2012-02-19-apiapprover/ApiChange.png)

# How do I use it?

Add the `ApiApprover` NuGet package
It will drop a few files into your test project
PublicApiGenerator.cs
PublicApiApprovalTest.cs

Open up PublicApiApprovalTest and fix up the compilation error (specify assembly, and properly attribute up the test for your framework of choice. Then get started!