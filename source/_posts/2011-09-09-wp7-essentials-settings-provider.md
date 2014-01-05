---
layout: post
title: WP7 Essentials Settings Provider
metaTitle: WP7 Essentials Settings Provider
description: The WP7 Essentials library is constantly evolving, the most recent addition is a UI for settings used with the Settings Provider
revised: 2011-09-10
date: 2011-09-09
categories: [Windows Phone, Open Source]
migrated: true
comments: true
sharing: true
footer: true
permalink: /wp7-essentials-settings-provider/
summary: | 
  The WP7 Essentials library is constantly evolving, the most recent addition is a UI for settings used with the Settings Provider

---
Update: This project has been superseeded by [SettingsProvider.net](https://github.com/JakeGinnivan/SettingsProvider.net)
<!-- more -->
# Settings Provider?
The settings provider is a port from the FunnelWeb settings provider to Windows Phone. It is a really simple interface

    public interface ISettingsProvider
    {
        T GetSettings<T>() where T : ISettings, new();
        void SaveSettings<T>(T settings) where T : ISettings;
    }

So what does `T` actually look like:

    public class ApplicationSettings : NotifyPropertyChanged, ISettings
    {
        [DefaultValue(5)]
        [DisplayName("Number Results")]
        public int NumberResults { get; set; }

        public string Nickname { get; set; }

        [DefaultValue(true)]
        [DisplayName("Enable Stuff")]
        public bool EnableStuff { get; set; }

        public DateTime? Birthday { get; set; }

        [DisplayName("Some Options")]
        public Options SomeOptions { get; set; }
    }

    public enum Options
    {
        Value1, 
        Value2
    }

Pretty easy right? We can use NotifyPropertyWeaver to keep the class nice and clean. Then to fetch or persist our settings we just need to do

    var applicationSettings = settingsProvider.GetSettings<ApplicationSettings>();
    applicationSettings.SomeOptions = Options.Value2;
    settingsProvider.SaveSettings(applicationSettings);

The Settings Provider is limited to types supported by the Convert.ChangeType method (for a number of reasons, if this is too restrictive, let me know and why).

So in the last release we added the SettingsList control, which will generate this for you (just the settings control, not the page, highlighted in red is what you get generated)

![Settings Provider control](/assets/posts/2011-09-09-wp7-essentials-settings-provider/SettingsProvider.png)

This is a first cut, and over time I will polish this page, and add support for ordering the properties and stuff (or feel free to submit pull requests :)). But I recon this is pretty cool for a first cut.

Also the reason I am using a combobox instead of the ListPicker is there is a bug where the control simply doesn't work inside a scroll viewer. Bah. Anyone interested in doing a community fork which simply fixes bugs in the toolkit and doesn't add any new features?

# Get it

This control is in a separate package (to keep the essentials package lean and mean :P).

    Install-Package WindowsPhoneEssentials.Controls.Settings

[Codeplex](http://wp7essentials.codeplex.com/)  
[NuGet](http://nuget.org/List/Search?searchTerm=WindowsPhoneEssentials)