---
layout: post
title: Getting Started with ApprovalTests without installing TortoiseSVN
metaTitle: Getting Started with ApprovalTests without installing TortoiseSVN
description: If you don't have TortoiseSVN installed the DiffReporter will fail, use Chocolatey to get started!
revised: 2012-03-08
date: 2012-03-08
categories: []
migrated: true
comments: true
sharing: true
footer: true
permalink: /getting-started-with-api-approver/
summary: | 
  

---
I have just installed Windows 8 consumer preview on my dev machine, then all my API Approval tests started failing with this exception because I don't have TortoiseMerge installed =(

    System.ComponentModel.Win32Exception: System.ComponentModel.Win32Exception : The system cannot find the file specified
       at System.Diagnostics.Process.StartWithShellExecuteEx(ProcessStartInfo startInfo)
       at System.Diagnostics.Process.Start()
       at System.Diagnostics.Process.Start(ProcessStartInfo startInfo)
       at System.Diagnostics.Process.Start(String fileName, String arguments)
       at ApprovalTests.Reporters.DiffReporter.Launch(LaunchArgs launchArgs)
       at ApprovalTests.Reporters.DiffReporter.Report(String approved, String received)
       at ApprovalTests.Approvers.FileApprover.ReportFailure(IApprovalFailureReporter reporter)
       at ApprovalTests.Core.Approvals.Approve(IApprovalApprover approver, IApprovalFailureReporter reporter)
       at ApprovalTests.Approvals.Approve(IApprovalWriter writer, IApprovalNamer namer, IApprovalFailureReporter reporter)
       at Phoenix.Tests.ApiChanges.phoenix_has_no_public_api_changes() in C:\Users\Jake\_Code\Phoenix\src\net40\Phoenix.Tests\ApiChanges.cs:line 22
    System.ComponentModel.Win32Exception: System.ComponentModel.Win32Exception : The system cannot find the file specified
       at System.Diagnostics.Process.StartWithShellExecuteEx(ProcessStartInfo startInfo)
       at System.Diagnostics.Process.Start()
       at System.Diagnostics.Process.Start(ProcessStartInfo startInfo)
       at System.Diagnostics.Process.Start(String fileName, String arguments)
       at ApprovalTests.Reporters.DiffReporter.Launch(LaunchArgs launchArgs)
       at ApprovalTests.Reporters.DiffReporter.Report(String approved, String received)
       at ApprovalTests.Approvers.FileApprover.ReportFailure(IApprovalFailureReporter reporter)
       at ApprovalTests.Core.Approvals.Approve(IApprovalApprover approver, IApprovalFailureReporter reporter)
       at ApprovalTests.Approvals.Approve(IApprovalWriter writer, IApprovalNamer namer, IApprovalFailureReporter reporter)


My issue is I don't want to install TortoiseSVN just to get TortoiseMerge which ApiApprover uses by default.

So I created a Chocolatey package at [http://chocolatey.org/packages/tortoisemerge](http://chocolatey.org/packages/tortoisemerge) which will download and add TortoiseMerge to your Path, then Approval Tests DiffReporter will just start working :)

# Install Steps

## 1. Install Chocolatey
**PS:\>** `iex ((new-object net.webclient).DownloadString('http://bit.ly/psChocInstall'))`

## 2. Install TortoiseMerge via Chocolatey
**PS:\>** `cinst TortoiseMerge`

Done :)