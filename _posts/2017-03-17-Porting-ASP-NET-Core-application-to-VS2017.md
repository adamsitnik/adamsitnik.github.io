---
layout: post
title: Porting ASP.NET Core application to VS 2017.
excerpt_separator: <!--more-->
---

## Introduction

It's been ten days since Visual Studio 2017 RTM was released. `.xproj` and `project.json` files are not going to be supported anymore so it's time to port our (ASP).NET Core projects to the new, simplified `.csproj` project file format. 

This post is not a complete guide on porting. I want to show you what problems I have faced and how I had solved them. I hope that somebody will find this useful. The examples are based on a ASP.NET Core application for trading energy that I develop at work. So it's something real, not "Hello World". I have also [ported](https://github.com/dotnet/BenchmarkDotNet/pull/357) BenchmarkDotNet to the new .csproj file format so it's a mixture of my personal experience.

<!--more-->

## Before you start

Nate McMaster, who is a developer on the ASP.NET team wrote two great blog posts about porting to MSBuild. If you want to have a good understanding of what you are doing you should read them first:

* [Project.json to MSBuild conversion guide](http://www.natemcmaster.com/blog/2017/01/19/project-json-to-csproj/)
* [Part 2 - Caveats of project.json to MSBuild conversion](http://www.natemcmaster.com/blog/2017/02/01/project-json-to-csproj-part2/)

## Use the tool

The first thing you need to do is to open your solution with new Visual Studio 2017 and approve the One-way upgrade. I recommend doing this from VS, not console (`dotnet migrate`). VS takes better care of all the MSBuild-related paths. So if your solution contains a lot of different projects (C#, F# and VB, .NET Core and classic .NET framework) you have higher chance for sucess. (BenchmarkDotNet contains all of them because we support all possible scenarios ;) )

{: .center}
![Visual Studio 2017 - One way upgrade approval](/images/porting/openWithVs2017.png)

Then you just wait for the `dotnet migrate` to do its job.

{: .center}
![Visual Studio 2017 - porting progress](/images/porting/progress.png)

After it's finished a Migration Reports is opened in the web browser. **Make sure you read the errors part.** Zero errors in the report does not mean that you are done with the porting ;)

{: .center}
![Migration Report](/images/porting/migrationReport.png)

You will notice that all `project.json` and `*.xproj` files are gone. Don't worry, they all have been saved in the `Backup` folder. Of course, the new `.csproj` files have been created. At this point of time you should commit all the changes. Later on you will see what was done by the tool, and what changes you have applied manually.

{: .center}
![Changes](/images/porting/changes.png)

## Verify the output

Software engineers should have limited trust in tools they use. So now it's time to compare your old `project.json` files with the new `.csproj` files. If you have read the two recommended posts you should already know what the mappings should be. Look for the missing parts, the less common or more complicated feature, the higher the chances that something is wrong. Once you find a difference you need to do the "manual" porting ;) I recommend you to do a separate commit per every "fix" so your team members can understand what you did.

For me the first thing I noticed was that paths to custom ruleset files for static code analysis were missing.

```json
"configurations": {
    "Release": {
      "buildOptions": {
        "additionalArguments": [ "/ruleset:../../StyleCop.Analyzers.ruleset" ],
      }
}
"dependencies": {
  "StyleCop.Analyzers": {
    "version": "1.0.0",
    "type": "build"
  }
}
```

was translated to:

```xml
<ItemGroup>
  <PackageReference Include="StyleCop.Analyzers" Version="1.0.0">
    <PrivateAssets>All</PrivateAssets>
  </PackageReference>
</ItemGroup>
```

In case you also use `StyleCop.Analyzers` with custom rules you need to fix it:

```xml
<ItemGroup>
  <PackageReference Include="StyleCop.Analyzers" Version="1.0.0" />
</ItemGroup>

<PropertyGroup Condition=" '$(Configuration)' == 'Release' ">
  <CodeAnalysisRuleSet>$(SolutionDir)\StyleCop.Analyzers.ruleset</CodeAnalysisRuleSet>
</PropertyGroup>
```

Please notice the usage of `$(SolutionDir)`. MSBuild has a [long list](http://stackoverflow.com/a/1453023) of available properties. By using them you can take advantage of all MSBuild features!

You might also discover that VS has created `runtimeconfig.template.json` file next to your ASP.NET Core app project file.

```json
{
  "gcServer": true,
  "gcConcurrent": true
}
```

For me the less the files the better, so I advise you to move these two flags to the `.csproj` of your app and remove the `runtimeconfig.template.json`.

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

Please also keep in mind that Concurrent Server GC mode which is the default mode for APS.NET Core apps has some drawbacks. It gives you a great performance, but not for free. It creates two dedicated GC threads per every logical core. So if you have a hundred of "Microservices" running on your 24 core server you end up with `100 x (24 x 2 + 1) = 4900` GC threads. `+1` is for single dedicated finalizer thread for every .NET process. Which option fits you the best? Just measure it ;)

## Build and Run

Now it's the time to build and run your app. But before you do so I recommend you to cleanup your solution directory to make sure that you perform a "clean" build. What works best for me:

``` cmd
// commit your changes, close VS
git clean -xfd
dotnet restore
// open VS
// build from VS
```

The first problem that I have encountered was `The debug profile 'web' is missing the path executable to debug`. The fact that I could not google it pushed me to blog so others don't waste their time.

{: .center}
![Unable to run](/images/porting/unableToRun.png)

It turned out that my `launchSettings.json` file did not get ported correctly. It was:

```json
"web": {
  "environmentVariables": {
    "ASPNETCORE_ENVIRONMENT": "Development"
  }
},
"web-production": {
  "environmentVariables": {
    "ASPNETCORE_ENVIRONMENT": "Production"
  }
```  

But should be:

```json
"web": {
  "commandName": "Project",
  "environmentVariables": {
    "ASPNETCORE_ENVIRONMENT": "Development"
  }
},
"web-production": {
  "commandName": "Project",
  "environmentVariables": {
    "ASPNETCORE_ENVIRONMENT": "Production"
  }
```

Which got me one step closer: 

`System.IO.FileLoadException: 'Could not load file or assembly 'Microsoft.Win32.Primitives, Version=4.0.0.0` 

{: .center}
![Unable to run](/images/porting/unhandledException.png)

Which I fixed with help of [Stack Overflow user Cody](http://stackoverflow.com/a/42315239) by installing `Microsoft.Win32.Primitives 4.0.0.0` NuGet package. This particular app is using ASP.NET Core and Kestrel, but runs on `net46`. Those of you who target `netcoreapp` might not encounter the same problem.

Now I was able to compile, run and debug our app.

As of today ASP.NET Core app that targets classic .NET is `x86` by default. You can enforce `x64` by applying: `<PlatformTarget>x64</PlatformTarget>` in your `.csproj`.

## Versioning and Deployment

We used to have some PowerShell script that was setting version for every `project.json` file in our repository. With `.csproj` you don't need to write your own scripts for doing this. MSBuild is a mature platform, this problem has already been solved.

If you want to use your build number that is passed by your CI to every process it spawns you can do it with MSBuild by reading the environment variable.

```xml
<BuildNumber Condition=" '$(APPVEYOR_BUILD_NUMBER)' != '' ">$(APPVEYOR_BUILD_NUMBER)</BuildNumber> <!-- for AppVeyor -->
<BuildNumber Condition=" '$(BUILD_NUMBER)' != '' ">$(BUILD_NUMBER)</BuildNumber> <!-- for Team City -->
<BuildNumber Condition=" '$(BuildNumber)' == '' ">0</BuildNumber> <!-- if not set -->
```

then you can set whatever suits you best by using the combination of MSBuild properties, features, and variables. Sample:

```xml
<AssemblyVersion>$(BuildNumber)</AssemblyVersion>
<AssemblyFileVersion>$(BuildNumber)</AssemblyFileVersion>
<InformationalVersion>$(BuildNumber)</InformationalVersion>
<PackageVersion>$(BuildNumber)</PackageVersion>
```

To build a package you still need to call `dotnet publish`. However the way of handling relative path for `--output` option has changed. Whatever you pass to it is now relative to the `.csproj` file, not the working directory (as it was before).

One important thing about CI: if you install the new .NET Core SDK it will became the default one. It means that all projects, that still use `project.json` file will fail to compile. It's bad for other projects that are compiled on the same CI machine. You can avoid this problem by setting the sdk version for old projects it the corresponding `global.json` file:

```json
{
  "sdk": { "version": "1.0.0-preview2-1-003177" }
}
```


## Refactoring

MSBuilds allows you to include multiple `.props` files in your `.csproj` files. Props file can contain most of the `.csproj` settings. I recommend you to move all your common settings that are the same for every project to a `.props` file and simply include it in your `.csproj` files.

Sample common settings of [Kestrel project](https://github.com/aspnet/KestrelHttpServer/blob/dev/build/common.props):

```xml
<Project>
  <Import Project="dependencies.props" />
  <Import Project="..\version.props" />

  <PropertyGroup>
    <Product>Microsoft ASP.NET Core</Product>
    <RepositoryUrl>https://github.com/aspnet/KestrelHttpServer</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
    <AssemblyOriginatorKeyFile>$(MSBuildThisFileDirectory)Key.snk</AssemblyOriginatorKeyFile>
    <SignAssembly>true</SignAssembly>
    <PublicSign Condition="'$(OS)' != 'Windows_NT'">true</PublicSign>
    <VersionSuffix Condition="'$(VersionSuffix)'!='' AND '$(BuildNumber)' != ''">$(VersionSuffix)-$(BuildNumber)</VersionSuffix>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Internal.AspNetCore.Sdk" Version="1.0.1-*" PrivateAssets="All" />
  </ItemGroup>

  <ItemGroup Condition="'$(TargetFrameworkIdentifier)'=='.NETFramework' AND '$(OutputType)'=='library'">
    <PackageReference Include="NETStandard.Library" Version="$(NetStandardImplicitPackageVersion)" />
  </ItemGroup>
</Project>
```

And then in your `.csproj`:
```xml
<Import Project="..\..\settings\common.props" />
```

In our solution I have moved all things related to versioning to single `.props` file, common settings to another `.props` file. Then included these in `.csprojs` and removed all `AssemblyInfo.cs` files. In my case I just don't need them anymore. And my new`.csproj` files are now smaller than my old, beloved `project.json` files!

##### Did you like this blog post? Leave a comment so I continue blogging about my ASP.NET Core experience.
