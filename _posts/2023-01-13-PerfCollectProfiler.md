---
layout: post
title: Profiling .NET on Linux with BenchmarkDotNet
excerpt_separator: <!--more-->
---

# PerfCollectProfiler

`PerfCollectProfiler` is a new BenchmarkDotNet diagnoser (plugin) that was released as part of [0.13.3](https://benchmarkdotnet.org/changelog/v0.13.3.html). It can profile the benchmarked .NET code on Linux and export the data to a trace file which can be opened with [PerfView](https://github.com/Microsoft/perfview), [speedscope](https://www.speedscope.app/) or any other tool that supports [perf](https://en.wikipedia.org/wiki/Perf_%28Linux%29) file format.

<!--more-->

## Demo

Following code is one of the official BenchmarkDotNet [samples](https://github.com/dotnet/BenchmarkDotNet/blob/12bf220e11fddc8e65b066eb1f300b63bfde7e9b/samples/BenchmarkDotNet.Samples/IntroPerfCollectProfiler.cs)

```cs
using System.IO;
using BenchmarkDotNet.Attributes;

namespace BenchmarkDotNet.Samples
{
    [PerfCollectProfiler(performExtraBenchmarksRun: false)]
    public class IntroPerfCollectProfiler
    {
        private readonly string path = Path.Combine(Path.GetTempPath(), Path.GetRandomFileName());
        private readonly string content = new string('a', 100_000);

        [Benchmark]
        public void WriteAllText() => File.WriteAllText(path, content);

        [GlobalCleanup]
        public void Delete() => File.Delete(path);
    }
}
```

The command:

```cmd
```

The regular output:

```
BenchmarkDotNet=v0.11.1.755-nightly, OS=Windows 10.0.17134.285 (1803/April2018Update/Redstone4)
Intel Xeon CPU E5-1650 v4 3.60GHz, 1 CPU, 12 logical and 6 physical cores
Frequency=3507505 Hz, Resolution=285.1029 ns, Timer=TSC
.NET Core SDK=2.2.100-preview2-009404
  [Host] : .NET Core 2.1.4 (CoreCLR 4.6.26814.03, CoreFX 4.6.26814.02), 64bit RyuJIT
  Dry    : .NET Core 2.1.4 (CoreCLR 4.6.26814.03, CoreFX 4.6.26814.02), 64bit RyuJIT

// * Diagnostic Output - EtwProfiler *
Exported 1 trace file(s). Example:
"C:\Projects\machinelearning\test\Microsoft.ML.Benchmarks\BenchmarkDotNet.Artifacts\Microsoft\ML\Benchmarks\RankingTrain\FastTree.etl"
```
<div class="scrollable-table-wrapper" markdown="block">

 |   Method |    Mean |    Error |   StdDev |
 |--------- |--------:|---------:|---------:|
 | FastTree | 32.48 s |  1.347 s | 0.0761 s |

</div>

And the new trace file opened with speedscope:

{: .center}
![speedscope](/images/perfcollectprofiler/speedscope.png)

## The Story

In the middle of 2019 I was working on improving the performance of `string` methods on Linux ([#24889](https://github.com/dotnet/coreclr/pull/24889), [#24973](https://github.com/dotnet/coreclr/pull/24973)). When I was done with the issues reported by the customers, it became clear to me that I need to get a better understanding of all Windows vs Unix .NET performance gaps. My goal was to fix the most important issues before the customers hit them. Thanks to previous investments in the performance culture of the .NET Team it was an easy job, as all I had to do was running all [dotnet/performance](https://github.com/dotnet/performance) micro-benchmarks on the same hardware for Windows and Unix and compare the results using [ResultsComparer](https://github.com/dotnet/performance/tree/main/src/tools/ResultsComparer) tool. To make it an apples-to-apples comparison I configured my work PC to [dual-boot](https://en.wikipedia.org/wiki/Multi-booting) Windows 10 and Ubuntu 18.04. I wanted to include macOS too, as many .NET users develop software on macOS. So I've used [Boot Camp](https://en.wikipedia.org/wiki/Boot_Camp_(software)) and installed Windows on my MacBook Pro. The comparison has identified multiple gaps: [#13628](https://github.com/dotnet/runtime/issues/13628), [#31268](https://github.com/dotnet/runtime/issues/31268), [#31269](https://github.com/dotnet/runtime/issues/31269), [#31270](https://github.com/dotnet/runtime/issues/31270), [#31271](https://github.com/dotnet/runtime/issues/31271), [#31273](https://github.com/dotnet/runtime/issues/31273), [#31275](https://github.com/dotnet/runtime/issues/31275), [#13669](https://github.com/dotnet/runtime/issues/13669), [#13675](https://github.com/dotnet/runtime/issues/13675), [#13676](https://github.com/dotnet/runtime/issues/13676), [#13684](https://github.com/dotnet/runtime/issues/13684) and [#31396](https://github.com/dotnet/runtime/issues/31396). 

Since there was plenty of them, I decided to automate the profiling. I've created a new BenchmarkDotNet branch and started working on a wrapper for [perfcollect](https://github.com/dotnet/runtime/blob/main/docs/project/linux-performance-tracing.md). perfcollect is a very powerful [bash script](https://github.com/microsoft/perfview/blob/main/src/perfcollect/perfcollect) that automates data collection. It's internally using perf and LTTng. All the credit for perfcollect goes to [Brian Robbins](https://github.com/brianrob), who authored this tool. perfcollect does all the heavy lifting, my **BenchmarkDotNet plugin is built upon on the work of Brian**.

I was not able to get it working quickly and I got stuck, so I pushed my changes and just switched to [VTune](https://github.com/dotnet/performance/blob/main/docs/profiling-workflow-dotnet-runtime.md#vtune). For this particular investigation, I've stopped using perfcollect as I did not like the fact that I had to copy the produced trace file to Windows to open it with PerfView. VTune provided me the answers I needed and I've moved one to some other work.

{: .center}
![VTune](/images/perfcollectprofiler/vtune.png)

I got back to working on it in 2020, but again with no success. In September 2022 together with [Jan Vorlicek](https://github.com/janvorli) we have started working on adding arm64 support to the [Disassembly Diagnoser](https://adamsitnik.com/Disassembly-Diagnoser/) (another topic for a blog post). We did that as a part of a one week long internal Microsoft Open Source hackathon. We made some great progress quicker than expected and still had some time left, so I've asked Jan for help with the perfcollect plugin (BTW Jan is one of the smartest people I've ever got to work with, at the same time being very humble and always eager to help). Jan has identified the source of my problems, I've changed the way of stopping the perfcollect process and got it working. The rest is history.

## How it works

`PerfCollectProfiler` uses [perfcollect](https://github.com/microsoft/perfview/blob/main/src/perfcollect/perfcollect) bash script for profiling and [dotnet symbol](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-symbol) for downloading symbols for the native libraries.

Before the process with benchmarked code is started, the plugin searches for perfcollect file stored in artifacts folder. If it's not present it means that the tool has not been installed yet. In such case, it loads the script file from library resources (the script is embeded in the `.dll` to ensure we are using a version that we tested), stores it one the disk, makes it an executable and invokes the install command (with `-force` option to avoid the need of user input for confirmation).

The next thing it does is identifying .NET SDK folder path and searching for missing native symbol files (`.so.dbg`). When some symbols are missing, it installs `dotnet symbol` tool in a dedicated folder (to always use latest version and avoid issues with broken existing configs) and commands it to recursively download symbols for all native libraries present in the SDK folder.

Once everything is in place, the diagnoser starts perfcollect process. perfcollect does all the heavy lifting (the creation of lttng sessions, perf tool usage etc). When BenchmarkDotNet starts the benchmarking process, it [sets all the necessary environment variables](https://github.com/dotnet/BenchmarkDotNet/blob/12bf220e11fddc8e65b066eb1f300b63bfde7e9b/src/BenchmarkDotNet/Extensions/ProcessExtensions.cs#L133-L142). By doing that, it ensures that **all** symbols will get solved and the trace file will be complete.

When benchmarking process quits, the plugin stops the perfcollect process by killing it with `SIGINT` signal. perfcollect works for a moment, exports the trace file and quits.

## Limitations

`PerfCollectProfiler` has following limitations:

* It supports only Linux. For Windows you can use [EtwProfiler](https://adamsitnik.com/ETW-Profiler/), for other Unixes like macOS the [EventPipeProfiler](https://wojciechnagorski.com/2020/04/cross-platform-profiling-.net-code-with-benchmarkdotnet/). EventPipeProfiler supports every OS, but it has no information about native methods.
* Requires to run as root. It's a PITA as all the files BDN creates directly and indirectly will be created by the root.
* No `InProcessToolchain` support (and no plans to add it).
* Currently the trace file contains no events, but [we are working on it](https://github.com/microsoft/perfview/issues/1718).

## How to use it?

You need to install `BenchmarkDotNet` 0.13.3 or newer.

It can be enabled via command line arguments (as long as you pass `args` to `BenchmarkSwitcher` or `BenchmarkRunner`). You won't need to recompile your code to use it:

```cmd
--profiler perf
```


Or you can extend the `DefaultConfig.Instance` with new instance of `PerfCollectProfiler` and the profiler will work for all benchmarks:

```cs
class Program
{
    static void Main(string[] args) 
        => BenchmarkSwitcher
            .FromAssembly(typeof(Program).Assembly)
            .Run(args,
                DefaultConfig.Instance
                    .With(PerfCollectProfiler.Default)); // HERE
}
```

Or you can apply the attribute, but it will work only for benchmarks from given `class`:

```cs
[PerfCollectProfiler]
public class TheClassThatContainsBenchmarks { /* benchmarks go here */ }
```


## Configuration

To configure the new diagnoser you need to create an instance of `PerfCollectProfilerConfig` class and pass it to the `PerfCollectProfiler` constructor. The parameters that config ctor accepts are:

* `performExtraBenchmarksRun`: when set to true, benchmarks will be executed one more time with the profiler attached. If set to false, there will be no extra run but the results will contain overhead. False by default, as I expect the overhead to be less than 3%.
* `timeoutInSeconds`: how long BenchmarkDotNet should wait for the perfcollect script to finish processing the trace. 300s by default.

The default config should be fine for 99% of users ;)

## Analyzing the trace files

There are multiple ways to work with the trace files produced by perfcollect:
* You can unzip the trace file, take the file produced by perf utility and open it with any tool that supports perf file format.
* You can copy them to Windows and open with [PerfView](https://learn.microsoft.com/en-us/shows/perfview-tutorial/).

By default, BenchmarkDotNet performs Warmup, Pilot and Overhead phases before starting the actual benchmark workload. I recommend to just filter the trace file to actual workload.

Here is how you can do it with speedscope:

{: .center}
![Nofilters](/images/etwprofiler/flamegraph_not_filtered.png)

If you are not familiar with speedscope you can read my [old blog post](https://adamsitnik.com/speedscope/#demo) about it. But to be honest the tool is so intutive that you don't really need to prepare yourself for using it.

### Special Thanks

I wanted to thank:

* Brian Robbins for authoring perfcollect and providing ongoing help.
* Jan Vorlicek for helping me with the investigation and unblocking me.