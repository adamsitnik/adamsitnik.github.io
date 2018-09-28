---
layout: post
title: Profiling .NET Code with BenchmarkDotNet
excerpt_separator: <!--more-->
---

# ETW Profiler

EtwProfiler is the new diagnoser for BenchmarkDotNet that I have just finished. It's going to be released as part of `0.11.2`. Soon! It allows to profile the benchmarked .NET code on Windows and exports the data to a trace file which can be opened with [PerfView](https://github.com/Microsoft/perfview) or [Windows Performance Analyzer](https://docs.microsoft.com/en-us/windows-hardware/test/wpt/windows-performance-analyzer).

**Again with a single config!**
<!--more-->

## Demo

Following code is a real-world benchmark from the [ML.NET repository](https://github.com/dotnet/machinelearning/blob/17ee205e585beb62777475af6d59cba816675eeb/test/Microsoft.ML.Benchmarks/Numeric/Ranking.cs#L36)

```cs
[EtwProfiler(performExtraBenchmarksRun: false)] // !!! use the new diagnoser!!
public class RankingTrain
{
    private string _mslrWeb10k_Validate;
    private string _mslrWeb10k_Train;

    [GlobalSetup]
    public void Setup()
    {
        _mslrWeb10k_Validate = Path.GetFullPath(TestDatasets.MSLRWeb.validFilename);
        _mslrWeb10k_Train = Path.GetFullPath(TestDatasets.MSLRWeb.trainFilename);
    }

    [Benchmark]
    public void FastTree()
    {
        string cmd = @"TrainTest test=" + _mslrWeb10k_Validate +
            " eval=RankingEvaluator{t=10}" +
            " data=" + _mslrWeb10k_Train +
            " loader=TextLoader{col=Label:R4:0 col=GroupId:TX:1 col=Features:R4:2-138}" +
            " xf=HashTransform{col=GroupId} xf=NAHandleTransform{col=Features}" +
            " tr=FastTreeRanking{}";

        using (var environment = EnvironmentFactory.CreateRankingEnvironment<RankerEvaluator, TextLoader, HashTransformer, FastTreeRankingTrainer>())
        {
            Maml.MainCore(environment, cmd, alwaysPrintStacktrace: false);
        }
    }
}
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

And the new trace file opened with PerfView:

{: .center}
![Flamegraph](/images/etwprofiler/flamegraph.png)

## The Story

Recently I have been working on porting all of the 3 000+ CoreFX and CoreCLR benchmarks from [xunit-performance](https://github.com/Microsoft/xunit-performance) to BenchmarkDotNet. My job was to port all of the benchmarks, compare the results, fix the bugs and last but not least implement missing features. EtwProfiler is one of the things that were present in xunit-performance, but not in BenchmarkDotNet.

Initially I was sceptical about this idea because profiling running benchmark is an easy job, however with the amount of benchmarks we have, automating it was a must have.

And now I am very happy about the outcome!

## How it works

`EtwProfiler` uses `TraceEvent` library which internally uses Event Tracing for Windows (ETW) to capture stack traces and important .NET Runtime events.

Before the process with benchmarked code is started, EtwProfiler starts User and Kernel ETW sessions. Every session writes data to it's own file and captures different data. User session listens for the .NET Runtime events (GC, JIT etc) while the Kernel session gets CPU stacks and Hardware Counter events. After this, the process with benchmarked code is started. During the benchmark execution all the data is captured and written to a trace file. Moreover, BenchmarkDotNet Engine emits it's own events to be able to differentiate jitting, warmup, pilot and actual workload when analyzing the trace file. When the benchmarking is over, both sessions are closed and the two trace files are merged into one.

Stopping the sessions after process exit was very important because CLR emits all the symbol information as part of the CLR Rundown.

## Limitations

What we have today comes with following limitations:

* EtwProfiler works only on Windows (one day I might implement similar thing for Unix using EventPipe)
* Requires to run as Admin (to create ETW Kernel Session)
* No `InProcessToolchain` support 
* To get the best possible managed code symbols you should configure your project in following way:

```xml
<DebugType>pdbonly</DebugType>
<DebugSymbols>true</DebugSymbols>
```

## How to use it?

You need to install `BenchmarkDotNet.Diagnostics.Windows` package. The official `0.11.2` version should be released to nuget.org in October. If you can't wait and want to give it a try today you need to download `0.11.1.755` preview package from our CI feed by adding following line `<add key="appveyor-bdn" value="https://ci.appveyor.com/nuget/benchmarkdotnet" />` to your `NuGet.config` file (if you don't have such file you can generate if by running `dotnet new nugetconfig` command). 

It can be enabled in few ways, some of them:

* Use the new attribute (apply it on a class that contains Benchmarks):

```cs
[EtwProfiler]
public class TheClassThatContainsBenchmarks { /* benchmarks go here */ }
```

* Extend the `DefaultConfig.Instance` with new instance of `EtwProfiler`:

```cs
class Program
{
    static void Main(string[] args) 
        => BenchmarkSwitcher
            .FromAssembly(typeof(Program).Assembly)
            .Run(args,
                DefaultConfig.Instance
                    .With(new EtwProfiler())); // HERE
}
```

* Passing `-p` or `--profile` command line argument to `BenchmarkSwitcher`


## Configuration

To configure the new diagnoser you need to create an instance of `EtwProfilerConfig` class and pass it to the `EtwProfiler` constructor. The parameters that `EtwProfilerConfig` ctor takes are:

* `performExtraBenchmarksRun` - if set to true, benchmarks will be executed one more time with the profiler attached. If set to false, there will be no extra run but the results will contain overhead. True by default.
* `bufferSizeInMb` - ETW session buffer size, in MB. 256 by default.
* `intervalSelectors` - interval per harwdare counter, if not provided then default values will be used.
* `kernelKeywords` - kernel session keywords, ImageLoad (for native stack frames) and Profile (for CPU Stacks) are the defaults.
* `providers` - providers that should be enabled, if not provided then default values will be used.

## Using PerfView to work with trace files

PerfView is a free .NET profiler from Microsoft. If you don't know how to use it you should watch [these instructional videos](https://channel9.msdn.com/Series/PerfView-Tutorial) first.

If you are familiar with PerfView, then the only thing you need to know is that BenchmarkDotNet performs Jitting by running the code once, Pilot Experiment to determine how many times benchmark should be executed per iteration, non-trivial Warmup and Actual Workload. This is why when you open your trace file in PerfView you will see your benchmark in a few different places of the StackTrace.

{: .center}
![Nofilters](/images/etwprofiler/flamegraph_not_filtered.png)

The simplest way to filter the data to the actual benchmarks runs is to open the `CallTree` tab, put "EngineActualStage" in the Find box, press enter and when PerfView selects `EngineActualStage` in the `CallTree` press `Alt+R` to Set Time Range.

{: .center}
![Filter](/images/etwprofiler/perfview.gif)

If you want to filter the trace to single iteration, then you must go to the Events panel and search for the `WorkloadActual/Start` and `WorkloadActual/Stop` events.

1. Open Events window
2. Put "WorkloadActual" in the Filter box and hit enter.
3. Press control or shift and choose the Start and Stop events from the left panel. Hit enter.
4. Choose iteration that you want to investigate (events are sorted by time).
5. Select two or more cells from the "Time MSec" column.
6. Right click, choose "Open Cpu Stacks".
7. Choose the process with benchmarks, right-click, choose "Drill Into"

{: .center}
![Filter](/images/etwprofiler/perfview_events.gif)

### Special Thanks

I wanted to thank:

* Jose Rivero who implemented this feature for xunit-performance and reviewed my code. I took a lot from his code.
* Brian Robbins for explaining me how CLR Rundown works.
* Vance Morrison for immediate release of TraceEvent with bug fixes in the area that was touching private Windows APIs.
* Andrey Akinshin for reviewing the PR and pushing me to write the docs. Without Andrey I would not write this blog post ;)