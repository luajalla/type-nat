---
title: Days and Ghost Refinements
date: 2013-07-07
tags: [ fstar, refinements, Excel ]
categories: [ languages ]
comments:
- id: 152
  author: 'F# Weekly #27 2013 | Sergey Tihon&#039;s Blog'
  author_email: ''
  author_url: http://sergeytihon.wordpress.com/2013/07/08/f-weekly-27-2013/
  date: '2013-07-07 21:05:06 +0200'
  date_gmt: '2013-07-07 21:05:06 +0200'
  content: '[...] Natallie Baikevich blogged &#8220;Days and Ghost Refinements&#8220;.
    [...]'
---
Let's look at the simple function, which calculates the number of days between dates, when there're 30 days in a month (and 360 in a year).  
  
Something like this F# code:  

{{< highlight fsharp >}}
let days360 sy sm sd ey em ed =
    (ey - sy) * 360 + (em - sm) * 30 + (ed - sd)
{{< /highlight >}}

We can even write a bunch of tests to be sure the function works:  

{{< highlight fsharp >}}
let tests = [
    DateTime(2012, 02, 29), DateTime(2013, 02, 28), 359
    DateTime(2012, 02, 29), DateTime(2012, 03, 01),   2
    DateTime(2012, 02, 29), DateTime(2013, 03, 01), 362
    DateTime(2012, 03, 01), DateTime(2013, 03, 01), 360
    DateTime(2011, 02, 28), DateTime(2013, 02, 28), 720
    DateTime(2012, 05, 31), DateTime(2012, 07, 31),  60
]

// true
tests |> Seq.forall(fun (s, e, res) ->
    days360 s.Year s.Month s.Day e.Year e.Month e.Day = res)
{{< /highlight >}}

As you see `days360` doesn't take .NET `DateTime` as parameters, but ints. Would be nice to check them. Say, a month can take value from 1 to 12 - so what we need is a refinement type. F* supports two types of refinements: concrete and ghost. Here is how they are defined in “Secure Distributed Programming with Value-Dependent Types” <a title="Secure Distributed Programming with Value-Dependent Types" href="http://research.microsoft.com/apps/pubs/default.aspx?id=141708" target="_blank">paper</a>:

> **Concrete refinements** are pairs representing a value and a proof term serving as a logical evidence of the refinement property, similar to those in Coq and Fine.  
<p>&nbsp;</p>

> **Ghost refinements** are used to state specifications for which proof terms are not maintained at run time. Ghost refinements have the form $x:t\\{\phi\\}$ where $x$ is a value variable, $t$ is a type, and $\phi$ is a logical formula, itself represented as a type that must have kind $E$ and may depend on $x$ and other in-scope variables. Ghost refinements provide the following benefits:  
 - they enable precise symbolic models for many cryptographic patterns and primitives, and evidence for ghost refinement properties can be constructed and communicated using cryptographic constructions, such as digital signatures;  
 - they benefit from a powerful subtyping relation: $x:t\\{\phi\\}$ is a subtype of $t$; this structural subtyping is convenient to write and verify higher-order programs;  
 - they provide precise specification to legacy code without requiring any modifications;  
 - when used in conjunction with concrete refinements, they support selective erasure and dynamic reconstruction of evidence, enabling a variety of new applications and greatly reducing the performance penalty for runtime proofs.


Here we use the numbers as an example, because they are simple and intersections/unions of types are obvious. Note that we don't verify that the dates are entirely valid. This is an artificial example - small enough to try it online =).

{{< highlight fsharp >}}
type year  = x:int{ 1900 <= x /\ x <= 9999 }
type month = x:int{    1 <= x /\ x <=   12 }
type day   = x:int{    1 <= x /\ x <=   31 }

val days360: year -> month -> day -> int

let days360 sy sm sd ey em ed =
    (ey - sy) * 360 + (em - sm) * 30 + (ed - sd)

