---
layout: post
title: MultiTargeted WP7 Project
metaTitle: MultiTargeted WP7 Project
description: After seeing Shaun Wildermuths post on this subject, I thought I would share how I do it
revised: 2011-08-23
date: 2011-08-22
categories: [wp7,open-source,msbuild]
migrated: true
comments: true
sharing: true
footer: true
permalink: /multitargeted-windows-phone-project/
summary: | 
  

---
I saw Shawn Wildermuth's post on maintaining a project targeting multiple versions of windows phone, I thought I would share the way I do it as I think it is far easier than the alternatives.
[http://wildermuth.com/2011/08/23/Maintaining_a_Project_with_Two_Windows_Phone_Versions](http://wildermuth.com/2011/08/23/Maintaining_a_Project_with_Two_Windows_Phone_Versions)

# Multi-Targeted .csproj file

First off I open up my .csproj file and add this under the initial property group

    <TargetFrameworkProfile Condition="'$(TargetFrameworkProfile)' == ''">WindowsPhone71</TargetFrameworkProfile>

This means I target Windows Phone 71 by default, then I modify the define constants property under each build configuration to be conditional:


    <DefineConstants Condition="'$(TargetFrameworkProfile)' == 'WindowsPhone'">TRACE;SILVERLIGHT;WINDOWS_PHONE</DefineConstants>
    <DefineConstants Condition="'$(TargetFrameworkProfile)' == 'WindowsPhone71'">TRACE;SILVERLIGHT;WINDOWS_PHONE71</DefineConstants>

Now I can easily include files in only my mango build, for example:

    #if WINDOWS_PHONE71
    namespace WindowsPhoneMVC.Navigation
    {
        public static class DeepLink
        {
            public static string UriFor(string controller, string action, IDictionary<string, string> parameters)
            {
                var parametersString = string.Join("&", parameters.Select(p => p.Key + "=" + p.Value));
                return string.Format("/Shell.xaml?controller={0}&action={1}{2}{3}", controller, action, 
                    string.IsNullOrEmpty(parametersString) ?  string.Empty : "&", parametersString);
            }

            public static NavigationRequest DecodeUri(string navigationFrame, string deepLink)
            {
                var kvp = deepLink.Split(new[] {'&'})
                    .Select(o => o.Split(new[] {'='}))
                    .ToDictionary(k => k[0], k => k[1]);

                string controller = null;
                if (kvp.ContainsKey("controller"))
                {
                    controller = kvp["controller"];
                    kvp.Remove("controller");
                }

                string action = null;
                if (kvp.ContainsKey("action"))
                {
                    action = kvp["action"];
                    kvp.Remove("action");
                }

                return new NavigationRequest(navigationFrame, controller, action, new NavigationParameter(kvp));
            }
        }
    }
    #endif

This way in visual studio I always build for 71, then my build.cmd which looks like this

    @echo off
    call "%VS100COMNTOOLS%vsvars32.bat"
    mkdir .\build\log\

    msbuild.exe /ToolsVersion:4.0 "WindowsPhoneMVC.msbuild" 

    pause

Then the important part of that msbuild script, which builds my project as 7.0 and 7.1. After it builds it goes on to create my NuGet packages and releases meaning I can make a fix and test the new NuGet package locally within minutes, it works quite nice.

    <Target Name="Compile" DependsOnTargets="Version">
        <MSBuild Projects="$(Root)src\WindowsPhoneMVC\WindowsPhoneMVC.csproj"
                 Properties="Configuration=Release;Platform=Any CPU;OutputPath=bin\Build;TargetFrameworkProfile=WindowsPhone;" />

        <MSBuild Projects="$(Root)src\WindowsPhoneMVC\WindowsPhoneMVC.csproj"
                 Properties="Configuration=Release;Platform=Any CPU;OutputPath=bin\Build_71;TargetFrameworkProfile=WindowsPhone71;" />
    </Target>

### Complete build script

Here is my complete build script if you are interested, or just go to [http://windowsphonemvc.codeplex.com](http://windowsphonemvc.codeplex.com) to grab the latest version out of source control.

    <Project DefaultTargets="Package" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
      <PropertyGroup>
        <MSBuildCommunityTasksPath>$(MSBuildProjectDirectory)\build\lib\</MSBuildCommunityTasksPath>
        <Root>$(MSBuildProjectDirectory)\</Root>
        <Major>0</Major>
        <Minor>4</Minor>
        <Build>0</Build>
        <Revision>0</Revision>

        <NuGet>$(Root)build\lib\NuGet.exe</NuGet>
        <ContentSource>$(Root)src\SampleProjects\NuGetContentSource\</ContentSource>
      </PropertyGroup>

      <Import Project="$(MSBuildCommunityTasksPath)\MSBuild.Community.Tasks.Targets"/>
      <Import Project="$(MSBuildCommunityTasksPath)\MSBuild.Deployment.Tasks.Targets"/>
      <Import Project="$(MSBuildCommunityTasksPath)\MSBuild.Mercurial.tasks"/>

      <Target Name="GetVersion">
        <Error Condition="$(MSBuildCommunityTasksPath) == ''" Text="MSBuildCommunityTasksPath variable must be defined" />
        <Error Condition="$(Root) == ''" Text="Root variable must be defined" />

        <HgVersion LocalPath=".">
          <Output TaskParameter="Revision" PropertyName="Revision" />
        </HgVersion>
        <!-- Diagnostics -->
        <Message Text="Diagnostics:"/>
        <Message Text="Build Number:    $(Major).$(Minor).$(Build).$(Revision)" />
        <Message Text="Project root:    $(Root)" />
        <Message Text="Drop path:       build\Artifacts" />

        <!-- Clean up -->
        <ItemGroup>
          <FilesToDelete Include="$(Root)**\bin\**\*.*" />
          <FilesToDelete Include="$(Root)**\obj\**\*.*" />
        </ItemGroup>
        <Delete Files="@(FilesToDelete)" />
      </Target>

      <Target Name="Version" DependsOnTargets="GetVersion">
        <RemoveDir Directories="build\artifacts\" />
        <RemoveDir Directories="build\temp\" />

        <AssemblyInfo CodeLanguage="CS"
		      OutputFile="$(Root)src\WindowsPhoneMVC\Properties\VersionInfo.cs"
		      AssemblyVersion="$(Major).$(Minor).$(Build).$(Revision)"
		      AssemblyFileVersion="$(Major).$(Minor).$(Build).$(Revision)"
		      Condition="$(Revision) != '-1' "/>
      </Target>

      <Target Name="Compile" DependsOnTargets="Version">
        <MSBuild Projects="$(Root)src\WindowsPhoneMVC\WindowsPhoneMVC.csproj"
             Properties="Configuration=Release;Platform=Any CPU;OutputPath=bin\Build;TargetFrameworkProfile=WindowsPhone;" />
        <MSBuild Projects="$(Root)src\WindowsPhoneMVC.Extensions.Transitions\WindowsPhoneMVC.Extensions.Transitions.csproj"
             Properties="Configuration=Release;Platform=Any CPU;OutputPath=bin\Build;TargetFrameworkProfile=WindowsPhone;" />
        <MSBuild Projects="$(Root)src\WindowsPhoneMVC.Extensions.AutofacIntegration\WindowsPhoneMVC.Extensions.AutofacIntegration.csproj"
             Properties="Configuration=Release;Platform=Any CPU;OutputPath=bin\Build;TargetFrameworkProfile=WindowsPhone;" />

        <MSBuild Projects="$(Root)src\WindowsPhoneMVC\WindowsPhoneMVC.csproj"
             Properties="Configuration=Release;Platform=Any CPU;OutputPath=bin\Build_71;TargetFrameworkProfile=WindowsPhone71;" />
        <MSBuild Projects="$(Root)src\WindowsPhoneMVC.Extensions.Transitions\WindowsPhoneMVC.Extensions.Transitions.csproj"
             Properties="Configuration=Release;Platform=Any CPU;OutputPath=bin\Build_71;TargetFrameworkProfile=WindowsPhone71;" />
        <MSBuild Projects="$(Root)src\WindowsPhoneMVC.Extensions.AutofacIntegration\WindowsPhoneMVC.Extensions.AutofacIntegration.csproj"
             Properties="Configuration=Release;Platform=Any CPU;OutputPath=bin\Build_71;TargetFrameworkProfile=WindowsPhone71;" />

        <MSBuild Projects="$(Root)src\Templates\WPMvcTemplatesExtension\WPMvcTemplatesExtension.csproj"
             Properties="Configuration=Release;OutputPath=bin\Build" />
      </Target>

      <Target Name="NuGet" DependsOnTargets="Compile">
        <MakeDir Directories="$(Root)build\artifacts" />

        <ItemGroup>
          <AutofacContent Include="$(Root)build\autofaccontent\**\*.*" />
          <TransitionsContent Include="$(Root)build\transitionscontent\**\*.*" />
        </ItemGroup>

        <!--Main NuGet package-->
        <CallTarget Targets="MvcNuGet" />
        <CallTarget Targets="MvcLibsNuGet" />
        <CallTarget Targets="AutofacNuGet" />
        <CallTarget Targets="TransitionsNuGet" />
      </Target>

      <Target Name="MvcNuGet">
        <PropertyGroup>
          <NuGetManifest>$(Root)build\temp\WindowsPhoneMVC.nuspec</NuGetManifest>
          <MainNuGetContent>$(Root)build\temp\NuGet\content\</MainNuGetContent>
        </PropertyGroup>

        <MakeDir Directories="$(Root)build\temp\NuGet\WindowsPhoneMVC\lib" />
        <Copy SourceFiles="$(Root)build\WindowsPhoneMVC.nuspec"
			      DestinationFiles="$(NuGetManifest)" />

        <FileUpdate Files="$(NuGetManifest)"
					    Regex="0.0.0.1"
					    ReplacementText="$(Major).$(Minor).$(Build).$(Revision)"/>

        <Copy SourceFiles="$(ContentSource)Controllers\HomeController.cs" DestinationFolder="$(MainNuGetContent)Controllers\" />
        <Copy SourceFiles="$(ContentSource)ViewModels\Home\AboutViewModel.cs" DestinationFolder="$(MainNuGetContent)ViewModels\Home\" />
        <Copy SourceFiles="$(ContentSource)ViewModels\Home\MainViewModel.cs" DestinationFolder="$(MainNuGetContent)ViewModels\Home\" />
        <Copy SourceFiles="$(ContentSource)Views\Home\MainPage.xaml" DestinationFolder="$(MainNuGetContent)Views\Home\" />
        <Copy SourceFiles="$(ContentSource)Views\Home\MainPage.xaml.cs" DestinationFolder="$(MainNuGetContent)Views\Home\" />
        <Copy SourceFiles="$(ContentSource)Shell.xaml" DestinationFolder="$(MainNuGetContent)" />
        <Copy SourceFiles="$(ContentSource)Shell.xaml.cs" DestinationFolder="$(MainNuGetContent)" />
        <Copy SourceFiles="$(ContentSource)WindowsPhoneMVC_GettingStarted.htm" DestinationFolder="$(MainNuGetContent)" />

        <ItemGroup>
          <MvcContentFiles Include="$(MainNuGetContent)\**\*.*" />
        </ItemGroup>
        <FileUpdate Files="@(MvcContentFiles)"
					    Regex="NuGetContentSource"
					    ReplacementText="$rootnamespace$"/>
        <Move SourceFiles="@(MvcContentFiles)"
			      DestinationFiles="@(MvcContentFiles->'$(MainNuGetContent)%(RecursiveDir)%(Filename)%(Extension).pp')" />

        <!--SL3-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC\bin\Build\WindowsPhoneMVC.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL3-WP\WindowsPhoneMVC.dll"/>
        <!--SL4-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC\bin\Build\WindowsPhoneMVC.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL4-WindowsPhone\WindowsPhoneMVC.dll"/>
        <!--SL4_71-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC\bin\Build_71\WindowsPhoneMVC.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL4-WindowsPhone71\WindowsPhoneMVC.dll"/>
        <Exec Command='"$(NuGet)" pack "$(NuGetManifest)" -BasePath "$(Root)build\temp\NuGet" -OutputDirectory "$(Root)build\artifacts"' />
        <RemoveDir Directories="build\temp\" />
      </Target>

      <Target Name="MvcLibsNuGet">
        <PropertyGroup>
          <NuGetManifest>$(Root)build\temp\WindowsPhoneMVC.Libs.nuspec</NuGetManifest>
        </PropertyGroup>

        <MakeDir Directories="$(Root)build\temp\NuGet\WindowsPhoneMVC\lib" />
        <Copy SourceFiles="$(Root)build\WindowsPhoneMVC.nuspec"
			      DestinationFiles="$(NuGetManifest)" />

        <FileUpdate Files="$(NuGetManifest)"
					    Regex="0.0.0.1"
					    ReplacementText="$(Major).$(Minor).$(Build).$(Revision)"/>

        <!--SL3-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC\bin\Build\WindowsPhoneMVC.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL3-WP\WindowsPhoneMVC.dll"/>
        <!--SL4-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC\bin\Build\WindowsPhoneMVC.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL4-WindowsPhone\WindowsPhoneMVC.dll"/>
        <!--SL4_71-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC\bin\Build_71\WindowsPhoneMVC.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL4-WindowsPhone71\WindowsPhoneMVC.dll"/>
        <Exec Command='"$(NuGet)" pack "$(NuGetManifest)" -BasePath "$(Root)build\temp\NuGet" -OutputDirectory "$(Root)build\artifacts"' />
        <RemoveDir Directories="build\temp\" />
      </Target>

      <Target Name="AutofacNuGet">
        <PropertyGroup>
          <NuGetAutofacManifest>$(Root)build\temp\WindowsPhoneMVC.Extensions.AutofacIntegration.nuspec</NuGetAutofacManifest>
          <MainNuGetContent>$(Root)build\temp\NuGet\content\</MainNuGetContent>
        </PropertyGroup>

        <MakeDir Directories="$(Root)build\temp\NuGet\WindowsPhoneMVC.Extensions.AutofacIntegration\lib" />
        <Copy SourceFiles="$(Root)build\WindowsPhoneMVC.Extensions.AutofacIntegration.nuspec"
			      DestinationFiles="$(NuGetAutofacManifest)" />

        <FileUpdate Files="$(NuGetAutofacManifest)"
					    Regex="0.0.0.1"
					    ReplacementText="$(Major).$(Minor).$(Build).$(Revision)"/>

        <Copy SourceFiles="$(ContentSource)To_Enable_Autofac.txt" DestinationFolder="$(MainNuGetContent)" />
        <Copy SourceFiles="$(ContentSource)ApplicationModule.cs" DestinationFolder="$(MainNuGetContent)" />

        <ItemGroup>
          <AutofacContentFiles Include="$(MainNuGetContent)\**\*.*" />
        </ItemGroup>
        <FileUpdate Files="@(AutofacContentFiles)"
					    Regex="NuGetContentSource"
					    ReplacementText="$rootnamespace$"/>
        <Move SourceFiles="@(AutofacContentFiles)"
			      DestinationFiles="@(AutofacContentFiles->'$(MainNuGetContent)%(RecursiveDir)%(Filename)%(Extension).pp')" />

        <!--SL3-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC.Extensions.AutofacIntegration\bin\Build\WindowsPhoneMVC.Extensions.AutofacIntegration.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL3-WP\WindowsPhoneMVC.Extensions.AutofacIntegration.dll"/>
        <!--SL4-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC.Extensions.AutofacIntegration\bin\Build\WindowsPhoneMVC.Extensions.AutofacIntegration.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL4-WindowsPhone\WindowsPhoneMVC.Extensions.AutofacIntegration.dll"/>
        <!--SL4_71-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC.Extensions.AutofacIntegration\bin\Build_71\WindowsPhoneMVC.Extensions.AutofacIntegration.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL4-WindowsPhone71\WindowsPhoneMVC.Extensions.AutofacIntegration.dll"/>
        <Exec Command='"$(NuGet)" pack "$(NuGetAutofacManifest)" -BasePath "$(Root)build\temp\NuGet" -OutputDirectory "$(Root)build\artifacts"' />
        <RemoveDir Directories="build\temp\" />
      </Target>

      <Target Name="TransitionsNuGet">
        <PropertyGroup>
          <NuGetTransitionsManifest>$(Root)build\temp\WindowsPhoneMVC.Extensions.Transitions.nuspec</NuGetTransitionsManifest>
          <MainNuGetContent>$(Root)build\temp\NuGet\content\</MainNuGetContent>
        </PropertyGroup>

        <MakeDir Directories="$(Root)build\temp\NuGet\WindowsPhoneMVC.Extensions.Transitions\lib" />
        <Copy SourceFiles="$(Root)build\WindowsPhoneMVC.Extensions.Transitions.nuspec"
			      DestinationFiles="$(NuGetTransitionsManifest)" />

        <FileUpdate Files="$(NuGetTransitionsManifest)"
					    Regex="0.0.0.1"
					    ReplacementText="$(Major).$(Minor).$(Build).$(Revision)"/>

        <Copy SourceFiles="$(ContentSource)To_Enable_Transitions.txt" DestinationFolder="$(MainNuGetContent)" />

        <ItemGroup>
          <TransitionsContentFiles Include="$(MainNuGetContent)\**\*.*" />
        </ItemGroup>
        <FileUpdate Files="@(TransitionsContentFiles)"
					    Regex="NuGetContentSource"
					    ReplacementText="$rootnamespace$"/>
        <Move SourceFiles="@(TransitionsContentFiles)"
			      DestinationFiles="@(TransitionsContentFiles->'$(MainNuGetContent)\%(RecursiveDir)%(Filename)%(Extension).pp')" />

        <!--SL3-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC.Extensions.Transitions\bin\Build\WindowsPhoneMVC.Extensions.Transitions.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL3-WP\WindowsPhoneMVC.Extensions.Transitions.dll"/>
        <!--SL4-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC.Extensions.Transitions\bin\Build\WindowsPhoneMVC.Extensions.Transitions.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL4-WindowsPhone\WindowsPhoneMVC.Extensions.Transitions.dll"/>
        <!--SL4_71-->
        <Copy SourceFiles="$(Root)src\WindowsPhoneMVC.Extensions.Transitions\bin\Build_71\WindowsPhoneMVC.Extensions.Transitions.dll"
			      DestinationFiles="$(Root)build\temp\NuGet\lib\SL4-WindowsPhone71\WindowsPhoneMVC.Extensions.Transitions.dll"/>
        <Exec Command='"$(NuGet)" pack "$(NuGetTransitionsManifest)" -BasePath "$(Root)build\temp\NuGet" -OutputDirectory "$(Root)build\artifacts"' />
        <RemoveDir Directories="build\temp\" />
      </Target>

      <Target Name="Test" DependsOnTargets="NuGet">

      </Target>

      <Target Name="Package" DependsOnTargets="Test">
        <Copy SourceFiles="$(Root)src\Templates\WPMvcTemplatesExtension\bin\Build\WindowsPhoneMvcTemplatesExtension.vsix"
			      DestinationFiles="$(Root)build\artifacts\WindowsPhoneMvcTemplates.vsix" />
      </Target>
    </Project>
