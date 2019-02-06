---
title: "Reshaping Arrays in .NET"
categories: [ interop ]
tags: [ .net, matlab, fsharp ]
date: 2015-02-21
---
Reshape is one of these little functions, which look so simple and straightforward that you don't think much about them. It's quite useful and, I'm pretty sure, familiar to everyone who's written something in a language like Matlab. However, in other languages it might become a bit tricky.

### Question
I asked a bunch of people how they would write [reshape](http://ch.mathworks.com/help/matlab/ref/reshape.html) in a .NET language. That's not really a problem when you know the exact type and dimensionality, e.g. when you *always* convert 2D-array to 3D, but what if you don't? Given the limitations of generic constraints, the only requirement to the output (in addition to correctness, of course) was an ability to get the full information about the underlying types. And that's where the challenge starts...    

### Requirements

Let's take a look at a somewhat simplified problem - converting a single-dimensional array into a multidimensional one. My usecase was reading the files in .mat binary format as a part of the [type provider](https://github.com/luajalla/matprovider): read the bytes, figure out the type and if it's supposed to be an array - make it an array. Type conversions are not relevant for reshaping itself, so we can skip that for now.  

- The input is a single-dimensional array of any type and another array with dimension sizes;
- The output is another array, properly reshaped, where the type information is preserved (not necessary explicitly);
- Use standard types if possible.

### Meet System.Array
Unfortunately, that's not the case for fancy type signatures, so we'll go with a plain old `System.Array`.

The simplest example is 1D -> 1D "conversion", it's possible to get the type information or cast the output to a real array type, whether it's ````int[]````, ````string[]````, ````float[][]```` or anything else:  

{{< highlight fsharp >}}
let f (xs: System.Array) = xs

let xs = [| 1;2;3;4;5 |]
let ys = f xs     // val ys : System.Array = [|1; 2; 3; 4; 5|]
ys.GetType().Name // "Int32[]"
{{< /highlight >}}

Great! The next step is to add a dimension:  

{{< highlight fsharp >}}
let f2 (xs: System.Array) = [| xs; xs |]
let ys = f2 xs // val ys : System.Array [] = [|[|1; 2; 3; 4; 5|]; [|1; 2; 3; 4; 5|]|]
{{< /highlight >}}

This one doesn't work for us because of its return type, so:  

{{< highlight fsharp >}}
let f3 (xs: Array) = [| xs; xs |] :> Array;;
let ys = f3 xs    // val ys : Array = [|[|1; 2; 3; 4; 5|]; [|1; 2; 3; 4; 5|]|]
ys.GetType().Name // "Array[]"
{{< /highlight >}}

And the same output with `Array.init` methods:  

{{< highlight fsharp >}}
let f4 (xs: Array) = Array.init 2 (fun _ -> xs) :> Array;;
let ys = f4 xs    // val ys : System.Array = [|[|1; 2; 3; 4; 5|]; [|1; 2; 3; 4; 5|]|]
ys.GetType().Name // "Array[]"
{{< /highlight >}}


You also can't cast the result any more - `ys :?> int[][]` throws an exception:  

{{< highlight text >}}
System.InvalidCastException: Cannot cast from source type to destination type.
      at &lt;StartupCode$FSI_0017&gt;.$FSI_0017.main@ () [0x00000] in &lt;filename unknown&gt;:0 
      at (wrapper managed-to-native) System.Reflection.MonoMethod:InternalInvoke (System.Reflection.MonoMethod,object,object[],System.Exception&)
      at System.Reflection.MonoMethod.Invoke (System.Object obj, BindingFlags invokeAttr, System.Reflection.Binder binder, System.Object[] parameters, System.Globalization.CultureInfo culture) [0x00000] in &lt;filename unknown&gt;:0 
{{< /highlight >}}

And yes, of course, you *can* get the ints out of it:  

{{< highlight fsharp >}}
let arr = ys.GetValue 0 // val arr : obj = [|1; 2; 3; 4; 5|]
arr.GetType().Name      // val it : string = "Int32[]"
arr :?> int[]           // val it : int [] = [|1; 2; 3; 4; 5|]
{{< /highlight >}}

That's somewhat disappointing, because we do need something castable, with more or less concrete type. But it's quite intuitive: the argument is ````Array```` - the output is ````Array[]````, in the typed version ````'T[]```` - ````'T[][]````:  

{{< highlight fsharp >}}
let f5 (xs: int[]) = Array.init 2 (fun _ -> xs) :> Array
let ys = f5 xs    // val ys : System.Array = [|[|1; 2; 3; 4; 5|]; [|1; 2; 3; 4; 5|]|]
ys.GetType().Name // val it : string = "Int32[][]"
ys :?> int[][]    // val it : int [] [] = [|[|1; 2; 3; 4; 5|]; [|1; 2; 3; 4; 5|]|]
{{< /highlight >}}

or something like that:  
 
{{< highlight fsharp >}}
let f6 (xs: Array) = Array.init 2 (fun _ -> xs :?> int[]) :> Array
let ys = f6 xs    // val ys : Array = [|[|1; 2; 3; 4; 5|]; [|1; 2; 3; 4; 5|]|]
ys.GetType().Name // val it : string = "Int32[][]"  
{{< /highlight >}}

At this point  we're back to `System.Array` and its method `CreateInstance`:  

{{< highlight fsharp >}}
let f7 (xs: Array) =                   
    let t  = xs.GetType()                                                
    let ys = Array.CreateInstance(t, 2)
    for i in 0..1 do ys.SetValue(xs, i)
    ys 
let ys = f7 xs    // val ys : Array = [|[|1; 2; 3; 4; 5|]; [|1; 2; 3; 4; 5|]|]
ys.GetType().Name // val it : string = "Int32[][]"  
{{< /highlight >}}


### Reshape  

So this is the way to go. For an arbitrary number of dimensions we can use the same set of functions, the only thing left is filling the array with actual values. 

{{< highlight fsharp >}}
// init array of specific type
let fill t (len: int) f =
    let xs = Array.CreateInstance(t, len)
    for i in 0..len - 1 do xs.SetValue(f i, i)
    xs

// prod of prev dimensions: [|2; 3; 4|] -> [|1; 2; 6; 24|]
let inline prods dims = Array.scan ((*)) 1 dims 

// all array types from T[][]..[] to T[]
let types (t: Type) n =
    (t, 0)
    |> Seq.unfold (fun (t, i) -> if i = n then None else Some (t, (t.MakeArrayType(), i + 1)))
    |> Seq.toArray
    |> Array.rev

{{< /highlight >}}

For simplicity let's assume that all inputs are already nice and valid (no nulls, the product of dimensions is equal to the length of array and so on), then ````reshape```` might look like:

{{< highlight fsharp >}}
let reshape (arr: Array) (dims: int[]) =
    let t  = arr.GetType().GetElementType()
    let ts = types t dims.Length

    let rec init dim k =
        if dim = dims.Length - 1 then 
            fill ts.[dim] dims.[dim] (fun i -> arr.GetValue (k * dims.[dim] + i))
        else         
            fill ts.[dim] dims.[dim] (fun i -> init (dim+1) (k * dims.[dim] + i))
    init 0 0

{{< /highlight >}}


This function initializes the elements sequentially:

{{< highlight fsharp >}}
reshape [|1..12|] [|2;3;2|]                                                   
// val it : Array =
//   [|[|[|1; 2|]; [|3; 4|]; [|5; 6|]|]; [|[|7; 8|]; [|9; 10|]; [|11; 12|]|]|]
reshape [|1..12|] [|2;6|];;  
// val it : Array = [|[|1; 2; 3; 4; 5; 6|]; [|7; 8; 9; 10; 11; 12|]|]
reshape [|1..12|] [|1;1;12|];;                                                   
// val it : Array = [|[|[|1; 2; 3; 4; 5; 6; 7; 8; 9; 10; 11; 12|]|]|]
(reshape [|1..12|] [|2;3;2|]).GetValue 1 :?> int[][]
// val it : int [] [] = [|[|7; 8|]; [|9; 10|]; [|11; 12|]|]
{{< /highlight >}}

However, that's not how the original function works. For example (you can try it out in [octave-online](http://octave-online.net/)),


{{< highlight m >}}
squeeze(reshape((1:12),[2 3 2])(2,:,:))
    ans =
        2    8
        4   10
        6   12
{{< /highlight >}}

It might take a while to get a feeling how the arrays are reshaped, just run some examples and check where the numbers go. Meanwhile here's a function which does what's required:  

{{< highlight fsharp >}}
let reshape (arr: Array) (dims: int[]) = 
    let t  = arr.GetType().GetElementType()

    let ps = prods dims
    let ts = types t dims.Length

    let rec init dim k =
        if dim = dims.Length - 1 then
            fill ts.[dim] dims.[dim] (fun i -> arr.GetValue (ps.[dim] * i + k))
        else 
            fill ts.[dim] dims.[dim] (fun i -> init (dim+1) (ps.[dim] * i + k))
    init 0 0  
    
(reshape [|1..12|] [|2;3;2|]).GetValue 1 :?> int[][];;                             
// val it : int [] [] = [|[|2; 8|]; [|4; 10|]; [|6; 12|]|]
{{< /highlight >}}

In the end we do get a new array together with its type information, so the type provider can help you to avoid the casts from `System.Array`.  

