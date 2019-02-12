---
title: "Vectorize All the Things: Multiply-Add"
date: 2019-02-12
tags: [ ".net core", performance, mkl, fsharp, vectors ]
categories: [ performance ]
---

The rise of Machine Learning is bringing the fashion for numerical tricks and hardware acceleration back.
And now that I've completed my bacterial DNA counting adventures at ETHZ, it's time to reboot this blog.

As someone who believes in [mechanical sympathy](https://mechanical-sympathy.blogspot.com/2011/07/why-mechanical-sympathy.html) 
I'd like to share some of the experiments in vectorizing simple-yet-fundamental operations.
And if you have some insights how to beat the benchmarks below, I'd gladly accept them and extend the list.

After working for some time with not the most recent .NET Framework I couldn't wait to check out .NET Core and all performance goodies that come with it.
For benchmarking I highly recommend [BenchmarkDotNet](https://benchmarkdotnet.org/index.html), it's an amazing tool that will take the best care of your benchmarking needs.

So let's dig into Multiply-Add operations and have a look at different ways to vectorize it - in .NET Framework, .NET Core and MKL.
The code examples are in F# but it's of course also applicable to other .NET languages.
Try out your performance intuition ;)

{{% admonition type="question" title="Is vectorization more efficient on .NET Framework 4.7.2 or .NET Core 2.1.6?" %}} {{% /admonition %}}

The solution with examples is [here](https://github.com/luajalla/Experimental). It doesn't include a custom-built `Experimental.Mkl.dll`, but the list of functions `Experimental.Mkl.Functions.txt`
and the template for the commands required to build it (`BuildMkl.bat`), are there as well as full benchmark outputs.
Describing MKL features is not the goal of this post as it's used just as another reference result to compare against .NET implementation, so the setup is not discussed in detail.
If you're interested in finding out more about [Intel MKL Builder](https://software.intel.com/en-us/mkl-windows-developer-guide-building-custom-dynamic-link-libraries)
or the [library itself](https://software.intel.com/en-us/mkl-developer-reference-c), please check out Intel documentation.

Note, that the tests were performed only on a single Windows 10 machine.


{{< highlight ini >}}
BenchmarkDotNet=v0.11.3, OS=Windows 10.0.17134.523 (1803/April2018Update/Redstone4)       
Intel Xeon E-2176M CPU 2.70GHz, 1 CPU, 12 logical and 6 physical cores                    
Frequency=2648441 Hz, Resolution=377.5806 ns, Timer=TSC                                   
.NET Core SDK=2.1.500                                                                     
  [Host] : .NET Core 2.1.6 (CoreCLR 4.6.27019.06, CoreFX 4.6.27019.05), 64bit RyuJIT DEBUG
  Clr    : .NET Framework 4.7.2 (CLR 4.0.30319.42000), 64bit RyuJIT-v4.7.3260.0           
  Core   : .NET Core 2.1.6 (CoreCLR 4.6.27019.06, CoreFX 4.6.27019.05), 64bit RyuJIT      
{{< /highlight >}}


## Multiply-Add

What are the first operations on vectors that you can think of? I'd bet add and scale (multiplication with a scalar) are probably in this list. 
To make things more interesting let's talk about their combo:

$$\mathbf{z} = a\mathbf{x} + \mathbf{y}$$

This kind of operation is not only common in numerical code, but also has a performance boost potential
because modern processors support so-called *fused multiply-add* operations or *FMA* as an extension to SIMD instructions
(a nice introduction on the topic is available [here](https://instil.co/2016/03/21/parallelism-on-a-single-core-simd-with-c/)).
Those interested in GPU computing can also benefit from [FMAs in CUDA C](https://docs.nvidia.com/cuda/floating-point/index.html#fused-multiply-add-fma) code: the compiler is able to generate them most of the time,
and of course writing `fma` directly is also an option.

We consider two scenarios:

- Out of place, $ \mathbf{z} = a\mathbf{x} + \mathbf{y} $ (the result is stored into the 3rd array)
- In place,     $ \mathbf{y} = a\mathbf{x} + \mathbf{y} $ (one of the arrays is overwritten)

{{% admonition type="question" title="In place or Out of place?" %}} {{% /admonition %}}

For simplicity all arrays are of type `double[]` (`float[]`) and assumed to have valid dimensions.
The relative results for `single[]` (`float32[]`) are very similar but with x2 better performance.


## Baseline

The baseline for our tests is a good old `for` loop.
This is not something you'd see often in F# code, but for some application immutable-only style would be simply too expensive.

{{< highlight fsharp >}}
let forIn (a : double) (xs : double[]) (ys : double[]) =
    for i = 0 to ys.Length - 1 do
        ys.[i] <- a * xs.[i] + ys.[i]
    ys
    

let forOut (a : double) (xs : double[]) (ys : double[]) (zs : double[]) =
    for i = 0 to zs.Length - 1 do
        zs.[i] <- a * xs.[i] + ys.[i]
    zs
{{< /highlight >}}


## Vectors

{{% admonition type="question" title="For or Vectors?" %}} {{% /admonition %}}

.NET provides a set of [SIMD-enabled types](https://docs.microsoft.com/en-us/dotnet/standard/numerics) including `Vector<'T>` that we are going to test here.
For example .NET hardware intrinsics have already been successfully applied for [improving ML.NET implementation](https://blogs.msdn.microsoft.com/dotnet/2018/10/10/using-net-hardware-intrinsics-api-to-accelerate-machine-learning-scenarios/).

There're multiple ways to implement multiply-add using Vectors, and we need to take the following aspects into consideration.

### Arrays and Alignment

{{% admonition type="question" title="To align or not to align?" %}} {{% /admonition %}}

There're two options for defining input arrays: we can either use the data as is or align arrays considering the number of elements in a vector,
`Vector<double>.Count` (which is equal to 4 on my machine).
Intuitively aligned arrays should be more SIMD-friendly because in this case handling boundary cases is not required.


### Multiplication Factor 

{{% admonition type="question" title="(a : double) or Vector<double>(a)?" %}} {{% /admonition %}}
The factor `a` can be represented as either `double` or `Vector<double>`, which is interesting to check as well.

### Span

{{% admonition type="question" title="double[] with Vector(xs, i * 4) or Vector<double>[] with xs.[i]?" %}} {{% /admonition %}}

`Span<'T>` can help to cast regular arrays to arrays of vectors, which is particularly helpful for overwriting the contents of result array.
However, when it comes to function inputs we are free to choose between using a vector with offset or arrays of vectors.
To be fair, another way to save the results is to use `Vector.CopyTo`, but it seems to be far less efficient so let's skip this case.
For a comprehensive introduction into Span have a look at [this post](https://adamsitnik.com/Span/) by Adam Sitnik.

{{< highlight fsharp >}}
    /// Out of place, arrays are not aligned, vector multiplier and inputs are not spans.
    let vectorNotAlignedOut (a : double) (xs : double[]) (ys : double[]) (zs : double[]) =
        let nAligned = lengthAlignedLeq xs.Length
        let zsVector = MemoryMarshal.Cast<double, Vector<double>>(Span<double>(xs, 0, nAligned))
        let aVector  = Vector(a)
        let v        = Vector<double>.Count
        
        for i = 0 to zsVector.Length - 1 do
            zsVector.[i] <- aVector * Vector(xs, i * v) + Vector(ys, i * v)
        
        for i = nAligned to zs.Length - 1 do
            zs.[i] <- a * xs.[i] + ys.[i]
        zs

    
    /// Out of place, arrays are aligned, scalar multiplier and inputs are not spans.
    let vectorScalarOut (a : double) (xs : double[]) (ys : double[]) (zs : double[]) =
        let zsVector = MemoryMarshal.Cast<double, Vector<double>>(Span<double>(xs))
        let v        = Vector<double>.Count
        
        for i = 0 to zsVector.Length - 1 do
            zsVector.[i] <- a * Vector(xs, i * v) + Vector(ys, i * v)
        zs


    /// In place, arrays are aligned, vector multiplier and inputs are not spans.
    let vectorIn (a : double) (xs : double[]) (ys : double[]) =
        let ysVector = MemoryMarshal.Cast<double, Vector<double>>(Span<double>(ys))
        let aVector  = Vector(a)
        let v        = Vector<double>.Count
            
        for i = 0 to ysVector.Length - 1 do
            ysVector.[i] <- aVector * Vector(xs, i * v) + ysVector.[i]
        ys

    
    /// Out of place, arrays are aligned, vector multiplier and inputs are not spans.
    let vectorOut (a : double) (xs : double[]) (ys : double[]) (zs : double[]) =
        let zsVector = MemoryMarshal.Cast<double, Vector<double>>(Span<double>(xs))
        let aVector  = Vector(a)
        let v        = Vector<double>.Count
            
        for i = 0 to zsVector.Length - 1 do
            zsVector.[i] <- aVector * Vector(xs, i * v) + Vector(ys, i * v)
        zs


    /// Out of place, arrays are aligned, vector multiplier and inputs are spans.
    let vectorSpansOut (a : double) (xs : double[]) (ys : double[]) (zs : double[]) =
        let xsVector = MemoryMarshal.Cast<double, Vector<double>>(Span<double>(xs))
        let ysVector = MemoryMarshal.Cast<double, Vector<double>>(Span<double>(ys))
        let zsVector = MemoryMarshal.Cast<double, Vector<double>>(Span<double>(zs))
        let aVector  = Vector(a)
            
        for i = 0 to zsVector.Length - 1 do
            zsVector.[i] <- aVector * xsVector.[i] + ysVector.[i]
        zs
{{< /highlight >}}

Here're some of the results for the [vectors benchmark](https://github.com/luajalla/Experimental/blob/master/VectorsBmk/BenchmarkResults/results/VectorsBmk.MultiplyAddVectorMiniBmk-report.html):

- **.NET Core** code is up to ~1.8 times faster than **.NET Framework** (Clr job), this is expected because only .NET Core supports vectorization natively.
- For the same reason the relative results are not very consistent between frameworks, so further we refer only to .NET Core.
- **Aligned** version is slightly faster than **not aligned**, because the boundary counditions do not need to be handled separately, however for large arrays it doesn't seem to be the case anymore.

<table class="bmk">
<thead><tr><th> Method</th><th>Job</th><th>N</th><th>   Mean</th><th>Error</th><th>StdDev</th><th>StdErr</th><th>    Min</th><th>     Q1</th><th> Median</th><th>     Q3</th><th>    Max</th><th>   Op/s</th><th>Ratio</th><th>RatioSD</th><th>Gen 0/1k Op</th><th>Gen 1/1k Op</th><th>Gen 2/1k Op</th><th>Allocated Memory/Op</th>
</tr>
</thead><tbody>
</tr><tr><td>NotAlignedOut</td><td>Clr</td><td>100</td><td>53.441 ns</td><td>0.1227 ns</td><td>0.1148 ns</td><td>0.0296 ns</td><td>53.264 ns</td><td>53.348 ns</td><td>53.438 ns</td><td>53.518 ns</td><td>53.646 ns</td><td>18,712,195.0</td><td>1.84</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AlignedOut</td><td>Clr</td><td>100</td><td>56.728 ns</td><td>0.1804 ns</td><td>0.1599 ns</td><td>0.0427 ns</td><td>56.557 ns</td><td>56.602 ns</td><td>56.699 ns</td><td>56.792 ns</td><td>57.161 ns</td><td>17,628,008.5</td><td>1.95</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>NotAlignedOut</td><td>Core</td><td>100</td><td>29.076 ns</td><td>0.0799 ns</td><td>0.0708 ns</td><td>0.0189 ns</td><td>28.994 ns</td><td>29.024 ns</td><td>29.060 ns</td><td>29.097 ns</td><td>29.263 ns</td><td>34,392,627.8</td><td>1.00</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AlignedOut</td><td>Core</td><td>100</td><td>29.002 ns</td><td>0.4895 ns</td><td>0.4339 ns</td><td>0.1160 ns</td><td>28.459 ns</td><td>28.614 ns</td><td>28.902 ns</td><td>29.278 ns</td><td>29.991 ns</td><td>34,480,284.3</td><td>1.00</td><td>0.02</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>NotAlignedOut</td><td>Clr</td><td>103</td><td>57.559 ns</td><td>0.3002 ns</td><td>0.2661 ns</td><td>0.0711 ns</td><td>57.325 ns</td><td>57.353 ns</td><td>57.440 ns</td><td>57.645 ns</td><td>58.127 ns</td><td>17,373,491.3</td><td>1.80</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AlignedOut</td><td>Clr</td><td>103</td><td>58.622 ns</td><td>0.2398 ns</td><td>0.2243 ns</td><td>0.0579 ns</td><td>58.206 ns</td><td>58.375 ns</td><td>58.711 ns</td><td>58.783 ns</td><td>58.931 ns</td><td>17,058,484.3</td><td>1.84</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>NotAlignedOut</td><td>Core</td><td>103</td><td>31.917 ns</td><td>0.0629 ns</td><td>0.0491 ns</td><td>0.0142 ns</td><td>31.848 ns</td><td>31.880 ns</td><td>31.900 ns</td><td>31.963 ns</td><td>31.995 ns</td><td>31,331,161.4</td><td>1.00</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AlignedOut</td><td>Core</td><td>103</td><td>29.720 ns</td><td>0.0881 ns</td><td>0.0824 ns</td><td>0.0213 ns</td><td>29.612 ns</td><td>29.634 ns</td><td>29.727 ns</td><td>29.769 ns</td><td>29.884 ns</td><td>33,647,276.1</td><td>0.93</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr></tbody></table>


- As one could expect **in place** is faster than **out of place**. For example, introducing another array also requires additional boundary checks - watch out for `cmp` and `jae` instructions in disassembly:

{{< highlight asm >}}
cmp     r14d,esi
jae     00007ff9`02324920
{{< /highlight>}}

<table class="bmk">
<thead><tr><th> Method</th><th>Job</th><th>N</th><th>   Mean</th><th>Error</th><th>StdDev</th><th>StdErr</th><th>    Min</th><th>     Q1</th><th> Median</th><th>     Q3</th><th>    Max</th><th>   Op/s</th><th>Ratio</th><th>RatioSD</th><th>Gen 0/1k Op</th><th>Gen 1/1k Op</th><th>Gen 2/1k Op</th><th>Allocated Memory/Op</th>
</tr>
</thead><tbody>
</tr><tr><td>AlignedOut</td><td>Core</td><td>100003</td><td>36,877.708 ns</td><td>356.7336 ns</td><td>333.6889 ns</td><td>86.1581 ns</td><td>36,401.196 ns</td><td>36,489.669 ns</td><td>36,927.122 ns</td><td>37,110.404 ns</td><td>37,340.884 ns</td><td>27,116.7</td><td>1.04</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AlignedIn</td><td>Core</td><td>100003</td><td>34,712.877 ns</td><td>231.3058 ns</td><td>216.3635 ns</td><td>55.8648 ns</td><td>34,474.578 ns</td><td>34,511.612 ns</td><td>34,684.755 ns</td><td>34,814.202 ns</td><td>35,143.618 ns</td><td>28,807.8</td><td>0.98</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr></tbody></table>

- Multiplying with a `a : double` value is really not a good idea
and leads to ~2x performance decrease comparing to `Vector<double>(a)` version.
The reason is that a single value is not automatically vectorized
and `call` to CoreLib function is performed instead of `vmulpd` instruction:

{{< highlight asm >}}
call    System.Numerics.Vector`1[[System.Double, System.Private.CoreLib]].op_Multiply(Double, System.Numerics.Vector`1<Double>)
{{< /highlight >}}

<table class="bmk">
<thead><tr><th> Method</th><th>Job</th><th>N</th><th>   Mean</th><th>Error</th><th>StdDev</th><th>StdErr</th><th>    Min</th><th>     Q1</th><th> Median</th><th>     Q3</th><th>    Max</th><th>   Op/s</th><th>Ratio</th><th>RatioSD</th><th>Gen 0/1k Op</th><th>Gen 1/1k Op</th><th>Gen 2/1k Op</th><th>Allocated Memory/Op</th>
</tr>
</thead><tbody>
</tr><tr><td>NotAlignedOut</td><td>Core</td><td>100003</td><td>35,495.294 ns</td><td>324.4563 ns</td><td>270.9357 ns</td><td>75.1440 ns</td><td>34,658.200 ns</td><td>35,480.689 ns</td><td>35,562.836 ns</td><td>35,595.042 ns</td><td>35,808.595 ns</td><td>28,172.7</td><td>1.00</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>ScalarOut</td><td>Core</td><td>100003</td><td>77,333.464 ns</td><td>749.1684 ns</td><td>664.1182 ns</td><td>177.4931 ns</td><td>76,488.622 ns</td><td>76,955.850 ns</td><td>77,255.329 ns</td><td>77,621.272 ns</td><td>79,002.630 ns</td><td>12,931.0</td><td>2.18</td><td>0.02</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr></tbody></table>

- Finally, `Span<Vector<double>>` version seems to be faster in all cases except for large arrays.
This is an interesting behaviour, and would need a closer investigation to explain why that happens - I'll leave it for another time.

<table class="bmk">
<thead><tr><th> Method</th><th>Job</th><th>N</th><th>   Mean</th><th>Error</th><th>StdDev</th><th>StdErr</th><th>    Min</th><th>     Q1</th><th> Median</th><th>     Q3</th><th>    Max</th><th>   Op/s</th><th>Ratio</th><th>RatioSD</th><th>Gen 0/1k Op</th><th>Gen 1/1k Op</th><th>Gen 2/1k Op</th><th>Allocated Memory/Op</th>
</tr>
</thead><tbody>
</tr><tr><td>AlignedOut</td><td>Core</td><td>1003</td><td>304.120 ns</td><td>1.2893 ns</td><td>1.2060 ns</td><td>0.3114 ns</td><td>302.499 ns</td><td>303.135 ns</td><td>303.741 ns</td><td>305.064 ns</td><td>306.713 ns</td><td>3,288,176.4</td><td>0.96</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>SpansOut</td><td>Core</td><td>1003</td><td>247.484 ns</td><td>2.4008 ns</td><td>2.2457 ns</td><td>0.5798 ns</td><td>244.554 ns</td><td>245.691 ns</td><td>246.610 ns</td><td>248.767 ns</td><td>253.063 ns</td><td>4,040,668.9</td><td>0.78</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AlignedOut</td><td>Core</td><td>100003</td><td>36,877.708 ns</td><td>356.7336 ns</td><td>333.6889 ns</td><td>86.1581 ns</td><td>36,401.196 ns</td><td>36,489.669 ns</td><td>36,927.122 ns</td><td>37,110.404 ns</td><td>37,340.884 ns</td><td>27,116.7</td><td>1.04</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>SpansOut</td><td>Core</td><td>100003</td><td>38,141.007 ns</td><td>418.7630 ns</td><td>349.6861 ns</td><td>96.9855 ns</td><td>37,677.874 ns</td><td>37,876.792 ns</td><td>37,998.140 ns</td><td>38,378.497 ns</td><td>38,960.044 ns</td><td>26,218.5</td><td>1.07</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr></tbody></table>


### MKL
{{% admonition type="question" title="Applying matrix functions: [n x 1] or [1 x n]?" %}} {{% /admonition %}}

According to the [documentation](https://software.intel.com/en-us/mkl-developer-reference-c-blas-and-sparse-blas-routines)
MKL has multiple functions we can use to implement our multiply-add operation in BLAS Level 1 functions and BLAS-like extensions.
Without going into too much detail, there exist lower-level functions and extensions.
The latter support transpose operations and different matrix formats as well as offsets between rows or columns.
Here we have some freedom to pick the dimensions of the matrix, because the inputs are anyway represented as 1D arrays,
for example `[n x 1]` vs `[1 x n]` for an array with `n` elements.

{{< highlight fsharp >}}
/// In place, CBLAS daxpy 
let mklAxpyIn (a : double) (xs : double[]) (ys : double[]) =
    MKL.cblas_daxpy (xs.Length, a, xs, 1, ys, 1)
    ys


/// In place, CBLAS daxpby
let mklAxpbyIn (a : double) (xs : double[]) (ys : double[]) =
    MKL.cblas_daxpby(xs.Length, a, xs, 1, 1.0, ys, 1)
    ys
    

/// Out of place, CBLAS dcopy + daxpy 
let mklAxpyOut (a : double) (xs : double[]) (ys : double[]) (zs : double[]) =
    MKL.cblas_dcopy (xs.Length, ys, 1, zs, 1)
    MKL.cblas_daxpy (xs.Length, a, xs, 1, zs, 1)
    ys


/// Out of place, mkl_domatcopy with "matrix" dimensions [1 x n]
let mklMataddRowOut (a : double) (xs : double[]) (ys : double[]) (zs : double[]) =
    MKL.MKL_Domatadd ('R', 'N', 'N', 1, xs.Length, a, xs, xs.Length, 1.0, ys, ys.Length, zs, zs.Length)
    zs


/// Out of place, mkl_domatcopy with "matrix" dimensions [n x 1]
let mklMataddColOut (a : double) (xs : double[]) (ys : double[]) (zs : double[]) =
    MKL.MKL_Domatadd ('R', 'N', 'N', xs.Length, 1, a, xs, 1, 1.0, ys, 1, zs, 1)
    zs
{{< /highlight >}}

A quick look at the [results](https://github.com/luajalla/Experimental/blob/master/VectorsBmk/BenchmarkResults/results/VectorsBmk.MultiplyAddMklMiniBmk-report.html)
shows that one has to be very careful with functions selection:

- While flexible extensions are very useful in some scenarios our Multiply-Add usecase is not that complex.
`mkl_domatadd` is 4-8 times slower than simple `cblas_daxpy`.
To be fair `char` type for arguments in `extern` definition of `mkl_domatadd` is not a smart choice because of allocations,
it's here for clarity reasons. The better solution is to use integers, but the function will still be slower than `cblas_daxpy` (and this is noticeable mostly on smaller arrays anyway).

- Interestingly, single **Row** format `1 x n` is 3-7 times more efficient than single **Column** `n x 1` (for large array it's 53x baseline!).
- **Out of place** is around 2 times slower because it has to be performed via copy and **in place** function `cblas_daxpy`. 

<table class="bmk">
<thead><tr><th>Method</th><th>Job</th><th>N</th><th>   Mean</th><th>  Error</th><th> StdDev</th><th>StdErr</th><th>    Min</th><th>     Q1</th><th> Median</th><th>     Q3</th><th>    Max</th><th>  Op/s</th><th>Ratio</th><th>RatioSD</th><th>Gen 0/1k Op</th><th>Gen 1/1k Op</th><th>Gen 2/1k Op</th><th>Allocated Memory/Op</th>
</tr>
</thead><tbody>
</tr><tr><td>AxpyIn</td><td>Core</td><td>103</td><td>28.40 ns</td><td>0.1583 ns</td><td>0.1404 ns</td><td>0.0375 ns</td><td>28.36 ns</td><td>28.25 ns</td><td>28.32 ns</td><td>28.42 ns</td><td>28.74 ns</td><td>35,205,224.0</td><td>1.00</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AxpbyIn</td><td>Core</td><td>103</td><td>33.16 ns</td><td>0.0733 ns</td><td>0.0612 ns</td><td>0.0170 ns</td><td>33.14 ns</td><td>33.08 ns</td><td>33.12 ns</td><td>33.18 ns</td><td>33.32 ns</td><td>30,157,209.4</td><td>1.17</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AxpyOut</td><td>Core</td><td>103</td><td>49.94 ns</td><td>0.2313 ns</td><td>0.2050 ns</td><td>0.0548 ns</td><td>49.86 ns</td><td>49.71 ns</td><td>49.81 ns</td><td>49.98 ns</td><td>50.41 ns</td><td>20,025,979.5</td><td>1.76</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>MataddRowOut</td><td>Core</td><td>103</td><td>119.93 ns</td><td>0.3128 ns</td><td>0.2773 ns</td><td>0.0741 ns</td><td>119.95 ns</td><td>119.48 ns</td><td>119.72 ns</td><td>120.03 ns</td><td>120.47 ns</td><td>8,338,274.5</td><td>4.22</td><td>0.02</td><td>0.0150</td><td>-</td><td>-</td><td>96 B</td>
</tr><tr><td>MataddColOut</td><td>Core</td><td>103</td><td>385.99 ns</td><td>0.9004 ns</td><td>0.7519 ns</td><td>0.2085 ns</td><td>386.11 ns</td><td>384.55 ns</td><td>385.41 ns</td><td>386.45 ns</td><td>387.34 ns</td><td>2,590,771.2</td><td>13.59</td><td>0.08</td><td>0.0148</td><td>-</td><td>-</td><td>96 B</td>
</tr><tr><td>AxpyIn</td><td>Core</td><td>100003</td><td>5,357.79 ns</td><td>148.5879 ns</td><td>423.9297 ns</td><td>43.7250 ns</td><td>5,240.98 ns</td><td>4,563.81 ns</td><td>5,103.49 ns</td><td>5,581.75 ns</td><td>6,401.51 ns</td><td>186,644.0</td><td>1.00</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AxpbyIn</td><td>Core</td><td>100003</td><td>5,840.88 ns</td><td>122.6757 ns</td><td>253.3467 ns</td><td>35.1329 ns</td><td>5,739.96 ns</td><td>5,540.38 ns</td><td>5,632.27 ns</td><td>5,933.44 ns</td><td>6,444.85 ns</td><td>171,207.2</td><td>1.11</td><td>0.10</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>AxpyOut</td><td>Core</td><td>100003</td><td>14,290.63 ns</td><td>285.8218 ns</td><td>267.3579 ns</td><td>69.0315 ns</td><td>14,251.62 ns</td><td>13,936.29 ns</td><td>14,060.52 ns</td><td>14,463.59 ns</td><td>14,936.70 ns</td><td>69,975.9</td><td>2.77</td><td>0.34</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>MataddRowOut</td><td>Core</td><td>100003</td><td>40,185.48 ns</td><td>81.8306 ns</td><td>72.5407 ns</td><td>19.3873 ns</td><td>40,191.22 ns</td><td>40,071.33 ns</td><td>40,142.59 ns</td><td>40,216.19 ns</td><td>40,332.23 ns</td><td>24,884.6</td><td>7.83</td><td>0.93</td><td>-</td><td>-</td><td>-</td><td>96 B</td>
</tr><tr><td>MataddColOut</td><td>Core</td><td>100003</td><td>271,920.30 ns</td><td>2,036.6226 ns</td><td>1,805.4129 ns</td><td>482.5169 ns</td><td>271,196.07 ns</td><td>270,153.21 ns</td><td>270,761.24 ns</td><td>272,728.61 ns</td><td>276,126.84 ns</td><td>3,677.5</td><td>52.96</td><td>6.37</td><td>-</td><td>-</td><td>-</td><td>96 B</td>

</tr></tbody></table>


## Results

So is there a difference between `for` loops and vectorized code?

- .NET `double` Vectors are approximately **2x faster** than `for`. But RyuJIT is actually quite smart and already uses vectorized operations:

{{< highlight asm >}}
vmovaps xmm1,xmm0
vmulsd  xmm1,xmm1,mmword ptr [rdx+r9*8+10h]
vaddsd  xmm1,xmm1,mmword ptr [r8+r9*8+10h]
vmovsd  qword ptr [r8+r9*8+10h],xmm1
{{< /highlight >}}

Compare to similar code that Legacy JIT would generate for this function (e.g. `addsd` instead of `vaddsd`):

{{< highlight asm >}}
movapd      xmm0,xmm1  
mulsd       xmm0,mmword ptr [r10+r9+10h]  
addsd       xmm0,mmword ptr [r8+r9+10h]  
movsd       mmword ptr [r8+r9+10h],xmm0  
{{< /highlight >}}

And more efficient version with vectors:

{{< highlight asm >}}
vbroadcastsd ymm0,xmm0
...
vmulpd  ymm1,ymm0,ymm1
vmovupd ymm2,ymmword ptr [r9]
vaddpd  ymm1,ymm1,ymm2
vmovupd ymmword ptr [rcx],ymm1
{{< /highlight >}}

- **MKL** has some overhead or similar results on smaller arrays but better performance (up to 12x in this test) on large ones.
One of the reasons is that MKL optimizes multiply-add operations even further by using an appropriate FMA instruction `vfmadd213sd`.
More information about Advanced Vector Extensions (AVX) is available [here](https://software.intel.com/en-us/articles/how-intel-avx2-improves-performance-on-server-applications).

{{< highlight asm >}}
vmovsd      xmm0,qword ptr [r8+FFFFFFFFFFFFFF00h]
add         r8,r9
vfmadd213sd xmm0,xmm7,mmword ptr [rsi+FFFFFFFFFFFFFF00h]
vmovlpd     qword ptr [rsi+FFFFFFFFFFFFFF00h],xmm0
{{< /highlight >}}



**Summing up**, the performance improvements in .NET Core and modern RyuJIT are quite impressive and for smaller arrays are even competitive as compared to native MKL libraries.
Simple `for` loops are transformed into vectorized instructions, even though explicitly vectorized code is still significantly faster.
At the same time JIT doesn't perform all possible optimizations (yet?) as we've seen with a specialized case of `fma` operation,
and one has to be very careful when multiplying vectors by a factor, because it makes quite a difference.

The tricks like casting numerical arrays to arrays of vectors and alignments not only simplify the code and make it more robust and clear,
but also leverage mechanical sympathy. So let's start writing a bit more machine-friendly code and enjoy the results!


<table class="bmk">
<thead><tr><th>Method</th><th>Job</th><th>N</th><th>   Mean</th><th>Error</th><th>StdDev</th><th>StdErr</th><th>    Min</th><th>     Q1</th><th> Median</th><th>     Q3</th><th>    Max</th><th>   Op/s</th><th>Ratio</th><th>RatioSD</th><th>Gen 0/1k Op</th><th>Gen 1/1k Op</th><th>Gen 2/1k Op</th><th>Allocated Memory/Op</th>
</tr></tr><tr><td>For</td><td>Core</td><td>4</td><td>3.727 ns</td><td>0.0384 ns</td><td>0.0359 ns</td><td>0.0093 ns</td><td>3.682 ns</td><td>3.707 ns</td><td>3.718 ns</td><td>3.751 ns</td><td>3.800 ns</td><td>268,323,787.9</td><td>1.00</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>Vector</td><td>Core</td><td>4</td><td>2.363 ns</td><td>0.0137 ns</td><td>0.0121 ns</td><td>0.0032 ns</td><td>2.343 ns</td><td>2.356 ns</td><td>2.364 ns</td><td>2.366 ns</td><td>2.387 ns</td><td>423,166,784.1</td><td>0.63</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>VectorSpans</td><td>Core</td><td>4</td><td>2.437 ns</td><td>0.0111 ns</td><td>0.0098 ns</td><td>0.0026 ns</td><td>2.420 ns</td><td>2.431 ns</td><td>2.435 ns</td><td>2.444 ns</td><td>2.458 ns</td><td>410,359,292.0</td><td>0.65</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>Axpy</td><td>Core</td><td>4</td><td>23.044 ns</td><td>0.0560 ns</td><td>0.0496 ns</td><td>0.0133 ns</td><td>22.958 ns</td><td>23.014 ns</td><td>23.036 ns</td><td>23.071 ns</td><td>23.140 ns</td><td>43,394,548.5</td><td>6.19</td><td>0.06</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>For</td><td>Core</td><td>103</td><td>69.797 ns</td><td>0.1113 ns</td><td>0.1041 ns</td><td>0.0269 ns</td><td>69.611 ns</td><td>69.729 ns</td><td>69.780 ns</td><td>69.865 ns</td><td>70.045 ns</td><td>14,327,201.8</td><td>1.00</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>Vector</td><td>Core</td><td>103</td><td>30.588 ns</td><td>0.1315 ns</td><td>0.1230 ns</td><td>0.0318 ns</td><td>30.453 ns</td><td>30.512 ns</td><td>30.534 ns</td><td>30.679 ns</td><td>30.866 ns</td><td>32,692,121.7</td><td>0.44</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>VectorSpans</td><td>Core</td><td>103</td><td>26.251 ns</td><td>0.0343 ns</td><td>0.0286 ns</td><td>0.0079 ns</td><td>26.193 ns</td><td>26.245 ns</td><td>26.249 ns</td><td>26.268 ns</td><td>26.310 ns</td><td>38,094,025.8</td><td>0.38</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>Axpy</td><td>Core</td><td>103</td><td>26.805 ns</td><td>0.0638 ns</td><td>0.0533 ns</td><td>0.0148 ns</td><td>26.701 ns</td><td>26.783 ns</td><td>26.803 ns</td><td>26.831 ns</td><td>26.910 ns</td><td>37,305,908.8</td><td>0.38</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>For</td><td>Core</td><td>1003</td><td>614.265 ns</td><td>1.9209 ns</td><td>1.6041 ns</td><td>0.4449 ns</td><td>612.373 ns</td><td>612.876 ns</td><td>614.061 ns</td><td>615.123 ns</td><td>618.375 ns</td><td>1,627,963.2</td><td>1.00</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>Vector</td><td>Core</td><td>1003</td><td>255.404 ns</td><td>0.4940 ns</td><td>0.4621 ns</td><td>0.1193 ns</td><td>254.778 ns</td><td>255.094 ns</td><td>255.329 ns</td><td>255.930 ns</td><td>256.335 ns</td><td>3,915,359.8</td><td>0.42</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>VectorSpans</td><td>Core</td><td>1003</td><td>210.470 ns</td><td>0.3816 ns</td><td>0.3382 ns</td><td>0.0904 ns</td><td>209.864 ns</td><td>210.221 ns</td><td>210.446 ns</td><td>210.580 ns</td><td>211.158 ns</td><td>4,751,267.2</td><td>0.34</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>Axpy</td><td>Core</td><td>1003</td><td>110.691 ns</td><td>0.2618 ns</td><td>0.2186 ns</td><td>0.0606 ns</td><td>110.399 ns</td><td>110.473 ns</td><td>110.689 ns</td><td>110.879 ns</td><td>110.996 ns</td><td>9,034,150.3</td><td>0.18</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>For</td><td>Core</td><td>100003</td><td>60,780.472 ns</td><td>148.1639 ns</td><td>131.3434 ns</td><td>35.1030 ns</td><td>60,565.275 ns</td><td>60,668.842 ns</td><td>60,819.284 ns</td><td>60,897.133 ns</td><td>60,966.039 ns</td><td>16,452.7</td><td>1.00</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>Vector</td><td>Core</td><td>100003</td><td>31,056.083 ns</td><td>164.5393 ns</td><td>145.8598 ns</td><td>38.9827 ns</td><td>30,897.109 ns</td><td>30,965.186 ns</td><td>30,998.533 ns</td><td>31,125.262 ns</td><td>31,459.747 ns</td><td>32,199.8</td><td>0.51</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>VectorSpans</td><td>Core</td><td>100003</td><td>32,314.710 ns</td><td>151.4383 ns</td><td>141.6555 ns</td><td>36.5753 ns</td><td>32,134.773 ns</td><td>32,228.085 ns</td><td>32,260.626 ns</td><td>32,414.202 ns</td><td>32,587.829 ns</td><td>30,945.7</td><td>0.53</td><td>0.00</td><td>-</td><td>-</td><td>-</td><td>-</td>
</tr><tr><td>Axpy</td><td>Core</td><td>100003</td><td>5,178.782 ns</td><td>103.2439 ns</td><td>241.3293 ns</td><td>29.9332 ns</td><td>4,488.865 ns</td><td>5,050.068 ns</td><td>5,193.735 ns</td><td>5,336.972 ns</td><td>5,628.889 ns</td><td>193,095.6</td><td>0.08</td><td>0.01</td><td>-</td><td>-</td><td>-</td><td>-</td>

</tr></tbody></table>


## Links

1. [Sources and benchmark results](https://github.com/luajalla/Experimental)
2. [BenchmarkDotNet](https://benchmarkdotnet.org/index.html) and its [DisassemblyDiagnoser](https://benchmarkdotnet.org/articles/features/disassembler.html)
3. [Span](https://adamsitnik.com/Span/)
4. [SIMD-enabled types](https://docs.microsoft.com/en-us/dotnet/standard/numerics)
5. [SIMD in C#](https://instil.co/2016/03/21/parallelism-on-a-single-core-simd-with-c/)
6. [Using .NET hardware intrinsics API to accelerate Machine Learning scenarios](https://blogs.msdn.microsoft.com/dotnet/2018/10/10/using-net-hardware-intrinsics-api-to-accelerate-machine-learning-scenarios/)
7. [BLAS routines](https://software.intel.com/en-us/mkl-developer-reference-c-blas-and-sparse-blas-routines)
8. [Intel AVX2](https://software.intel.com/en-us/articles/how-intel-avx2-improves-performance-on-server-applications)
9. [FMAs in CUDA C](https://docs.nvidia.com/cuda/floating-point/index.html#fused-multiply-add-fma)




