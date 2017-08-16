---
layout: post
title: Disassembling .NET Code with BenchmarkDotNet
excerpt_separator: <!--more-->
---

# Disassembly Diagnoser

Disassembly Diagnoser is the new diagnoser for BenchmarkDotNet that I have just finished. It's in preview now, going to be released as part of `0.10.10`. It allows to disassemble the benchmarked .NET code:

* to ASM:
	* desktop .NET: LegacyJit (32 & 64 bit), RyuJIT (64 bit)
	* .NET Core 1.1+ (**including .NET Core 2.0**) for RyuJIT (64 bit)
    * Mono: 32 & 64 bit, **including LLVM**
* to IL and corresponding C# code:    
	* desktop .NET: LegacyJit (32 & 64 bit), RyuJIT (64 bit)
	* .NET Core: 1.1+ (including .NET Core 2.0)

**With a single config!**
<!--more-->

## Demo

```cs
[DisassemblyDiagnoser(printAsm: true, printSource: true)] // !!! use the new diagnoser!!
[RyuJitX64Job]
public class Simple
{
    int[] field = Enumerable.Range(0, 100).ToArray();

    [Benchmark]
    public int SumLocal()
    {
        var local = field; // we use local variable that points to the field

        int sum = 0;
        for (int i = 0; i < local.Length; i++)
            sum += local[i];

        return sum;
    }

    [Benchmark]
    public int SumField()
    {
        int sum = 0;
        for (int i = 0; i < field.Length; i++)
            sum += field[i];

        return sum;
    }
}
```

The regular output:

```
BenchmarkDotNet=v0.10.9.281-nightly, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i7-4700MQ CPU 2.40GHz (Haswell), ProcessorCount=8
Frequency=2338344 Hz, Resolution=427.6531 ns, Timer=TSC
  [Host]    : .NET Framework 4.7 (CLR 4.0.30319.42000), 64bit RyuJIT-v4.7.2053.0
  RyuJitX64 : .NET Framework 4.7 (CLR 4.0.30319.42000), 64bit RyuJIT-v4.7.2053.0

Job=RyuJitX64  Jit=RyuJit  Platform=X64  
```
<div class="scrollable-table-wrapper" markdown="block">

 |   Method |     Mean |     Error |    StdDev |
 |--------- |---------:|----------:|----------:|
 | SumLocal | **78.27** ns | 0.6818 ns | 0.6377 ns |
 | SumField | 79.24 ns | 0.3923 ns | 0.3670 ns |

</div>

And the new disassembly output:

{: .center}
![Disassembly](/images/disasm/simpleDemo.png)

As you can see very similar C# code produces different assembly code which has different performance characteristics. The main goal of disassembly diagnoser is to allow the BenchmarkDotNet users to do an easy comparison of generated assembly code.

## The Story

I wanted to develop this feature for a long time. Many people asked for it but I simply did not have any spare time. This was about to change.