let _ = days360 2013 6 1 2013 6 42
{{< /highlight >}}

We run F* - and get a type check time failure!  

{{< highlight fsharp >}}
input(12,8-12,34) : Error : Expected an expression of type:
x_5_1:int{((LTE 1 x_5_1) && (LTE x_5_1 31))}
but got (42): int
Type checking failed
{{< /highlight >}}

Well, 12 was supposed to be here:  

{{< highlight fsharp >}}
let _ = days360 2013 6 1 2013 6 12 // ok
{{< /highlight >}}

But wait, there're still 31-day months... and February. With EU 30 / 360 convention 31 is simply replaced with 30, so we modify the function:  

{{< highlight fsharp >}}
let adjDay d = if d = 31 then 30 else d

let days360 sy sm sd ey em ed =
    let sd, ed = adjDay sd, adjDay ed
    (ey - sy) * 360 + (em - sm) * 30 + (ed - sd)

// DateTime(2012, 04, 25), DateTime(2012, 07, 31), 95
// DateTime(2012, 06, 30), DateTime(2012, 07, 31), 30
// DateTime(2012, 02, 28), DateTime(2012, 03, 31), 32
// DateTime(2012, 02, 29), DateTime(2012, 03, 31), 31

{{< /highlight >}}

In this case start and end dates can't be greater than 30. Can we define that with types? Sure!  

{{< highlight fsharp >}}
type day30 = x:day{ x <> 31 }

val adjDay: day -> day30
let adjDay d = if d = 31 then 30 else d // try to leave only d here - it fails to typecheck
{{< /highlight >}}

What if we want to add another convention? US 30 / 360 handles EOM dates differently, so if a start date is greater than end date the function results can become inconsistent. So we expect start date to be less than (or equal to) end date. A type checker can verify this requirement too:  

{{< highlight fsharp >}}
val daysInMonth: year -> month -> day

val lastDay: year -> month -> day -> bool -> bool
//EU 30/360: the last day of Feb is not changed to 30
let lastDay y m d eu =
    if eu && (m = 2) then false
    else d = (daysInMonth y m)

let adjDay d last = if last then 30 else d

val days360: sy:year -> sm:month -> sd: day
    -> ey:year{ey >= sy}
    -> em:month{ey > sy \/ (ey = sy /\ em >= sm)}
    -> ed:day{ey > sy \/ em > sm \/ ed >= sd}
    -> bool
    -> int

let days360 sy sm sd ey em ed convEU =
    let slast, elast = lastDay sy sm sd convEU, lastDay ey em ed convEU in

    // EU: 31 -> 30
    // US: 31 -> 30; end of Feb -> 30
    let sd = adjDay sd slast in
    // EU: 31 -> 30
    // US: 31 -> 30 if sd is EOM; end of Feb -> 30 if sd was end of Feb too
    let checkEndDate = convEU || (slast && ((sm = 2) || not (em = 2))) in
    let ed = adjDay ed (elast && (convEU || checkEndDate)) in

    (ey - sy)*360 + (em - sm)*30 + (ed - sd)

let _ = days360 2013 7 6 2015 7 6 true
let _ = days360 2015 7 6 2013 7 6 true // error
{{< /highlight >}}

The last line gives the following error:  

{{< highlight fsharp >}}
input(44,8-44,29) : Error :
Expected an expression of type: x_23_6: Dates.year{GTE x_23_6 2013}
but got (2013): int
Type checking failed
{{< /highlight >}}

All these >, \/, /\, < look funny, if it's not enough – just imagine the effect of adding time components. Fortunately, the relation between the dates can be defined as a separate type:  

{{< highlight fsharp >}}
// start date <= end date
type LTE = fun sy sm sd ey em ed => 
    ey > sy \/ (ey = sy /\ (em > sm \/ (em = sm /\ ed >= sd)))

