---
layout: post
title: Redirecting Process Output
metaTitle: Redirecting Process Output
description: When using Process.Start in .net it is actually not that easy to capture the output. Here is some code to help
revised: 2013-04-05
date: 2013-04-05
categories: [c#,.net]
migrated: true
comments: true
sharing: true
footer: true
permalink: /redirecting-process-output/
summary: | 
  When using Process.Start in .net it is actually not that easy to capture the output. Here is some code to help

---
I sometimes have the need to shell a process and redirect the output. But I often have issues with deadlocks between processes and other random issues.

Based on a blog post that Lucian Wischik posted a while back at [http://blogs.msdn.com/b/lucian/archive/2008/12/29/system-diagnostics-process-redirect-standardinput-standardoutput-standarderror.aspx](http://blogs.msdn.com/b/lucian/archive/2008/12/29/system-diagnostics-process-redirect-standardinput-standardoutput-standarderror.aspx) I have created a c# version of his code which makes it nice and easy.
<!-- more -->
# Usage

    string stdOut;
    string stdErr;
    var processStartInfo = new ProcessStartInfo(process, args)
    {
        UseShellExecute = false,
        RedirectStandardError = true,
        RedirectStandardInput = true,
        RedirectStandardOutput = true
    };
    Process.Start(processStartInfo).InputAndOutputToEnd(string.Empty, out stdOut, out stdErr);

    Console.Write(stdOut);
    Console.Write(stdErr);

# The Code
    public static class ProcessExtensions
    {
        public static void InputAndOutputToEnd(this Process p, string standardInput, out string standardOutput, out string standardError)
        {
            if (p == null)
                throw new ArgumentException("p must be non-null");
            // Assume p has started. Alas there's no way to check.
            if (p.StartInfo.UseShellExecute)
                throw new ArgumentException("Set StartInfo.UseShellExecute to false");
            if ((p.StartInfo.RedirectStandardInput != (standardInput != null)))
                throw new ArgumentException("Provide a non-null Input only when StartInfo.RedirectStandardInput");
            //
            var outputData = new InputAndOutputToEndData();
            var errorData = new InputAndOutputToEndData();

            //
            if (p.StartInfo.RedirectStandardOutput)
            {
                outputData.Stream = p.StandardOutput;
                outputData.Thread = new System.Threading.Thread(InputAndOutputToEndProc);
                outputData.Thread.Start(outputData);
            }
            if (p.StartInfo.RedirectStandardError)
            {
                errorData.Stream = p.StandardError;
                errorData.Thread = new System.Threading.Thread(InputAndOutputToEndProc);
                errorData.Thread.Start(errorData);
            }
            //
            if (p.StartInfo.RedirectStandardInput)
            {
                p.StandardInput.Write(standardInput);
                p.StandardInput.Close();
            }
            //
            if (p.StartInfo.RedirectStandardOutput)
            {
                outputData.Thread.Join();
                standardOutput = outputData.Output;
            }
            else
                standardOutput = string.Empty;

            if (p.StartInfo.RedirectStandardError)
            {
                errorData.Thread.Join();
                standardError = errorData.Output;
            }
            else
                standardError = string.Empty;

            if (outputData.Exception != null)
                throw outputData.Exception;
            if (errorData.Exception != null)
                throw errorData.Exception;
        }

        private class InputAndOutputToEndData
        {
            public System.Threading.Thread Thread;
            public System.IO.StreamReader Stream;
            public string Output;
            public Exception Exception;
        }

        private static void InputAndOutputToEndProc(object data)
        {
            var ioData = (InputAndOutputToEndData)data;
            try
            {
                ioData.Output = ioData.Stream.ReadToEnd();
            }
            catch (Exception e)
            {
                ioData.Exception = e;
            }
        }
    }


Hope this is useful