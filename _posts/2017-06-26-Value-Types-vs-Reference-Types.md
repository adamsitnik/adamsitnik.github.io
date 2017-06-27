---
layout: post
title: Value Types vs Reference Types
excerpt_separator: <!--more-->
---

tl;dr `structs` have better data locality and don't add any pressure for GC. But they are expensive to copy and you can accidentally  box them which is bad.

### Introduction

The .NET framework implements Reference Types and Value Types. `C#` allows us to define custom value types by using `struct` and `enum` keywords. `class`, `delegate` and `interface` are for reference types. Primitive types, like `byte`, `char`, `short`, `int` and `long` are value types, but developers can't define custom primitive types. In Java primitive types are also value types, but Java does not expose a possibility to define custom value types for developers ;) 

**Value Types and Reference Types are very different in terms of performance characteristics**. In my next blog posts, I am going to describe `ref returns and locals`, `ValueTask<T>` and `Span<T>`. But I need to clarify this matter first, so the readers can understand the benefits.

**Note:** To keep my comparison simple I am going to use `ValueTuple<int, int>` and `Tuple<int, int>` as the examples.


## Memory Layout

**Every instance of a reference type has extra two fields that are used internally by CLR.**

* `ObjectHeader` is a bitmask, which is used by CLR to store some additional information. For example: if you take a lock on a given object instance, this information is stored in `ObjectHeader`. 
* `MethodTable` is a pointer to the Method Table, which is a set of metadata about given type. If you call a virtual method, then CLR jumps to the Method Table and obtains the address of the actual implementation and performs the actual call.

Both hidden fields size is equal to the size of a pointer. So for `32 bit` architecture, we have 8 bytes overhead and for `64 bit` 16 bytes. 

{: .center}
![Reference Type Memory Layout](/images/valueTypesVsReferenceTypes/ReferenceTypes_MemoryLayout.png)

**Value Types don't have any additional overhead members**. What you see is what you get. This is why they are more limited in terms of features. You cannot derive from `struct`, `lock` it or write finalizer for it.

{: .center}
![Value Type Memory Layout](/images/valueTypesVsReferenceTypes/ValueTypes_MemoryLayout.png)

RAM is very cheap. So, what's all the fuss about?

<!--more-->

### CPU Cache

CPU implements numerous performance optimizations. One of them is cache, which is just a memory with the most recently used data.

{: .center}
![How CPU Cache work](/images/valueTypesVsReferenceTypes/Cache.png)

