---
layout: post
title: Collecting Hardware Performance Counters with BenchmarkDotNet
excerpt_separator: <!--more-->
---

## Introduction

BenchmarkDotNet is a powerful .NET library for benchmarking ([more about it](http://benchmarkdotnet.org/)). This post describes how you can collect [Hardware Performance Counters](https://en.wikipedia.org/wiki/Hardware_performance_counter) with BenchmarkDotNet. If you want to learn about the ETW internals behind it then you might find [this post](http://adamsitnik.com/Hardware-Counters-ETW/) useful.

## The Story

Some time ago I have noticed that software engineers who are working on the `Span<T>` are struggling with some benchmarking issues. With problems like unstable results, which BenchmarkDotNet solves. So I [asked](https://github.com/dotnet/corefxlab/pull/1278#issuecomment-284959960) why don't they use BenchmarkDotNet?

And then Krzysztof Cwalina [replied](https://github.com/dotnet/corefxlab/pull/1278#issuecomment-285082017):

> Yeah, we have. In general I love BenchmarkDotNet and use it in many cases. It does have better front end reporting capabilities and is more mature from reliability and statistical analysis perspective. But it seems like **BenchmarkDotNet does not support measuring instructions retired, cache misses, and branch mispredictions that we often find very useful**. Would be great to combine the efforts/features :-)
<!--more-->


And Tanner Gooding [mentioned](https://github.com/dotnet/corefxlab/pull/1278#issuecomment-285083415) that `xunit-performance` can do it:

> @KrzysztofCwalina, for Instructions Retired, you need [assembly: MeasureInstructionsRetired]. 
> 
> I also know cache misses and branch mispredictions are also supported, as they are displayed on the internal benchmark comparison site (I'm not sure how to enable those ones though).

I could finally add a new feature to BenchmarkDotNet that was related to performance, not the MSBuild/project.json stuff ;) So I just downloaded the source codes of [xunit-performance](https://github.com/Microsoft/xunit-performance) and [PerfView](https://github.com/microsoft/perfview) and dig into the details. `Microsoft.Diagnostics.Tracing.TraceEvent` is not OSS yet, so I used my favorite .NET decompiler [ILSpy](http://ilspy.net/) to get the full picture. 

## Limitations

What we have today comes with following limitations:

* Accuracy is limited by sampling frequency and additional code performed in the benchmarked process by our Engine. It's good but not perfect. For more accurate results you should use some profilers like Intel VTune Amplifier.
* dependency to `Microsoft.Diagnostics.Tracing.TraceEvent` [NuGet](https://www.nuget.org/packages/Microsoft.Diagnostics.Tracing.TraceEvent/) package (later on in this post I describe how to get it working for .NET Core/Mono on Windows)
* Windows 8+ only
* No Hyper-V (Virtualization) support
* Requires to run as Admin (ETW Kernel Session)
* No `InProcessToolchain` support ([#394](https://github.com/dotnet/BenchmarkDotNet/issues/394))
* The initial list of available Counters:
	* Timer,
	* TotalIssues,
	* BranchInstructions,
	* CacheMisses,
	* BranchMispredictions,
	* TotalCycles,
	* UnhaltedCoreCycles,
	* InstructionRetired,
	* UnhaltedReferenceCycles,
	* LlcReference,
	* LlcMisses,
	* BranchInstructionRetired,
	* BranchMispredictsRetired

## How to use it?

You need to install `BenchmarkDotNet.Diagnostics.Windows` package. The official 0.10.4 version should be released to nuget.org after 10th of April. As of today you can get the latest version from our CI feed by adding following line `<add key="appveyor-bdn" value="https://ci.appveyor.com/nuget/benchmarkdotnet" />` to your `NuGet.config` file. 

It can be enabled in two ways:

* Use the new attribute (apply it on a class that contains Benchmarks):

```cs
[HardwareCounters(HardwareCounter.BranchMispredictions, HardwareCounter.BranchInstructions)]
public class TheClassThatContainsBenchmarks { /* benchmarks go here */ }
```

* Tell your custom config to use it:

```cs
private class CustomConfig : ManualConfig
{
    public CustomConfig()
    {
        Add(Job.Default.WithHardwareCounters(HardwareCounter.BranchMispredictions, HardwareCounter.BranchInstructions);
    }
}
```


## Sample


Let's use our new diagnoser to answer the famous *"[Why is it faster to process a sorted array than an unsorted array?
](http://stackoverflow.com/questions/11227809/why-is-it-faster-to-process-a-sorted-array-than-an-unsorted-array)*".

```cs
class Program
{
    static void Main(string[] args) => BenchmarkRunner.Run<Cpu_BranchPerdictor>();
}

[HardwareCounters(HardwareCounter.BranchMispredictions, HardwareCounter.BranchInstructions)]
public class Cpu_BranchPerdictor
{
    private const int N = 32767;
    private readonly int[] sorted, unsorted;

    public Cpu_BranchPerdictor()
    {
        var random = new Random(0);
        unsorted = new int[N];
        sorted = new int[N];
        for (int i = 0; i < N; i++)
            sorted[i] = unsorted[i] = random.Next(256);
        Array.Sort(sorted);
    }

    private static int Branch(int[] data)
    {
        int sum = 0;
        for (int i = 0; i < N; i++)
            if (data[i] >= 128)
                sum += data[i];
        return sum;
    }

    private static int Branchless(int[] data)
    {
        int sum = 0;
        for (int i = 0; i < N; i++)
        {
            int t = (data[i] - 128) >> 31;
            sum += ~t & data[i];
        }
        return sum;
    }

    [Benchmark]
    public int SortedBranch() => Branch(sorted);

    [Benchmark]
    public int UnsortedBranch() => Branch(unsorted);

    [Benchmark]
    public int SortedBranchless() => Branchless(sorted);

    [Benchmark]
    public int UnsortedBranchless() => Branchless(unsorted);
}
```

and the results:

```
 |             Method |        Mean | Mispredict rate | BranchInstructions/Op | BranchMispredictions/Op |
 |------------------- |------------ |---------------- |---------------------- |------------------------ |
 |       SortedBranch |  21.4539 us |           0,04% |                 70121 |                      24 |
 |     UnsortedBranch | 136.1139 us |          23,70% |                 68788 |                   16301 |
 |   SortedBranchless |  28.6705 us |           0,06% |                 35711 |                      22 |
 | UnsortedBranchless |  28.9336 us |           0,05% |                 35578 |                      17 |
```

High Mispredict Rate (`23,70%`) is a clear signal that we are victims of branch prediction failure. Which results in `6x` slowdown!! 

**Note:** The results are scaled per operation. Mispredict rate is an extra column that is visible if you use `BranchInstructions` and `BranchMispredictions` counters.

## How to get it running for .NET Core/Mono on Windows?

In BenchmarkDotNet we create and compile a new project for every benchmark. The thing you run in the console is Host process, but each benchmark is executed as a separate process (child). `Microsoft.Diagnostics.Tracing.TraceEvent` does not support .NET Core yet ([issue](https://github.com/Microsoft/perfview/issues/165)), but it allows to gather data for any other process. So if your Host process is a classic desktop .NET Framework 4.6+ and you run your benchmarks for .NET Core/Mono the events can still be gathered by the Host. How to do it?

Your project file needs to support both frameworks:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFrameworks>netcoreapp1.1;net46</TargetFrameworks>
    <PlatformTarget>AnyCPU</PlatformTarget>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="BenchmarkDotNet" Version="0.10.3.78" />
  </ItemGroup>
  <ItemGroup Condition="'$(TargetFramework)' == 'net46'">
    <PackageReference Include="BenchmarkDotNet.Diagnostics.Windows" Version="0.10.3.78" />
  </ItemGroup>
</Project>
```

Your BenchmarkDotNet config must enable all the runtimes. You can do it by using the fluent configurator:

```cs
var config = ManualConfig.Create(DefaultConfig.Instance)
    .With(Job.Clr.WithHardwareCounters(HardwareCounter.BranchMispredictions, HardwareCounter.BranchInstructions))
    .With(Job.Core.WithHardwareCounters(HardwareCounter.BranchMispredictions, HardwareCounter.BranchInstructions));
            
BenchmarkRunner.Run<Cpu_BranchPerdictor>(config);
```

And then you just need to run the host as classic .NET: `dotnet run -c Release -f net46`

Sample results:


```
 |             Method | Runtime |        Mean | Mispredict rate | BranchInstructions/Op | BranchMispredictions/Op |
 |------------------- |-------- |------------ |---------------- |---------------------- |------------------------ |
 |       SortedBranch |     Clr |  21.3434 us |           0,03% |                 65452 |                      22 |
 |     UnsortedBranch |     Clr | 136.2859 us |          23,81% |                 73371 |                   17470 |
 |   SortedBranchless |     Clr |  28.7980 us |           0,05% |                 38253 |                      19 |
 | UnsortedBranchless |     Clr |  28.9019 us |           0,05% |                 33123 |                      15 |
 |       SortedBranch |    Core |  21.2255 us |           0,03% |                 82293 |                      28 |
 |     UnsortedBranch |    Core | 137.5567 us |          24,03% |                 67572 |                   16237 |
 |   SortedBranchless |    Core |  28.6537 us |           0,05% |                 35452 |                      16 |
 | UnsortedBranchless |    Core |  29.0826 us |           0,05% |                 41360 |                      19 |
```

## Sources

* [Understanding How General Exploration Works in Intel® VTune™ Amplifier XE](https://software.intel.com/en-us/articles/understanding-how-general-exploration-works-in-intel-vtune-amplifier-xe) by Jackson Marusarz (Intel)
* [CPU Performance Counters on Windows](https://randomascii.wordpress.com/2016/11/27/cpu-performance-counters-on-windows/) by Bruce Dawson
* [What is Branch Prediction?](http://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-an-unsorted-array/11227902) by [Mysticial](http://stackoverflow.com/users/922184/mysticial)
