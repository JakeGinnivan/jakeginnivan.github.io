---
layout: post
title: Markpad Performance Push
metaTitle: Markpad Performance Push
description: 
revised: 2012-06-30
date: 2012-08-12
categories: [wpf,code52]
migrated: true
comments: true
sharing: true
footer: true
permalink: /markpad-performance-push/
summary: | 
  

---
If you haven't seen Markpad, think Windows Live Writer, but for Jekyll, Pretzel, FunnelWeb blogs, and also editing normal markdown files.

![MarkPad](/assets/posts/2012-08-12-markpad-performance-push/screenshot.png)

# Hosting Awesomium in it's own AppDomain
This was by far the biggest performance win for us, we moved Awesomium (the .net wrapper around the Chrome rendering engine) into it's own AppDomain, and show 'Preview Loading...' much like visual studio does for it's designer.

Basically the way it works is we have a HtmlPreview.xaml control which is pretty simple:

    <UserControl x:Class="MarkPad.PreviewControl.HtmlPreview"
                 xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                 xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                 Loaded="HtmlPreviewLoaded">
      <UserControl.Effect>
        <DropShadowEffect BlurRadius="10"
                          Color="Black"
                          Opacity="0.25"
                          Direction="270" />
      </UserControl.Effect>
    
      <Border BorderThickness="0"
              Background="White">
        <TextBlock Text="Preview loading..."
                   HorizontalAlignment="Center"
                   VerticalAlignment="Center" />
      </Border>
    </UserControl>

The magic is inside the Loaded event of this control:

    private void HtmlPreviewLoaded(object sender, RoutedEventArgs e)
    {
        Loaded -= HtmlPreviewLoaded;
        var context = TaskScheduler.FromCurrentSynchronizationContext();

        //We are hosting the Awesomium preview in another appdomain so our main UI thread does not take the hit
        hostAppDomain = AppDomain.CreateDomain("HtmlPreviewDomain");
        var filename = FileName;

        // create the AppDomain on a new thread as we want to ensure it is an 
        // STA thread as this makes life easier for creating UI components
        var thread = new Thread(() =>
        {
            var awesomiumHostType = typeof(AwesomiumHost);
            host = (AwesomiumHost)hostAppDomain.CreateInstanceAndUnwrap(awesomiumHostType.Assembly.FullName, awesomiumHostType.FullName,
            false, BindingFlags.Default, null, new object[] { filename, BaseDirectory }, CultureInfo.CurrentCulture, null);

            host.SetHtml(content);

            var controlHandle = host.ControlHandle;

            Task.Factory.StartNew(() =>
            {
                //Delay until preview control has loaded before creating content host
                host.LoadedWaitHandle.WaitOne();

                // We need to invoke on the Markpad dispatcher, we are currently in the host appdomains STA Thread.
                Dispatcher.BeginInvoke(new Action(() =>
                {
                    hwndContentHost = new HwndContentHost(controlHandle);
                    //Without the border we don't get the dropshadows
                    Content = new Border
                    {
                        Background = Brushes.White,
                        Padding = new Thickness(3),
                        Child = hwndContentHost
                    };
                }));
            }, TaskCreationOptions.LongRunning);

            host.Run();
            //I can't get this unloading without an error, 
            // I am gathering Application.Shutdown is causing the appdomain to shutdown too
            //AppDomain.Unload(hostAppDomain);
        });

        thread.SetApartmentState(ApartmentState.STA);
        thread.Start();
    }

So what does this actually do!

First we create the AppDomain, and spin up a dedicated thread which will be the UI thread of new AppDomain.

    hostAppDomain = AppDomain.CreateDomain("HtmlPreviewDomain");

    var thread = new Thread(() =>
    {
        // We initialise it all in here
    });

    thread.SetApartmentState(ApartmentState.STA);
    thread.Start();

Inside this new thread, we create an instance of the AwesomiumHost, which is a class which can be marshalled across AppDomains

    public class AwesomiumHost : MarshalByRefObject, IDisposable
    {
        public string FileName { get; private set; }
        public string Html { get; set; }
        public double ScrollPercentage { get; set; }

        public IntPtr ControlHandle { get; }
        public ManualResetEvent LoadedWaitHandle { get; }

        public void SetHtml(string content);
        public void WbProcentualZoom();
        public void Print();
        public void Run();

        public void Dispose();
    }

So we create the instance and unwrap it:

    var awesomiumHostType = typeof(AwesomiumHost);
    host = (AwesomiumHost)hostAppDomain.CreateInstanceAndUnwrap(awesomiumHostType.Assembly.FullName, awesomiumHostType.FullName,
            false, BindingFlags.Default, null, new object[] { filename, BaseDirectory }, CultureInfo.CurrentCulture, null);

    host.SetHtml(content);
    var controlHandle = host.ControlHandle;

This has gone and created the control, and got the ControlHandle which we need to host WPF controls across appdomains, there is some nasty code in the `static long CreateWindowHandle(Visual frameworkElement)` method if you want to know how to do it.

We then start a long running task (background operation in the preview appdomain) which blocks until the control has fully loaded (the LoadedWaitHandle is set once the Loaded event fires on the Awesomium control).

    Task.Factory.StartNew(() =>
    {
        //Delay until preview control has loaded before creating content host
        host.LoadedWaitHandle.WaitOne();

        // We need to invoke on the Markpad dispatcher, we are currently in the host appdomains STA Thread.
        Dispatcher.BeginInvoke(new Action(() =>
        {
            hwndContentHost = new HwndContentHost(controlHandle);
            //Without the border we don't get the dropshadows
            Content = new Border
            {
                Background = Brushes.White,
                Padding = new Thickness(3),
                Child = hwndContentHost
            };
        }));
    }, TaskCreationOptions.LongRunning);

We then invoke the creation of the content host on Markpads UI thread, and replace the content of the UserControl with the MarkPad preview.

There are a few more things, but this should help you follow the Markpad codebase if you ever want to host a WPF control in another AppDomain :)