Whenever you try to read a value, CPU checks the first level of cache (L1). If it's a **hit**, the value is being returned. Otherwise, it checks the second level of cache (L2). If the value is there, it's being copied to L1 and returned. Otherwise, it checks L3 (if it's present). 

If the data is not in the cache, CPU goes to the main memory and copies it to the cache. This is called **cache miss**.

### Latency Numbers Every Programmer Should Know

According to [Latency Numbers Every Programmer Should Know](https://people.eecs.berkeley.edu/~rcs/research/interactive_latency.html) going to main memory is really expensive when compared to referencing cache.

|             Operation |   Time |
|---------------------- |------- |
|    L1 cache reference |    1ns |
|    L2 cache reference |    4ns |
| Main memory reference | 100 ns |

So how can we reduce the ratio of cache misses?

### Data Locality

CPU is smart, it's aware of the following data locality principles:

* Spatial
> If a particular storage location is referenced at a particular time, then it is likely that nearby memory locations will be referenced in the near future.
* Temporal
> If at one point a particular memory location is referenced, then it is likely that the same location will be referenced again in the near future.

CPU is taking advantage of this knowledge. Whenever CPU copies a value from main memory to cache, it is copying whole **cache line**, not just the value. A cache line is usually 64 bytes. So it is well prepared in case you ask for the nearby memory location.

### The .NET Story

How the two extra fields per every reference type instance affect data locality? Let's take a look at the following diagram which shows how many instances of `ValueTuple<int, int>` and `Tuple<int, int>` can fit into single cache line for `64bit` architecture. 

{: .center}
![CPU Cache Line](/images/valueTypesVsReferenceTypes/CacheLines.png)

For this simple example, the difference is really huge. In our case, we could fit 8 instances of value type and 2.66 reference type. 

### Benchmarks!

It's important to know the theory, but we need to run some benchmarks to measure the performance difference. Once again I am using `BenchmarkDotNet` and its feature called `HardwareCounters` which allows me to track CPU Cache Misses. [Here](http://adamsitnik.com/Hardware-Counters-Diagnoser/) you can find my blog post about Collecting Hardware Performance Counters with BenchmarkDotNet.
The benchmark is a simple loop with read access in it's every iteration. I would say that it's just a CPU cache benchmark.

**Note**: This benchmark is not a real life scenario. In real life, your struct would most probably be bigger (usually two fields is not enough). Hence the extra overhead of two fields for reference types would have a smaller performance impact. Smaller but still significant in high-performance scenarios!

```cs
class Program
{
    static void Main(string[] args) => BenchmarkRunner.Run<DataLocality>();
}

[HardwareCounters(HardwareCounter.CacheMisses)]
[RyuJitX64Job, LegacyJitX86Job]
public class DataLocality
{
    [Params(
        100,
        1000000,
        10000000,
        100000000)]
    public int Count { get; set; } // for smaller arrays we don't get enough of Cache Miss events

    Tuple<int, int>[] arrayOfRef;
    ValueTuple<int, int>[] arrayOfVal;

    [GlobalSetup]
    public void Setup()
    {
        arrayOfRef = Enumerable.Repeat(1, Count).Select((val, index) => Tuple.Create(val, index)).ToArray();
        arrayOfVal = Enumerable.Repeat(1, Count).Select((val, index) => new ValueTuple<int, int>(val, index)).ToArray();
    }

    [Benchmark(Baseline = true)]
    public int IterateValueTypes()
    {
        int item1Sum = 0, item2Sum = 0;

        var array = arrayOfVal;
        for (int i = 0; i < array.Length; i++)
        {
            ref ValueTuple<int, int> reference = ref array[i];
            item1Sum += reference.Item1;
            item2Sum += reference.Item2;
        }

        return item1Sum + item2Sum;
    }

    [Benchmark]
    public int IterateReferenceTypes()
    {
        int item1Sum = 0, item2Sum = 0;

        var array = arrayOfRef;
        for (int i = 0; i < array.Length; i++)
        {
            ref Tuple<int, int> reference = ref array[i];
            item1Sum += reference.Item1;
            item2Sum += reference.Item2;
        }

        return item1Sum + item2Sum;
    }
}
```

### The Results

``` ini
BenchmarkDotNet=v0.10.8, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i7-4700MQ CPU 2.40GHz (Haswell), ProcessorCount=8
Frequency=2338337 Hz, Resolution=427.6544 ns, Timer=TSC
  [Host]       : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.6.1649.1
  LegacyJitX86 : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.6.1649.1
  RyuJitX64    : Clr 4.0.30319.42000, 64bit RyuJIT-v4.6.1649.1

Runtime=Clr  
```

 |                Method |       Jit | Platform |     Count |              Mean | Scaled | CacheMisses/Op |
 |---------------------- |---------- |--------- |---------- |------------------:|-------:|---------------:|
 |     IterateValueTypes | LegacyJit |      X86 |       100 |          68.96 ns |   1.00 |              **0** |
 | IterateReferenceTypes | LegacyJit |      X86 |       100 |         317.49 ns |   **4.60** |              **0** |
 |     IterateValueTypes |    RyuJit |      X64 |       100 |          76.56 ns |   1.00 |              **0** |
 | IterateReferenceTypes |    RyuJit |      X64 |       100 |         252.23 ns |   **3.29** |              **0** |

As you can see the difference (Scaled column) is really significant! 

But the `CacheMisses/Op` column is empty?!? What does it mean? In this case, it means that I run too few loop iterations (just 100). 

An explanation for the curious: BenchmarkDotNet is using [ETW](http://adamsitnik.com/Hardware-Counters-ETW/) to collect hardware counters. ETW is simply exposing what the hardware has to offer. Each Performance Monitoring Units (PMU) register is configured to count a specific event and given a sample-after value (SAV). For my PC the minimum Cache Miss HC sampling interval is 4000. In value type benchmark I should get Cache Miss once every 8 loop iterations (`cacheLineSize / sizeOf(ValueTuple<int, int>) = 64 / 8 = 8`). I have 100 iterations here, so it should be 12 Cache Misses for Benchmark. But the PMU will notify ETW, which will notify BenchmarkDotNet every 4 000 events. So once every 333 (`4 000 / 12`) benchmark invocation. BenchmarkDotNet implements a heuristic which decides how many times the benchmarked method should be invoked. It this example the method was executed too few times to capture enough of events. **So if you want to capture some hardware counters with BenchmarkDotNet you need to perform plenty of iterations!** For more info about PMU you can refer to [this article](https://software.intel.com/en-us/articles/understanding-how-general-exploration-works-in-intel-vtune-amplifier-xe) by Jackson Marusarz (Intel).

 |                Method |       Jit | Platform |       Count |              Mean | Scaled | CacheMisses/Op |
 |---------------------- |---------- |--------- |------------ |------------------:|-------:|---------------:|
 |     IterateValueTypes |    RyuJit |      X64 | 100 000 000 |  88,735,182.11 ns |   1.00 |        **3545088** |
 | IterateReferenceTypes |    RyuJit |      X64 | 100 000 000 | 280,721,189.70 ns |   **3.16** |        **8456940** |


The more loop iterations (Count column), the more Cache Misses events we get. **For the iteration of reference types cache misses were 2.38 times more common** (8456940 / 3545088).

**Note:** Accuracy of Hardware Counters diagnoser in BenchmarkDotNet is limited by sampling frequency and additional code performed in the benchmarked process by our Engine. It's good but not perfect. For more accurate results you should use some profilers like Intel VTune Amplifier.

## GC Impact

Reference Types are always allocated on the managed heap (it may change in the [future](http://xoofx.com/blog/2015/10/08/stackalloc-for-class-with-roslyn-and-coreclr/)). Heap is managed by Garbage Collector (GC). The allocation of heap memory is fast. **The problem is that the deallocation is non-deterministic and it takes some time to perform the cleanup**.

Value Types can be allocated both on the stack and the heap. Stack is not managed by GC. Anytime you declare a local value type variable it's allocated on the stack. When method ends, the stack is being unwinded and the value is gone. **This deallocation is super fast. And no extra pressure for the GC!**

But the Value Types can be also allocated on the managed heap. If you allocate an array of bytes, then the array is allocated on the managed heap. This content is transparent to GC. They are not reference type instances, so GC does not track them in any way. So we still follow the "NO GC" rule.

### Benchmarks

Let's run some benchmark that includes the cost of allocation and deallocation for Value Types and Reference Types.

```cs
[Config(typeof(AllocationsConfig))]
public class NoGC
{
    [Benchmark(Baseline = true)]
    public ValueTuple<int, int> CreateValueTuple() => ValueTuple.Create(0, 0);

    [Benchmark]
    public Tuple<int, int> CreateTuple() => Tuple.Create(0, 0);
}

public class AllocationsConfig : ManualConfig
{
    public AllocationsConfig()
    {
        var gcSettings = new GcMode
        {
            Force = false // tell BenchmarkDotNet not to force GC collections after every iteration
        };

        const int invocationCount = 1 << 20; // let's run it very fast, we are here only for the GC stats

        Add(Job
            .RyuJitX64 // 64 bit
            .WithInvocationCount(invocationCount)
            .With(gcSettings.UnfreezeCopy()));
        Add(Job
            .LegacyJitX86 // 32 bit
            .WithInvocationCount(invocationCount)
            .With(gcSettings.UnfreezeCopy()));

        Add(MemoryDiagnoser.Default);
    }
}
```

### The Results

If you are not familiar with the output produced by BenchmarkDotNet with Memory Diagnoser enabled, you can read my [dedicated blog post](http://adamsitnik.com/the-new-Memory-Diagnoser/#how-to-read-the-results) to find out how to read these results.

``` ini
BenchmarkDotNet=v0.10.8, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i7-4700MQ CPU 2.40GHz (Haswell), ProcessorCount=8
Frequency=2338337 Hz, Resolution=427.6544 ns, Timer=TSC
  [Host]     : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.6.1649.1
  Job-QZDRYZ : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.6.1649.1
  Job-XFJRTH : Clr 4.0.30319.42000, 64bit RyuJIT-v4.6.1649.1

Runtime=Clr  Force=False  InvocationCount=1048576  
```

 |           Method |       Jit | Platform |  Gen 0 | Allocated |
 |----------------- |---------- |--------- |-------:|----------:|
 | CreateValueTuple | LegacyJit |      X86 |      **-** |       **0 B** |
 |      CreateTuple | LegacyJit |      X86 | **0.0050** |      **16 B** |
 | CreateValueTuple |    RyuJit |      X64 |      **-** |       **0 B** |
 |      CreateTuple |    RyuJit |      X64 | **0.0076** |      **24 B** |

 As you can see, creating Value Types means No GC (`-` in Gen 0 column).

## Boxing

Whenever a reference is required value types are being boxed. When the CLR boxes a value type, it wraps the value inside a System.Object and stores it on the managed heap. **GC tracks references to boxed Value Types!** This is something you definitely want to avoid.

Obvious boxing example:

```cs
string CallToString(object input) => input.ToString();

int value = 123;
var text = CallToString(value);
```

`CallToString` accepts `object`. CLR needs to box the value before passing it to this method. It's clear when you analyse the IL code:

{: .center}
![Boxing](/images/valueTypesVsReferenceTypes/boxing.png)

**Note:** You can use ReSharper's [Heap Allocation Viewer plugin](https://blog.jetbrains.com/dotnet/2014/06/06/heap-allocations-viewer-plugin/) to detect boxing in your code.

### Invoking interface methods with Value Types

The previous example was obvious. But what happens when we try to pass a struct to a method that accepts interface instance? Let's take a look.

```cs
[MemoryDiagnoser]
[RyuJitX64Job, LegacyJitX86Job]
public class ValueTypeInvokingInterfaceMethod
{
    interface IInterface
    {
        void DoNothing();
    }

    class ReferenceTypeImplementingInterface : IInterface
    {
        public void DoNothing() { }
    }

    struct ValueTypeImplementingInterface : IInterface
    {
        public void DoNothing() { }
    }

    private ReferenceTypeImplementingInterface reference = new ReferenceTypeImplementingInterface();
    private ValueTypeImplementingInterface value = new ValueTypeImplementingInterface();

    [Benchmark(Baseline = true)]
    public void ValueType() => AcceptingInterface(value);

    [Benchmark]
    public void ReferenceType() => AcceptingInterface(reference);

    void AcceptingInterface(IInterface instance) => instance.DoNothing();
}
```

``` ini
BenchmarkDotNet=v0.10.8, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i7-4700MQ CPU 2.40GHz (Haswell), ProcessorCount=8
Frequency=2338337 Hz, Resolution=427.6544 ns, Timer=TSC
  [Host]       : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.6.1649.1
  LegacyJitX86 : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.6.1649.1
  RyuJitX64    : Clr 4.0.30319.42000, 64bit RyuJIT-v4.6.1649.1

Runtime=Clr  
```

 |        Method |       Jit | Platform |     Mean | Scaled |  Gen 0 | Allocated |
 |-------------- |---------- |--------- |---------:|-------:|-------:|----------:|
 |     ValueType | LegacyJit |      X86 | 5.738 ns |   1.00 | **0.0038** |      **12 B** |
 | ReferenceType | LegacyJit |      X86 | 1.910 ns |   **0.33** |      - |       0 B |
 |     ValueType |    RyuJit |      X64 | 5.754 ns |   1.00 | **0.0076** |      **24 B** |
 | ReferenceType |    RyuJit |      X64 | 1.845 ns |   **0.32** |      - |       0 B |

Once again we got into boxing. Did you expect it?!

### How to avoid boxing with value types that implement interfaces?

We can tell the C# compiler, that given generic method accepts an instance of value type that implements given interface.

```cs
void Trick<T>(T instance)
    where T : struct, IInterface
{
    instance.Method();
}
```

#### Benchmarks

```cs
[MemoryDiagnoser]
[RyuJitX64Job]
public class ValueTypeInvokingInterfaceMethodSmart
{
    // IInterface, ReferenceTypeImplementingInterface, ValueTypeImplementingInterface and fields are declared in previous benchmark

    [Benchmark(Baseline = true, OperationsPerInvoke = 16)]
    public void ValueType()
    {
        AcceptingInterface(value); AcceptingInterface(value); AcceptingInterface(value); AcceptingInterface(value);
        AcceptingInterface(value); AcceptingInterface(value); AcceptingInterface(value); AcceptingInterface(value);
        AcceptingInterface(value); AcceptingInterface(value); AcceptingInterface(value); AcceptingInterface(value);
        AcceptingInterface(value); AcceptingInterface(value); AcceptingInterface(value); AcceptingInterface(value);
    }

    [Benchmark(OperationsPerInvoke = 16)]
    public void ValueTypeSmart()
    {
        AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value);
        AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value);
        AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value);
        AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value);
        AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value); AcceptingStructThatImplementsInterface(value);
    }

    [Benchmark(OperationsPerInvoke = 16)]
    public void ReferenceType()
    {
        AcceptingInterface(reference); AcceptingInterface(reference); AcceptingInterface(reference); AcceptingInterface(reference);
        AcceptingInterface(reference); AcceptingInterface(reference); AcceptingInterface(reference); AcceptingInterface(reference);
        AcceptingInterface(reference); AcceptingInterface(reference); AcceptingInterface(reference); AcceptingInterface(reference);
        AcceptingInterface(reference); AcceptingInterface(reference); AcceptingInterface(reference); AcceptingInterface(reference);
    } 

    void AcceptingInterface(IInterface instance) => instance.DoNothing();

    void AcceptingStructThatImplementsInterface<T>(T instance)
        where T : struct, IInterface
    {
        instance.DoNothing();
    }
}
```

**Note:** I have used `OperationsPerInvoke` feature of BenchmarkDotNet which is very usefull for nano-benchmarks.

``` ini
BenchmarkDotNet=v0.10.8, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i7-4700MQ CPU 2.40GHz (Haswell), ProcessorCount=8
Frequency=2338337 Hz, Resolution=427.6544 ns, Timer=TSC
  [Host]    : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.6.1649.1
  RyuJitX64 : Clr 4.0.30319.42000, 64bit RyuJIT-v4.6.1649.1

Job=RyuJitX64  Jit=RyuJit  Platform=X64  
```

 |         Method |     Mean |     Error |    StdDev | Scaled |  Gen 0 | Allocated |
 |--------------- |---------:|----------:|----------:|-------:|-------:|----------:|
 |      ValueType | 5.572 ns | 0.0322 ns | 0.0252 ns |   1.00 | 0.0076 |      24 B |
 | ValueTypeSmart | **1.145 ns** | 0.0101 ns | 0.0094 ns |   **0.21** |      - |       **0 B** |
 |  ReferenceType | **2.212 ns** | 0.0096 ns | 0.0081 ns |   **0.40** |      - |       0 B |

 By applying this simple trick we were able to not only avoid boxing but also outperform reference type interface method invocation! It was possible due to the optimization performed by JIT. I am going to call it method de-virtualization because I don't have a better name for it. How does it work? Let's consider following example:
 
 ```cs
 public void Method<T>(T instance)
        where T : struct, IDisposable
{
        instance.Dispose();
}
```
 
 When the `T` is constrained with `where T : struct, INameOfTheInterface`, the C# compiler emits additional `IL` instruction called `constrained` ([Docs](http://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.constrained.aspx)).
 
 ```cs
 .method public hidebysig 
    instance void Method<valuetype .ctor ([mscorlib]System.IDisposable, [mscorlib]System.ValueType) T> (
        !!T 'instance'
    ) cil managed 
{
    .maxstack 8

    IL_0000: ldarga.s 'instance'
    IL_0002: constrained. !!T
    IL_0008: callvirt instance void [mscorlib]System.IDisposable::Dispose()
    IL_000d: ret
} // end of method C::Method
```

If there is no constraint the instance can be anything: value or reference type. In case it's value type, the JIT performs boxing.
This generic constraint is a clear signal for JIT that `T` is a value type, so it can avoid boxing.

JIT handles value types in a different way than reference types. Operations, like passing to a method or returning from it are the same for all reference types. We always deal with pointers, which have single, same size for all reference types. So JIT is reusing the compiled generic code for reference types because it can treat them in the same way. Imagine an array of `objects` or `strings`. From JITs perspective, it is just an array of pointers. So the array's indexer implementation will be the same for all reference types.

Value Types are different. Each of them can have different size. For example passing `integer` and custom `struct` with two integer fields to a method has a different native implementation. In one case we push single int to the stack, in the other, we might need to move two fields to the registers, and then push them to the stack. So it's different per every value type. 

This is why JIT compiles ever generic method/type separately for generic value types arguments.

```cs
Method<object>(); // JIT compiled code is common for all reference types
Method<string>(); // JIT compiled code is common for all reference types
Method<int>(); // dedicated version for int
Method<long>(); // dedicated version for long
Method<DateTime>(); // dedicated version for DateTime
```

It might lead to [generic Code Bloat](https://blogs.msdn.microsoft.com/carlos/2009/11/09/net-generics-and-code-bloat-or-its-lack-thereof/). But the great thing is that at this point in time, JIT can compile **tailored** code per type. And since the type is know, it can **replace virtual call with direct call**. As [Victor Baybekov](https://twitter.com/buybackoff) mentioned in the comments, it can even remove the unnecessary null check for the call. It's value type, so it can not be null. Inlining is also possible. 
For small methods, which are executed very often, like `.Equals()` in [custom Dictionary implementation](https://github.com/Spreads/Spreads.Unsafe#fastdictionary) it can be very big performance gain.

**Note:** If you would like to play with generated IL code you can use the awesome [SharpLab](https://sharplab.io/#v2:C4LglgNgNAJiDUAfAAgBgATIIwG4CwAUMgMyYBM6AwugN6HoOanIAs6AsgKbAAWA9jAA8AFQB8ACmHowAOwDOwAIYyAxpwCU9RtoDuPTgCdO6KSHQKDAVxXAo6AJIARMHIAOfOYoBGETloZ0BNrBjLIKymoAdM5uHpzi6vhBjAC+hClAA===).

## Copying

In C# by default Value Types are passed to methods by value. It means that the Value Type instance is copied every time we pass it to a method. Or when we return it from a method. The bigger the Value Type is, the more expensive it is to copy it.

```cs
[RyuJitX64Job, LegacyJitX86Job]
public class CopyingValueTypes
{
    class ReferenceType1Field { int X; }
    class ReferenceType2Fields { int X, Y; }
    class ReferenceType3Fields { int X, Y, Z; }

    struct ValueType1Field { int X; }
    struct ValueType2Fields { int X, Y; }
    struct ValueType3Fields { int X, Y, Z; }

    ReferenceType1Field fieldReferenceType1Field = new ReferenceType1Field();
    ReferenceType2Fields fieldReferenceType2Fields = new ReferenceType2Fields();
    ReferenceType3Fields fieldReferenceType3Fields = new ReferenceType3Fields();

    ValueType1Field fieldValueType1Field = new ValueType1Field();
    ValueType2Fields fieldValueType2Fields = new ValueType2Fields();
    ValueType3Fields fieldValueType3Fields = new ValueType3Fields();

    [MethodImpl(MethodImplOptions.NoInlining)] ReferenceType1Field Return(ReferenceType1Field instance) => instance;
    [MethodImpl(MethodImplOptions.NoInlining)] ReferenceType2Fields Return(ReferenceType2Fields instance) => instance;
    [MethodImpl(MethodImplOptions.NoInlining)] ReferenceType3Fields Return(ReferenceType3Fields instance) => instance;

    [MethodImpl(MethodImplOptions.NoInlining)] ValueType1Field Return(ValueType1Field instance) => instance;
    [MethodImpl(MethodImplOptions.NoInlining)] ValueType2Fields Return(ValueType2Fields instance) => instance;
    [MethodImpl(MethodImplOptions.NoInlining)] ValueType3Fields Return(ValueType3Fields instance) => instance;

    [Benchmark(OperationsPerInvoke = 16)]
    public void TestReferenceType1Field()
    {
        var instance = fieldReferenceType1Field;
        instance = Return(instance); instance = Return(instance); instance = Return(instance); instance = Return(instance);
        instance = Return(instance); instance = Return(instance); instance = Return(instance); instance = Return(instance);
        instance = Return(instance); instance = Return(instance); instance = Return(instance); instance = Return(instance);
        instance = Return(instance); instance = Return(instance); instance = Return(instance); instance = Return(instance);
    }

    // removed
}
```

The rest of the code was removed for brevity. You can find full code [here](https://gist.github.com/adamsitnik/5c1b36c75c94c3ab819de47b5addf3bc).

``` ini
BenchmarkDotNet=v0.10.8, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i7-4700MQ CPU 2.40GHz (Haswell), ProcessorCount=8
Frequency=2338337 Hz, Resolution=427.6544 ns, Timer=TSC
  [Host]       : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.6.1649.1
  LegacyJitX86 : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.6.1649.1
  RyuJitX64    : Clr 4.0.30319.42000, 64bit RyuJIT-v4.6.1649.1

Runtime=Clr  
```

 |                   Method |       Jit | Platform |     Mean |
 |------------------------- |---------- |--------- |---------:|
 |  TestReferenceType1Field | LegacyJit |      X86 | 1.399 ns |
 | TestReferenceType2Fields | LegacyJit |      X86 | 1.392 ns |
 | TestReferenceType3Fields | LegacyJit |      X86 | 1.388 ns |
 |  TestReferenceType1Field |    RyuJit |      X64 | 1.737 ns |
 | TestReferenceType2Fields |    RyuJit |      X64 | 1.770 ns |
 | TestReferenceType3Fields |    RyuJit |      X64 | 1.711 ns |

 Passing and returning Reference Types is size-independent. Only a copy of the pointer is passed. And pointer can always fit into CPU register.

 |                   Method |       Jit | Platform |     Mean |
 |------------------------- |---------- |--------- |---------:|
 |      TestValueType1Field | LegacyJit |      X86 | 1.410 ns |
 |     TestValueType2Fields | LegacyJit |      X86 | 6.859 ns |
 |     TestValueType3Fields | LegacyJit |      X86 | 6.837 ns |
 |      TestValueType1Field |    RyuJit |      X64 | 1.465 ns |
 |     TestValueType2Fields |    RyuJit |      X64 | 8.403 ns |
 |     TestValueType3Fields |    RyuJit |      X64 | 2.627 ns |

 The bigger the Value Type is, the more expensive copying is. 
 Have you noticed that `TestValueType3Fields` was faster than `TestValueType2Fields` for `RyuJit`? To answer the question why we would need to analyse the generated native assembly code.

 
 **How can we avoid copying big Value Types? We should pass and return them by Reference!
 I am going to leave it here, and continue with my `ref returns and locals` blog post next week.**

## Summary

* Every instance of a reference type has two extra fields used internally by CLR.
* Value Types have no hidden overhead, so they have better data locality.
* Reference Types are managed by GC. It tracks the references, offers fast allocation and expensive, non-deterministic deallocation.
* Value Types are not managed by the GC. Value Types = No GC. And No GC is better than any GC!
* Whenever a reference is required value types are being boxed. Boxing is expensive, adds an extra pressure for the GC. You should avoid boxing if you can.
* By using generic constraints we can avoid boxing and even de-virtualize interface method calls for Value Types!
* Value Types are passed to and returned from methods by Value. So by default, they are copied all the time. 

**VERY Important!!** [Pro .NET Performance](https://www.amazon.com/dp/1430244585) book by Sasha Goldshtein, Dima Zurbalev, Ido Flatow has a whole chapter dedicated to Type Internals. If you want to learn more about it, you should definitely read it. It's the best source available, my blog post is just an overview!

## Sources

* [Pro .NET Performance](https://www.amazon.com/dp/1430244585) book by Sasha Goldshtein, Dima Zurbalev, Ido Flatow 
* [How does Object.GetType() really work?](http://tooslowexception.com/how-does-gettype-work/) blog post by Konrad Kokosa
* [Safe Systems Programming in C# and .NET](https://www.infoq.com/presentations/csharp-systems-programming) video by Joe Duffy
* [Memory Systems](http://cs.umw.edu/~finlayson/class/spring16/cpsc305/notes/14-memory.html) article by University Of Mary Washington
* [Latency Numbers Every Programmer Should Know](https://people.eecs.berkeley.edu/~rcs/research/interactive_latency.html) article by Berkeley University
* [Types of locality](https://en.wikipedia.org/wiki/Locality_of_reference#Types_of_locality) definition by Wikipedia
* [Understanding How General Exploration Works in Intel® VTune™ Amplifier XE](https://software.intel.com/en-us/articles/understanding-how-general-exploration-works-in-intel-vtune-amplifier-xe) by Jackson Marusarz (Intel)
* [A new stackalloc operator for reference types with CoreCLR and Roslyn](http://xoofx.com/blog/2015/10/08/stackalloc-for-class-with-roslyn-and-coreclr/) blog post by Alexandre Mutel
* [Boxing and Unboxing](https://msdn.microsoft.com/pl-pl/library/yz2be5wk(v=vs.90).aspx) article by MSDN
* [Heap Allocations Viewer plugin](https://blog.jetbrains.com/dotnet/2014/06/06/heap-allocations-viewer-plugin/) blog post by Matt Ellis (JetBrains)
* [SharpLab.io](https://sharplab.io/)
* [OpCodes.Constrained Field](http://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.constrained.aspx) article by MSDN
* [.NET Generics and Code Bloat](https://blogs.msdn.microsoft.com/carlos/2009/11/09/net-generics-and-code-bloat-or-its-lack-thereof/) article by MSDN
* [What happens with a generic constraint that removes this requirement?](https://stackoverflow.com/a/5532061) Stack Overflow answer by Eric Lippert
