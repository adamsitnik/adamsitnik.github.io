---
layout: post
title: ref returns and ref locals
excerpt_separator: <!--more-->
---

tl;dr Pass and return by reference to avoid large struct copying. It's type and memory safe. It can be even **faster** than `unsafe`!

## Introduction

Since C# 1.0 we could pass arguments to methods by reference. It means that instead of copying value types every time we pass them to a method we can just pass them by reference. It allows us to overcome one of the very few disadvantages of value types which I described in my [previous](http://adamsitnik.com/Value-Types-vs-Reference-Types/) blog post "Value Types vs Reference Types".

Passing is not enough to cover all scenarios. C# 7.0 adds new possibilities: declaring references to local variables and returning by reference from methods.

**Note:** I want to focus on the performance aspect here. If you want to learn more about `ref returns and ref locals` you should read these awesome [blog posts](http://mustoverride.com/tags/#refs) from Vladimir Sadov. He is the software engineer who has implemented this feature for C# compiler. So you can get it straight from the horse's mouth!

### Reminder

Let's analyse some simple C# examples to make sure that we have good common understanding of the syntax.

* `void method(ref int argument)` - The argument is passed to the method by **ref**erence. 
```cs
int localVariable = 123;
ref int localReference = ref localVariable;
```
*  `ref localVariable` - De**ref**erencing local variable. If you have C++ background you can think of it as of `*localVariable`
* `ref int localReference` - Defining local **ref**erence. `localReference` is an alias of an existing variable.
* `ref array[0]` - De**ref**erencing array's first element.
* `ref int method()` - The result of the method is passed by **ref**erence. The method still returns an int. [Not a pointer!](http://mustoverride.com/refs-not-ptrs/)

## Passing arguments to methods by reference

In C# by default Value Types are passed to methods by value. It means that the Value Type instance is copied every time we pass it to a method. Or when we return it from a method. The bigger the Value Type is, the more expensive it is to copy it.

We can pass arguments to methods by reference. It's not a new feature, it was part of C# 1.0. Anyway, I am going to measure it to make sure that it actually improves the performance. Once again I am using [BenchmarkDotNet](http://benchmarkdotnet.org/) for benchmarking.

### Benchmarks

```cs
[LegacyJitX86Job, LegacyJitX64Job, RyuJitX64Job] // run the benchmarks for all available jits
public class PassingByReference
{
    struct BigStruct
    {
        public int Int1, Int2, Int3, Int4, Int5;
    }

    private BigStruct field = new BigStruct();

    [Benchmark(OperationsPerInvoke = 16, Baseline = true)]
    public void PassByValue()
    {
        var copy = field; // access the field only once to not influence the benchmark too much
        Method(copy); Method(copy); Method(copy); Method(copy);
        Method(copy); Method(copy); Method(copy); Method(copy);
        Method(copy); Method(copy); Method(copy); Method(copy);
        Method(copy); Method(copy); Method(copy); Method(copy);
    }

    [Benchmark(OperationsPerInvoke = 16)]
    public void PassByReference()
    {
        ref var local = ref field; // access the field only once to not influence the benchmark too much
        Method(ref local); Method(ref local); Method(ref local); Method(ref local);
        Method(ref local); Method(ref local); Method(ref local); Method(ref local);
        Method(ref local); Method(ref local); Method(ref local); Method(ref local);
        Method(ref local); Method(ref local); Method(ref local); Method(ref local);
    }

    [MethodImpl(MethodImplOptions.NoInlining)]
    void Method(BigStruct value) { }

    [MethodImpl(MethodImplOptions.NoInlining)]
    void Method(ref BigStruct value) { }
}
```

### Results

``` ini
BenchmarkDotNet=v0.10.8, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i7-4700MQ CPU 2.40GHz (Haswell), ProcessorCount=8
Frequency=2338342 Hz, Resolution=427.6534 ns, Timer=TSC
  [Host]       : Clr 4.0.30319.42000, 64bit RyuJIT-v4.7.2053.0
  LegacyJitX64 : Clr 4.0.30319.42000, 64bit LegacyJIT/clrjit-v4.7.2053.0;compatjit-v4.7.2053.0
  LegacyJitX86 : Clr 4.0.30319.42000, 32bit LegacyJIT-v4.7.2053.0
  RyuJitX64    : Clr 4.0.30319.42000, 64bit RyuJIT-v4.7.2053.0

Runtime=Clr  
```

 |          Method |       Jit | Platform |     Mean | Scaled |
 |---------------- |---------- |--------- |---------:|-------:|
 |     PassByValue | LegacyJit |      X86 | 2.868 ns |   1.00 |
 | PassByReference | LegacyJit |      X86 | 1.434 ns |   **0.50** |

As you can see the 32bit JIT is struggling with copying large value types. Passing them by reference gave us x2 speed up in this scenario. But we have measured only the time required to pass an argument to a method. If a method is complex and time-consuming itself, the performance improvement might be very small. The smaller the method, the bigger improvement you will see.


 |          Method |       Jit | Platform |     Mean | Scaled |
 |---------------- |---------- |--------- |---------:|-------:|
 |     PassByValue | LegacyJit |      X64 | 2.062 ns |   1.00 |
 | PassByReference | LegacyJit |      X64 | 1.470 ns |   **0.71** |
 |     PassByValue |    RyuJit |      X64 | 2.098 ns |   1.00 |
 | PassByReference |    RyuJit |      X64 | 1.593 ns |   **0.76** |

 For 64 bit the difference is smaller but still noticeable.

We can say that passing argument by reference can bring you some benefits, but you should not expect x10 time improvement. The code gets also more complex. Measure your scenario and if you prove that it brings you worthy performance improvement then use it. 

## Local references

Using local references is another way to avoid copying of memory. Let's try to initialize an array of large value types and see how fast we can get with `ref locals`.

### Benchmarks

```cs
[LegacyJitX86Job, LegacyJitX64Job, RyuJitX64Job]
[RPlotExporter] // use R to get nice charts!
[CsvMeasurementsExporter] // use R to get nice charts!
public class InitializingBigStructs
{
    struct BigStruct
    {
        public int Int1, Int2, Int3, Int4, Int5;
    }

    private BigStruct[] array;

    [GlobalSetup]
    public void Setup() => array = new BigStruct[1000];

    [Benchmark]
    public void ByValue()
    {
        BigStruct[] variable = array;
        for (int i = 0; i < variable.Length; i++)
        {
            BigStruct value = variable[i]; // copy the value 1st time

            value.Int1 = 1;
            value.Int2 = 2;
            value.Int3 = 3;
            value.Int4 = 4;
            value.Int5 = 5;

            variable[i] = value; // copy the value 2nd time
        }
    }

    [Benchmark(Baseline = true)]
    public void ByReference()
    {
        BigStruct[] variable = array;
        for (int i = 0; i < variable.Length; i++)
        {
            ref BigStruct reference = ref variable[i]; // create local alias to array storage

            reference.Int1 = 1;
            reference.Int2 = 2;
            reference.Int3 = 3;
            reference.Int4 = 4;
            reference.Int5 = 5;
        }
    }
}
```

**Note:** This scenario could have been handled without ref locals. We could simply pass the argument by reference like this:

```cs
public void ByReferenceOldWay()
{
    for (int i = 0; i < array.Length; i++)
    {
        Init(ref array[i]);
    }
}

// try it with: [MethodImpl(MethodImplOptions.NoInlining)]
private void Init(ref BigStruct reference)
{
    reference.Int1 = 1;
    reference.Int2 = 2;
    reference.Int3 = 3;
    reference.Int4 = 4;
    reference.Int5 = 5;
}
```

### Results

This time I have used `RPlotExporter` which produces some fancy charts if you have `R` installed and `R_HOME` environment variable configured. More info can be obtained [here](http://benchmarkdotnet.org/Configs/Exporters.htm#plots).

{: .center}
![Initializing Big Structs](/images/references/InitializingBigStructs-barplot.png)

As you can see the difference is HUGE. In this simple scenario, we got x4.51 performance improvement for RyuJit! 5.58 for LegacyJitX64 and 5.05 for LegacyJitX86. It's clear that this feature can be very useful in similar scenarios.

But how is it possible that the new `ref locals` feature works with the legacy Jits? Have Microsoft released a Windows patch with .NET framework improvements? No! C# features like ref parameters, locals and returns are just using the existing feature of CLR called [managed pointers](http://mustoverride.com/managed-refs-CLR/). So to use it you just need IDE with Roslyn 2.0 version (Visual Studio 2017 or Rider) and you can deploy the code to your client's old virtual machine which might be running with some very old .NET framework ;)

## Returning references

Returning results by reference can bring us performance gains similar to passing arguments by reference. This is why I am not going to run any benchmarks here. The very important thing is that they can help us to make things like `Span<T>` come true. I'll describe `Span<T>` in my next blog post. Stay tuned!

## Safety

As you most probably know C# allows us to use `unsafe` `C++`-like pointers. Unsafe code can not pass the IL verification, which is one of the CLR mechanisms that ensure type and memory safety ([PEVerify](https://docs.microsoft.com/en-us/dotnet/framework/tools/peverify-exe-peverify-tool)). This is why the code that is using `unsafe` requires [FullTrust](https://stackoverflow.com/a/706578/5852046) to be executed. It might be not an option in some environments. I remember that long time ago the default settings in Azure were not allowing `unsafe` code to run. This is why a lot of common high-performance .NET libraries are not using `unsafe`. `mscorlib.dll` is using `unsafe`, but it is exceptionally not verified during runtime ;)

Let's compare the performance of safe vs unsafe in the scenario described previously.

### Benchmarks

```cs
[Benchmark]
public unsafe void ByReferenceUnsafe()
{
    BigStruct[] variable = array;
    fixed (BigStruct* pinned = variable)
    {
        for (int i = 0; i < variable.Length; i++)
        {
            BigStruct* pointer = &pinned[i];
            (*pointer).Int1 = 1;
            (*pointer).Int2 = 2;
            (*pointer).Int3 = 3;
            (*pointer).Int4 = 4;
            (*pointer).Int5 = 5;
        }
    }
}
```

### Results

 |            Method |       Jit | Platform |     Mean | Scaled |
 |------------------ |---------- |--------- |---------:|-------:|
 |       ByReference | LegacyJit |      X64 | 1.649 us |   1.00 |
 | ByReferenceUnsafe | LegacyJit |      X64 | 1.721 us |   **1.04** |
 |       ByReference | LegacyJit |      X86 | **1.666** us |   1.00 |
 | ByReferenceUnsafe | LegacyJit |      X86 | **1.673** us |   1.00 |
 |       ByReference |    RyuJit |      X64 | 1.684 us |   1.00 |
 | ByReferenceUnsafe |    RyuJit |      X64 | 1.709 us |   **1.02** |

To our surprise, the safe way is faster than unsafe! Why is that?

When we are using safe references, we don't need to pin objects in memory. GC understand managed pointers and knows how to update them when it's compacting the memory. With `unsafe` this is not true, the managed memory needs to be pinned before it can be used.

**Note:** This micro-benchmark is not including the side effects of pinning memory. If you pin many managed arrays in memory, then the GC has a lot of extra work to do when it's compacting the memory. Once again I will redirect you to [Pro .NET Performance](https://www.amazon.com/dp/1430244585) book by Sasha Goldshtein, Dima Zurbalev, Ido Flatow which has a whole chapter dedicated to Garbage Collection in .NET.

## Managed pointers arithmetic

C# 7.0 is not exposing managed pointers arithmetic, which is a part of the IL language. But the `System.Runtime.CompilerServices.Unsafe` class does. You can use it whenever you want to compare the references, move them by given offset etc.

You can find it the `System.Runtime.CompilerServices.Unsafe` [NuGet package](https://www.nuget.org/packages/System.Runtime.CompilerServices.Unsafe/4.3.0). It targets .NET Standard 1.0 so you can use it in both .NET 4.5+ and .NET Core 1.0 apps. Not to speak about other frameworks that implement the standard. The api is following:

```cs
namespace System.Runtime.CompilerServices
{
    public static partial class Unsafe
    {
        public static ref T AddByteOffset<T>(ref T source, System.IntPtr byteOffset) 
        public static ref T Add<T>(ref T source, int elementOffset)
        public static ref T Add<T>(ref T source, System.IntPtr elementOffset) 
        public static bool AreSame<T>(ref T left, ref T right)
        public unsafe static void* AsPointer<T>(ref T value)
        public unsafe static ref T AsRef<T>(void* source)
        public static T As<T>(object o) where T : class
        public static ref TTo As<TFrom, TTo>(ref TFrom source)
        public static System.IntPtr ByteOffset<T>(ref T origin, ref T target) 
        public static int SizeOf<T>()
        public static ref T SubtractByteOffset<T>(ref T source, System.IntPtr byteOffset) 
        public static ref T Subtract<T>(ref T source, int elementOffset)
        public static ref T Subtract<T>(ref T source, System.IntPtr elementOffset) 
    }
}
```

**Note:** it's not full api. Some methods were removed for brevity. You can find the single `.il` file that contains the implementation in the [corefx repo](https://github.com/dotnet/corefx/blob/master/src/System.Runtime.CompilerServices.Unsafe/src/System.Runtime.CompilerServices.Unsafe.il)

## Current limitations

C# 7.0, which is the current version for C# language as of today (3rd of July 2017) does not allow to:

* use `readonly` references (you can not dereference a `readonly` field)
* treat `this` as readonly references for `readonly` structs
* define `by-ref` fields
* define `by-ref` extension methods. It's possible today with Visual Basic!
* use conditional operator with refs (`condition ? ref left : ref right`)

Hopefully all these limitations are going to be addressed by C# 7.2. You can follow the issues today on GitHub:

* [Champion "Readonly ref"](https://github.com/dotnet/csharplang/issues/38)
* [Champion "readonly for locals and parameters"](https://github.com/dotnet/csharplang/issues/188)
* [Champion "ref extension methods on structs"](https://github.com/dotnet/csharplang/issues/186)
* [Champion "conditional ref operator"](https://github.com/dotnet/csharplang/issues/223)


Don't forget to support the new C# 7.2 features! Thumbs up!

{: .center}
![Thumbs up](/images/references/thumbsup.png)

## Sources

* [ref returns are not pointers](http://mustoverride.com/refs-not-ptrs/) blog post by Vladimir Sadov
* [Managed pointers](http://mustoverride.com/refs-not-ptrs/) blog post by Vladimir Sadov
* [Local variables cannot be returned by reference](http://mustoverride.com/ref-returns-and-locals/) blog post by Vladimir Sadov
* [Safe to return rules for ref returns.](http://mustoverride.com/safe-to-return/) blog post by Vladimir Sadov
* [Why ref locals allow only a single binding?](http://mustoverride.com/ref-locals_single-assignment/) blog post by Vladimir Sadov
* [Spans and ref part 1 : ref](http://blog.marcgravell.com/2017/04/spans-and-ref-part-1-ref.html) blog post by Marc Gravell
* [What are the implications of using unsafe code?](https://stackoverflow.com/a/706578/5852046) Stack Overflow answer by Jared Par
