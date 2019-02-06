---
title: Abs Puzzle
date: 2014-04-25
tags: [ fstar, discoveries, pex, .net, jvm ]
categories: [ languages ]
---
If someone asked you to implement abs function (say, for ints), how would you do that? A simple experiment shows that almost everyone comes up with something like that, whether it's F#, C#, Scala or anything else:  

{{< highlight fsharp >}}
let abs x = if x < 0 then -x else x
{{< /highlight >}}

The obvious question - do the answers match your expectations? That depends...  

<img src="/images/absx.jpg" alt="absx" style="text-align:center;margin-left:auto;margin-right:auto;display:block"/>

It's quite obvious when you think about the corner cases - here it's the minimal integer value:  

{{< highlight fsharp >}}
-(-2147483648) = -2147483648
{{< /highlight >}}

So, in Java/Scala the absolute value can be <a href="http://docs.oracle.com/javase/8/docs/api/java/lang/Math.html" title="Java Math" target="_blank">negative</a>:  

> Note that if the argument is equal to the value of Integer.MIN_VALUE, the most negative representable int value, the result is that same value, which is negative.  

In .NET you'll get an <a href="http://msdn.microsoft.com/en-us/library/dk4666yx%28v=vs.110%29.aspx" title=".NET Math.Abs" target="_blank">exception</a> - personally, I think that's a better behaviour for this function:  
`OverflowException - value equals Int32.MinValue`  

When I started to think about different examples for our future meetup (collecting all kinds of crashes on the way - from 'CLR detected an invalid program' to 'unexpected exception'), I couldn't resist but look at what happens when you move some checks to the type level.  

Let's define a refinement type for the natural numbers in F* and make sure it works (you can try the examples <a href="http://rise4fun.com/FStar" title="Try FStar" target="_blank">in your browser</a>):  

{{< highlight fsharp >}}
module Nums

type nat = x:int{x >= 0}

let good_x: nat = 42
let bad_x: nat = -42
input(6,17-6,20) : Error : Expected an expression of type: x_3:int{GTE x_3 0} but got
(op_Minus(42)): x_357_1:int{Eq2 int int x_357_1 (Minus (42))} Type checking failed: Nums
{{< /highlight >}}

Now it's turn for our `abs` function:  

{{< highlight fsharp >}}
val abs: int -> nat
let abs x = if x < 0 then -x

let _ = abs 2147483647  
let _ = abs (-2147483647) 

// both following statements give Syntax error:                                
let _ = abs 2147483648
let _ = abs (-2147483648)
{{< /highlight >}}

The first error is kind of expected, because the literal `2147483648` is too large for `int`. However, I'd think the second one should be ok, and after a quick look at the F* lexer not sure why they both fail (`"Allow &lt;max_int+1&gt; to parse as min_int"`). Anyway, there's another interesting case:  

{{< highlight fsharp >}}
let _ = abs (2147483647 + 1)
// And the result is...                                                         
Verified module: Nums
{{< /highlight >}}

A bit strange, isn't it? If you ask Scala or F# what `2147483647 + 1` is, you get `-2147483648`, that means `abs (2147483647 + 1)` can't be of ````nat```` type! Here's a bunch of checks for '+1' with different refinements:  

{{< highlight fsharp >}}
val check: x:nat -> y:int{y > x}
let check x = x + 1 

let _ = check (2147483647 + 1)
Error : Let-bound variable op_Addition(2147483647, 1) escapes its scope in type
x_8_1:int{GT x_8_1 x_11_3}; insert an explicit type annotation. (expected type none)
Type checking failed: Nums   
{{< /highlight >}}

{{< highlight fsharp >}}
val check: x:nat -> y:int{y = (x+1)} 
Error : Let-bound variable op_Addition(2147483647, 1) escapes its scope in type
x_6_1:int{Eq2 int int x_6_1 (Add (x_9_5) (1))}; insert an explicit type annotation.
(expected type none) Type checking failed: Nums
{{< /highlight >}}

{{< highlight fsharp >}}
val check: x:nat -> y:int{y > 0} 
Verified module: Nums        
{{< /highlight >}}

{{< highlight fsharp >}}
val check: x:nat -> y:int{y < 0 \/ y > x}
Error : Let-bound variable op_Addition(2147483647, 1) escapes its scope in type
x_8_1:int{((LT x_8_1 0) || (GT x_8_1 x_11_3))}; insert an explicit type annotation.
(expected type none) Type checking failed: Nums
{{< /highlight >}}

All the cases above succeeded for the max value:  

{{< highlight fsharp >}}
let _ = check 2147483647
{{< /highlight >}}

I also tried these examples on my old-slightly-working F* 0.7 alpha Mono build, where the results seem to be more reasonable:  

{{< highlight fsharp >}}
let _ = abs 2147483648 
Error : Expected an expression of type: int but got (2147483648.000000): float
Type checking failed: Nums
{{< /highlight >}}

{{< highlight fsharp >}}
let _ = abs (-2147483648) 
 starting backend
 starting derefinement
 starting expanding types
 finished with expanding types
Verified module: Nums    
{{< /highlight >}}

Maybe one day I'll finally setup a VM with Windows to take a look at IL, but for now the conclusion is to always be on the lookout.  

P.S. <a href="http://pexforfun.com/" title="Pex" target="_blank">Pex</a> can find the failing case (for some reason, at least in online version this example works only with C# and VB):  

{{< highlight csharp >}}
using System;
using System.Diagnostics.Contracts;

public class Program {
  public static void Puzzle(int x) {
    var y = x < 0 ? -x : x;
    Contract.Assert(y >= 0);
  }
}
x            | Output/Exception  | Error Message
-----------------------------------------------------------
0            |                   |
int.MinValue | ContractException | Assertion failed: y >= 0   
{{< /highlight >}}