val days360: sy:year -> sm:month -> sd:day
    -> ey:year -> em:month -> ed:day{LTE sy sm sd ey em ed}
    -> bool
    -> int
{{< /highlight >}}

Now let's assume that a date is valid when it's not 2 / 30 or 2/ 31. We can define a logical function using `logic val` construct. Such functions can be used in refinements but not in code itself. The axioms are defined using the `assume` construct:  

{{< highlight fsharp >}}
logic val isValidDate: year -> month -> day -> bool
assume Valid: forall y m (d:day{m <> 2 \/ d < 30}). isValidDate y m d = true 

val days360: sy:year -> sm:month -> sd:day{isValidDate sy sm sd = true}
    -> ey:year -> em:month -> ed:day{isValidDate ey em ed = true /\ LTE sy sm sd ey em ed}
    -> bool
    -> int
{{< /highlight >}}

Well, quite enough of refinements for a start. There’s much more ways to apply them: for example, refinements with affine values are great for verification of stateful programs (see the “Secure multi-party sessions in F*” section in the paper). With rise4fun you may get a request timeout in more complex cases, so I’d recommend to download the F* package for your experiments.  

### References  

<strong>1.</strong> "Secure Distributed Programming with Value-Dependent Types" <a title="Secure Distributed Programming with Value-Dependent Types" href="http://research.microsoft.com/apps/pubs/default.aspx?id=141708" target="_blank">paper</a>.  
<strong>2.</strong> F* <a title="F* project home" href="http://research.microsoft.com/en-us/projects/fstar/" target="_blank">home page</a>.  
<strong>3.</strong> Rise4fun <a title="Rise4Fun - F* tutorial" href="http://rise4fun.com/FStar/tutorial/guide" target="_blank">tutorial</a>.  


### Bonus  

Time to get back to the good old tests: the obvious idea is to compare the results with Excel. We are interested in two functions: DAYS360 and YEARFRAC - multiplying the result with 360. Look at the following comparison (when the <a title="Days - Comparison" href="https://docs.google.com/spreadsheet/ccc?key=0AsEtPrcHNCbXdHQyb2JJQzgzNkgyREFlWF9ENEVTeEE&amp;usp=sharing" target="_blank">spreadsheet</a> was converted to Google Docs format, the results were changed, so I added separate rows with Excel output):  

<iframe src="https://docs.google.com/spreadsheet/pub?key=0AsEtPrcHNCbXdHQyb2JJQzgzNkgyREFlWF9ENEVTeEE&amp;single=true&amp;gid=0&amp;output=html&amp;widget=true" height="480" width="640" frameborder="0"></iframe>  

There’re obvious problems with DAYS360 when the dates are swapped (start date > end date) and end-of-Feb cases. But our function is different from YEARFRAC too.  

> Date adjustments rules for 30/360US:  
1. If the investment is EOM and D1, D2 are the last day of Feb, then D2 = 30.  
2. If the investment is EOM and D1 is the last day of Feb, then D1 = 30.  
3. If D2 is 31 and D1 is 30 or 31, then D2 = 30.  
4. If D1 is 31, then D1 = 30.  

Let the start and end dates be 2/29/2012 and 3/31/2012 respectively. The rules are applied in order:  
````D2 - D1 = D2 - 30 = 30 - 30 = 0````  

But in Excel the rule 3 goes before the rule 2 (our function can be simply modified to behave the same way: ````sm = 2 || em <> 2```` should be replaced with ````(sm = 2) = (em = 2)````):  
````D2 - D1 = 31 - D1 = 31 - 30 = 1````  

So we expect to see ````(2012 – 2012)*360 + (3 – 2)*30 + 0 = 30```` and not ````31````.  
F# version is available <a title="GitHub - days360.fsx" href="https://github.com/luajalla/everything-fun/blob/master/days360.fsx" target="_blank">here</a>.  
<p>&nbsp;</p>
<p>&nbsp;</p>