I was getting back from an awesome [.NET Meetup](https://www.wug.cz/praha/akce/951--Net-TechTalks) organized by Karel Zikmund in Prague and I had few spare hours between the checkout from the hotel and my flight. I decided to write a simple PoC and see how it goes.

Initially, I had no idea where to start. Most of the people use the Disassembly window from Visual Studio. But VS is closed-source so I could only use it for validation of my results. The other option was WinDbg. Getting the disassembly with WinDb is [non-trivial](https://bret.codes/net-core-and-windbg/). And it's closed-source as well.

I have almost forgotten that Matt Warren did something [very similar](https://github.com/dotnet/BenchmarkDotNet/issues/53) to this for BenchmarkDotNet a long time ago. I have started analysing his code, which [led](https://github.com/Microsoft/clrmd/issues/34#issuecomment-256304015) me to [msos](https://github.com/goldshtn/msos) by Sasha Goldshtein. And msos by Sasha was exactly what I needed. The credit for super smart disassembling goes to Sasha. I just took his code, tweaked it a little and extended. I have  found and fixed some bugs in [msos](https://github.com/goldshtn/msos/pull/71) and [ClrMD](https://github.com/Microsoft/clrmd/pull/83). So all sides benefit from being OSS ;)

# How it works?

As some of you might know in BenchmarkDotNet we have the host process (what you run in the console) and child process, which executes the benchmark and reports results back to the host. The child process is generated, compiled and executed by the host. With such architecture we can benchmark given .NET code for any config desired by the user (any JIT, any .NET framework, any GC configuration). Last but not least it helps to make the results more stable. GC is self tuning and JIT can make some extra optimizations, but with process per benchmark you always get clean score.

## Desktop .NET

Based on the idea from msos the host is using ClrMD to attach to the child process. ClrMD allows us to get the text representation of assembly code. To get the IL we use the one and only Mono.Cecil. To get the corresponding C# code we once again use ClrMD.

ClrMD can attach to the process of the same bitness. To support all scenarios (host 32bit, child 64bit and the opposite) I have put the disassembler to a separate process. This is why we have `BenchmarkDotNet.Disassembler.x86.exe` and `BenchmarkDotNet.Disassembler.x64.exe`. Both disassemblers are stored in the resources of the `BenchmarkDotNet.Diagnostics.Windows.dll`. When the time comes, they are copied from resources to the hard drive and executed accordingly.

## .NET Core

The NuGet package of [ClrMD](https://www.nuget.org/packages/Microsoft.Diagnostics.Runtime/) implements .NET Core support, but targets only desktop .NET. It's not a problem, because we can use our architecture to get it running for .NET Core. If host is a desktop .NET process it can use ClrMD to attach to the child .NET Core process.

This is why if you want to get it running for .NET Core you have to target both classic .NET and .NET Core frameworks.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFrameworks>netcoreapp1.1;net46</TargetFrameworks>
    <PlatformTarget>AnyCPU</PlatformTarget>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="BenchmarkDotNet" Version="0.10.9.281" />
  </ItemGroup>
  <ItemGroup Condition="'$(TargetFramework)' == 'net46'">
    <PackageReference Include="BenchmarkDotNet.Diagnostics.Windows" Version="0.10.9.281" />
  </ItemGroup>
</Project>
```

```cs
[DisassemblyDiagnoser] // !!! use the new diagnoser!!
[CoreJob] // tell the Host to run the benchmarks for .NET Core
public class TheClassThatContainsBenchmarks { }
```

And run the host as desktop .NET:

```
dotnet run -f net46 -c Release
```

## Mono

With great help from Miguel de Icaza I was able to implement a simple disassembler for Mono. We just run:

```
mono -v -v -v -v --compile $namespace.$typeName:$methodName $exeName
```

and parse the output. The LLVM is supported and you don't need to install anything except BenchmarkDotNet. The downside is that as of now the parser can handle only simple benchmarks. I did not have the time to test all edge cases.

## Limitations

What we have today comes with following limitations:

* .NET Core disassembler works only on Windows
* Mono disassembler does not support recursive disassembling and produces output without IL and C#.
* Indirect calls are not tracked.
* To get the corresponding C#/F# code from disassembler you need to configure your project in following way:

```xml
<DebugType>pdbonly</DebugType>
<DebugSymbols>true</DebugSymbols>
```


# How to use it?

You need to install `BenchmarkDotNet.Diagnostics.Windows` package. The official 0.10.10 version should be released to nuget.org very soon. As of today you can get the latest version from our CI feed by adding following line `<add key="appveyor-bdn" value="https://ci.appveyor.com/nuget/benchmarkdotnet" />` to your `NuGet.config` file. 

It can be enabled in two ways:

* Use the new attribute (apply it on a class that contains Benchmarks):

```cs
[DisassemblyDiagnoser(printAsm: true, printSource: true)]
public class TheClassThatContainsBenchmarks { /* benchmarks go here */ }
```

* Tell your custom config to use it:

```cs
private class CustomConfig : ManualConfig
{
    public CustomConfig()
    {
        Add(Job.Default);
        Add(DisassemblyDiagnoser.Create(new DisassemblyDiagnoserConfig(printAsm: true, recursiveDepth: 1)));
    }
}
```

## Single config for ALL JITs

You can use a single config to compare the generated assembly code for ALL JITs. 

Let's check the Devirtualization that was [introduced recently](https://blogs.msdn.microsoft.com/dotnet/2017/06/29/performance-improvements-in-ryujit-in-net-core-and-net-framework/) for .NET Core 2.0:

```cs
public class MultipleJits : ManualConfig
{
    public MultipleJits()
    {
        Add(Job.ShortRun.With(new MonoRuntime(name: "Mono x86", customPath: @"C:\Program Files (x86)\Mono\bin\mono.exe")).With(Platform.X86));
        Add(Job.ShortRun.With(new MonoRuntime(name: "Mono x64", customPath: @"C:\Program Files\Mono\bin\mono.exe")).With(Platform.X64));

        Add(Job.ShortRun.With(Jit.LegacyJit).With(Platform.X86).With(Runtime.Clr));
        Add(Job.ShortRun.With(Jit.LegacyJit).With(Platform.X64).With(Runtime.Clr));

        Add(Job.ShortRun.With(Jit.RyuJit).With(Platform.X64).With(Runtime.Clr));

        // RyuJit for .NET Core 1.1
        Add(Job.ShortRun.With(Jit.RyuJit).With(Platform.X64).With(Runtime.Core).With(CsProjCoreToolchain.NetCoreApp11));

        // RyuJit for .NET Core 2.0
        Add(Job.ShortRun.With(Jit.RyuJit).With(Platform.X64).With(Runtime.Core).With(CsProjCoreToolchain.NetCoreApp20));

        Add(DisassemblyDiagnoser.Create(new DisassemblyDiagnoserConfig(printAsm: true, printPrologAndEpilog: true, recursiveDepth: 3)));
    }
}

[Config(typeof(MultipleJits))]
public class Jit_Devirtualization
{
    private Increment increment = new Increment();

    [Benchmark]
    public int CallVirtualMethod() => increment.OperateTwice(10);

    public abstract class Operation  // abstract unary integer operation
    {
        public abstract int Operate(int input);

        public int OperateTwice(int input) => Operate(Operate(input)); // two virtual calls to Operate
    }

    public sealed class Increment : Operation // concrete, sealed operation: increment by fixed amount
    {
        public readonly int Amount;
        public Increment(int amount = 1) { Amount = amount; }

        public override int Operate(int input) => input + Amount;
    }
}
```

The results:

```
BenchmarkDotNet=v0.10.9.281-nightly, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i7-4700MQ CPU 2.40GHz (Haswell), ProcessorCount=8
Frequency=2338344 Hz, Resolution=427.6531 ns, Timer=TSC
  [Host]     : .NET Framework 4.7 (CLR 4.0.30319.42000), 64bit RyuJIT-v4.7.2053.0
  Job-UBMWVM : .NET Framework 4.7 (CLR 4.0.30319.42000), 64bit LegacyJIT/clrjit-v4.7.2053.0;compatjit-v4.7.2053.0
  Job-JDGXXX : .NET Framework 4.7 (CLR 4.0.30319.42000), 32bit LegacyJIT-v4.7.2053.0
  Job-PXPXXE : .NET Framework 4.7 (CLR 4.0.30319.42000), 64bit RyuJIT-v4.7.2053.0
  Job-DULNTX : .NET Core 1.1.2 (Framework 4.6.25211.01), 64bit RyuJIT
  Job-GAPDXO : .NET Core 2.0.0 (Framework 4.6.00001.0), 64bit RyuJIT
  Job-ZXJTYF : Mono 4.4.1 (Visual Studio), 64bit 
  Job-NBVNXQ : Mono 5.2.0 (Visual Studio), 32bit 

LaunchCount=1  TargetCount=3  WarmupCount=3  
```
<div class="scrollable-table-wrapper" markdown="block">

 |            Method |       Jit | Platform |  Runtime |     Toolchain |     Mean |     Error |    StdDev |
 |------------------ |---------- |--------- |--------- |-------------- |---------:|----------:|----------:|
 | CallVirtualMethod | LegacyJit |      X64 |      Clr |       Default | 3.222 ns | 0.2984 ns | 0.0169 ns |
 | CallVirtualMethod | LegacyJit |      X86 |      Clr |       Default | 3.012 ns | 0.3651 ns | 0.0206 ns |
 | CallVirtualMethod |    RyuJit |      X64 |      Clr |       Default | 2.928 ns | 0.2941 ns | 0.0166 ns |
 | CallVirtualMethod |    RyuJit |      X64 |     Core | .NET Core 1.1 | 2.920 ns | 0.1688 ns | 0.0095 ns |
 | CallVirtualMethod |    RyuJit |      X64 |     Core | .NET Core 2.0 | **2.222** ns | 0.6163 ns | 0.0348 ns |
 | CallVirtualMethod |    RyuJit |      X64 | Mono x64 |       Default | 5.114 ns | 0.5626 ns | 0.0318 ns |
 | CallVirtualMethod |    RyuJit |      X86 | Mono x86 |       Default | 9.610 ns | 0.2672 ns | 0.0151 ns |

</div>

The disassembly result can be obtained [here](http://adamsitnik.com/files/disasm/Jit_Devirtualization-disassembly-report.html). The file was too big to put it here.

# The Ultimate Combination


Some time ago I have implemented [Hardware Counters](http://adamsitnik.com/Hardware-Counters-Diagnoser/) diagnoser for BenchmarkDotNet. Ever since then I wanted to combine the Instruction Pointers that comes with the events with the code.

Now it was finally possible. ClrMD gives me the asm with IPs, ETW gives me hardware counters with IPs. That's all I need.

Let's use both diagnosers to answer the famous *"[Why is it faster to process a sorted array than an unsorted array?
](http://stackoverflow.com/questions/11227809/why-is-it-faster-to-process-a-sorted-array-than-an-unsorted-array)*".

```cs
class Program
{
    static void Main(string[] args) => BenchmarkRunner.Run<Cpu_BranchPerdictor>();
}

[HardwareCounters(HardwareCounter.BranchMispredictions, HardwareCounter.BranchInstructions)]
[DisassemblyDiagnoser(printAsm: true, printSource: true)]
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

    [Benchmark]
    public int SortedBranch() => Branch(sorted);

    [Benchmark]
    public int UnsortedBranch() => Branch(unsorted);
}
```

The results:

```
BenchmarkDotNet=v0.10.9.281-nightly, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i7-4700MQ CPU 2.40GHz (Haswell), ProcessorCount=8
Frequency=2338344 Hz, Resolution=427.6531 ns, Timer=TSC
  [Host]     : .NET Framework 4.7 (CLR 4.0.30319.42000), 64bit RyuJIT-v4.7.2053.0
  DefaultJob : .NET Framework 4.7 (CLR 4.0.30319.42000), 64bit RyuJIT-v4.7.2053.0
```
<div class="scrollable-table-wrapper" markdown="block">

 |         Method |      Mean |     Error |    StdDev | Mispredict rate | BranchInstructions/Op | BranchMispredictions/Op |
 |--------------- |----------:|----------:|----------:|----------------:|----------------------:|------------------------:|
 |   SortedBranch |  21.15 us | 0.0550 us | 0.0488 us |           0,11% |                 61712 |                      65 |
 | UnsortedBranch | 135.32 us | 0.7503 us | 0.7018 us |          21,90% |                 80158 |                   17555 |

</div>

The new report:

{: .center}
![Hardware Counters](/images/disasm/hardwareCounters.png)

## How it works

When we attach with ClrMD to the benchmarked process we ask it for the asm instructions for given address. The address is Instruction Pointer (IP).

The other diagnoser is [using ETW](http://adamsitnik.com/Hardware-Counters-ETW/) to gather the PMC events. Each event comes with hardware counter type, interval, Instruction Pointer and process Id.

When we detect that user is using both diagnosers we enable [Instruction Pointer exporter](https://github.com/dotnet/BenchmarkDotNet/blob/master/src/BenchmarkDotNet.Core/Exporters/InstructionPointerExporter.cs). It eliminates the noise (events with IPs that don't belong to the benchmarked code like BenchmarkDotNet engine) and aggregates the results.

Please keep in mind that we only show what we get. Most of the PMC events are delayed. I will try to post an update about this soon. 