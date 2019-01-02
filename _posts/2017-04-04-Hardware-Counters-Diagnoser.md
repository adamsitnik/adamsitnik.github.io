---
layout: post
title: Collecting Hardware Performance Counters with BenchmarkDotNet
excerpt_separator: <!--more-->
---

## Introduction

BenchmarkDotNet is a powerful .NET library for benchmarking ([more about it](https://benchmarkdotnet.org/)). This post describes how you can collect [Hardware Performance Counters](https://en.wikipedia.org/wiki/Hardware_performance_counter) with BenchmarkDotNet. If you want to learn about the ETW internals behind it then you might find [this post](https://adamsitnik.com/Hardware-Counters-ETW/) useful.

## The Story

Some time ago I have noticed that software engineers who are working on the `Span<T>` are struggling with some benchmarking issues. With problems like unstable results, which BenchmarkDotNet solves. So I [asked](https://github.com/dotnet/corefxlab/pull/1278#issuecomment-284959960) why don't they use BenchmarkDotNet?

And then Krzysztof Cwalina [replied](https://github.com/dotnet/corefxlab/pull/1278#issuecomment-285082017):

> Yeah, we have. In general I love BenchmarkDotNet and use it in many cases. It does have better front end reporting capabilities and is more mature from reliability and statistical analysis perspective. But it seems like **BenchmarkDotNet does not support measuring instructions retired, cache misses, and branch mispredictions that we often find very useful**. Would be great to combine the efforts/features :-)
<!--more-->


And Tanner Gooding [mentioned](https://github.com/dotnet/corefxlab/pull/1278#issuecomment-285083415) that `xunit-performance` can do it:

> @KrzysztofCwalina, for Instructions Retired, you need [assembly: MeasureInstructionsRetired]. 
> 
> I also know cache misses and branch mispredictions are also supported, as they are displayed on the internal benchmark comparison site (I'm not sure how to enable those ones though).

I could finally add a new feature to BenchmarkDotNet that was related to performance, not the MSBuild/project.json stuff ;) So I just downloaded the source codes of [xunit-performance](https://github.com/Microsoft/xunit-performance) and [PerfView](https://github.com/microsoft/perfview) and dig into the details. `Microsoft.Diagnostics.Tracing.TraceEvent` is not OSS yet, so I used my favorite .NET decompiler [ILSpy](https://ilspy.net/) to get the full picture. 

## Limitations

What we have today comes with following limitations:

* Accuracy is limited by sampling frequency and additional code performed in the benchmarked process by our Engine. It's good but not perfect. For more accurate results you should use some profilers like Intel VTune Amplifier.
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

You need to install latest `BenchmarkDotNet.Diagnostics.Windows` package.

It can be enabled in three ways:

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
        Add(Job.Default);
        Add(HardwareCounter.BranchMispredictions, HardwareCounter.BranchInstructions);
    }
}
```

* Passing `--counters` command line argument to `BenchmarkSwitcher`. Example: `--counters BranchInstructions+BranchMispredictions`


## Sample


Let's use our new diagnoser to answer the famous *"[Why is it faster to process a sorted array than an unsorted array?
](https://stackoverflow.com/questions/11227809/why-is-it-faster-to-process-a-sorted-array-than-an-unsorted-array)*".

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

 |             Method |        Mean | Mispredict rate | BranchInstructions/Op | BranchMispredictions/Op |
 |------------------- |------------ |---------------- |---------------------- |------------------------ |
 |       SortedBranch |  21.4539 us |           0,04% |                 70121 |                      24 |
 |     UnsortedBranch | 136.1139 us |          **23,70%** |                 68788 |                   16301 |
 |   SortedBranchless |  28.6705 us |           0,06% |                 35711 |                      22 |
 | UnsortedBranchless |  28.9336 us |           0,05% |                 35578 |                      17 |

High Mispredict Rate (`23,70%`) is a clear signal that we are victims of branch prediction failure. Which results in `6x` slowdown!! 

**Note:** The results are scaled per operation. Mispredict rate is an extra column that is visible if you use `BranchInstructions` and `BranchMispredictions` counters.

## Sources

* [Understanding How General Exploration Works in Intel® VTune™ Amplifier XE](https://software.intel.com/en-us/articles/understanding-how-general-exploration-works-in-intel-vtune-amplifier-xe) by Jackson Marusarz (Intel)
* [CPU Performance Counters on Windows](https://randomascii.wordpress.com/2016/11/27/cpu-performance-counters-on-windows/) by Bruce Dawson
* [What is Branch Prediction?](https://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-an-unsorted-array/11227902) by [Mysticial](https://stackoverflow.com/users/922184/mysticial)
