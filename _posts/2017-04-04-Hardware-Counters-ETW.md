---
layout: post
title: Collecting Hardware Performance Counters with ETW
excerpt_separator: <!--more-->
---


Using the ETW to collect the PMC counters *is easy when you know how*. It took me some time to get it working so I am gonna describe it in case somebody wants to do it without  BenchmarkDotNet. The types used below come from the `Microsoft.Diagnostics.Tracing.TraceEvent` [NuGet](https://www.nuget.org/packages/Microsoft.Diagnostics.Tracing.TraceEvent/) package.

<!--more-->

First of all, you need to find out which Counters are available for your machine.

```cs
var infos = TraceEventProfileSources.GetInfo()
```

Or from command prompt:

```cmd
tracelog.exe -profilesources Help
```

You need to be aware of the fact that this feature is available only for Windows 8+. I couldn't resist sharing how the OS version check is implemented:

```cs
int num = Environment.OSVersion.Version.Major * 10 + Environment.OSVersion.Version.Minor;
if (num < 62)
	throw new ApplicationException("Profile source only availabe on Win8 and beyond.");
```

Counters are represented by following type:

```cs
public class ProfileSourceInfo
{
    /// Human readable name of the CPU performance counter (eg BranchInstructions, TotalIssues ...)
    public string Name;

    /// The ID that can be passed to SetProfileSources
    public int ID;

    /// This many events are skipped for each sample that is actually recorded
    public int Interval;

    /// The smallest Interval can be (typically 4K)
    public int MinInterval;

    /// The largest Interval can be (typically maxInt).
    public int MaxInterval;
}
```

Once you choose the counters you want to collect you need to set them up. The problem is that it's not clear how many counters you can use at the same time. For my machine, it's three. If I set up more, I get a silent error. The events are not being raised for the ones set up after first three. ;)

```cs
TraceEventProfileSources.Set(
    selectedCounters.Select(counter => counter.ID).ToArray(),
    selectedCounters.Select(counter => counter.MinInterval).ToArray());
```

Now you need to create a Kernel ETW Session:

```cs
var session = new TraceEventSession(KernelTraceEventParser.KernelSessionName);
```

The next thing is to enable the right Kernel Provider. You must be elevated (Admin) to use ETW Kernel Session. Only single kernel session can exist at the same time. If some session existed before, it will be restarted.

```cs
session.EnableKernelProvider(KernelTraceEventParser.Keywords.PMCProfile | KernelTraceEventParser.Keywords.Profile)
```

before you start processing events you should sign up for following two events:

```cs
session.Source.Kernel.PerfInfoCollectionStart += _ => { }; // we must subscribe to this event, otherwise the PerfInfoPMCSample is not raised ;)
session.Source.Kernel.PerfInfoPMCSample += OnPerfInfoPmcSample;
```

`PerfInfoCollectionStart` contains info about the interval (I am ignoring it because I set up it first so I know the value).
 
`PerfInfoPMCSample` is raised when a new sample is collected. It contains the id of the process and id of the counter, which should allow you to do the math (`counters[processId][counterId].SamplesCollected += Interval`). It also contains `InstructionPointer` which could be used to connect the sample with source code line. If I ever found few weeks of free time I would implement such feature for BenchmarkDotNet.


## Sources

* [xunit-performance](https://github.com/Microsoft/xunit-performance)
* [PerfView](https://github.com/microsoft/perfview)
* [Microsoft.Diagnostics.Tracing.TraceEvent](https://www.nuget.org/packages/Microsoft.Diagnostics.Tracing.TraceEvent/) + [ILSpy](http://ilspy.net/)