---
title: "FsCheck and co: Testing Excel Financial Functions"
date: 2014-03-06
tags: [ mono, fsharp, Excel, fscheck, testing ]
categories: [ fsharp ]
comments:
- id: 452
  author: Kurt
  author_email: kurt.schelfthout@gmail.com
  author_url: http://fortysix-and-two.blogspot.co.uk/
  date: '2014-03-07 08:25:30 +0100'
  date_gmt: '2014-03-07 08:25:30 +0100'
  content: Did you try writing FsCheck generators? If you had problems with argument
    exhaustion it'll make your tests faster and more readable.
- id: 462
  author: luajalla
  author_email: lu-a-jalla@ya.ru
  author_url: ''
  date: '2014-03-07 16:29:49 +0100'
  date_gmt: '2014-03-07 16:29:49 +0100'
  content: Thanks for the hint! I was thinking about that, but didn't have time to
    try yet.
---

> Excel Financial Functions library is a .NET library written by Luca Bolognese that provides the full set of financial functions from Excel. It can be used from both F# and C# as well as from other .NET languages. The main goal for the library is compatibility with Excel, by providing the same functions, with the same behaviour.  

![Excel-Funcs-Logo](/images/excelfuncslogo.png#floatright)

The library was recently moved to GitHub and its new home is <a title="Excel Financial Functions" href="http://fsprojects.github.io/ExcelFinancialFunctions/" target="_blank">here</a>. The goal was to turn an archive with sources into a community project, including the build scripts, tests, documentation and a fancy homepage. And thanks to <a title="FAKE" href="http://fsharp.github.io/FAKE/" target="_blank">FAKE</a> and <a title="FSharp.Formatting" href="http://tpetricek.github.io/FSharp.Formatting/" target="_blank">FSharp.Formatting</a> that was much more fun than I had expected.  

So, what did we want to see in the new version?  
- the library should be as close as possible to the original;  
- the tests, running automatically during the build;  
- mono compatibility;  
- a couple of pages with nicely formatted docs;  
- (ideally) no new unnecessary dependencies.  

The library itself doesn't depend on the Excel interop libraries (available only in .NET), that means it can be used on mono right out of the box. The only problem was in checking the results against Excel: there was an amazing test suite in form of console app, which I wanted to leave. But how to modify it to match the new requirements?  

### Choosing the Framework  

The first step, of course, is to ask the community ^\_^ - what frameworks people like most and use, in what cases, how that worked for them. Thanks again to <a href="https://twitter.com/mausch" target="_blank">@mausch</a>, <a href="https://twitter.com/kurt2001" target="_blank">@kurt2001</a>, <a href="https://twitter.com/brandewinder" target="_blank">@brandewinder</a>, <a href="https://twitter.com/foxyjackfox" target="_blank">@foxyjackfox</a>, <a href="https://twitter.com/sergey_tihon" target="_blank">@sergey_tihon</a>, <a href="https://twitter.com/davefancher" target="_blank">@davefancher</a>, <a href="https://twitter.com/c4fsharp" target="_blank">@c4fsharp</a>, <a href="https://twitter.com/kitlovesfsharp" target="_blank">@kitlovesfsharp</a> and <a href="https://twitter.com/_____c" target="_blank">@_____c</a> for the feedback!  

But then I realized that there's no particular need in any additional frameworks: the functions are rather simple, you can compare floats or dates with NUnit, why add new dependencies? A single FsUnit-like ````shouldEqual```` is more than enough (yes, couldn't resist):  

{{< highlight fsharp >}}
[<Literal>]
let PRECISION = 1e-6

let inline shouldEqual msg exp act =
    Assert.AreEqual(exp, float act, PRECISION, msg)
{{< /highlight >}}

The only thing left was to implement the tests itself.  

*If you're choosing the framework too, I'd recommend to check the thread <a title="Question about Testing Frameworks" href="https://groups.google.com/forum/#!msg/fsharp-opensource/2v7PAJBvs2E/QWJ_faVJFcUJ" target="_blank">Question about Testing Frameworks</a> in the F# OpenSource list.*  

### The Search for Values  

And this is the point I stuck at: there're almost 200,000 of tests! If the values match Excel results, the test is passed... but we don't want to use Excel interop because of mono. Ok, let's just store the arguments and expected values. There're 52 functions to test, in general case the different functions have different number and/or types of parameters. What is the best way to read them from the files, parse, pass to tests, compare the actual and expected results?  

Here's the most concise solution I came up with, it's based on F#'s inlining (source is <a title="Tests" href="https://github.com/fsprojects/ExcelFinancialFunctions/blob/master/tests/ExcelFinancialFunctions.Tests/crosstests.fs" target="_blank">here</a>):  

{{< highlight fsharp >}}
[<Test>]  
let disc() = runTests "disc" parse6 Financial.Disc   

[<Test>]
let price() = runTests "price" parse8 Financial.Price
{{< /highlight >}}

The test data is read from the "price" file. The function has 8 arguments (luckily, it's a tuple for C# users convenience) and the compiler is able to infer all the types - in our case they're standard (````float````, ````DateTime````, ````int````...) and have ````TryParse```` static method.  

{{< highlight fsharp >}}
let inline parse str =
    let mutable res = Unchecked.defaultof<_>
    let _ = (^a: (static member TryParse: string * byref< ^a > -> bool) (str, &res))
    res

let inline parse3 [| a; b; c |] =
    (parse a, parse b), parse c 
{{< /highlight >}}

As you may notice, there're some duplications - we still need several `parseN` functions.  

Back to our minimalistic approach, the function `runTests` should be inlined too, so the compiler can infer the types. The error messages are pretty detailed: they contain index, function name, arguments, expected and actual results - everything you hopefully won't ever need to know :)  

{{< highlight fsharp >}}
let inline runTests fname parsef f =
    readTestData fname    
    |> Seq.iteri (fun i data ->
        let param, expected = parsef data
        let actual = f param
        shouldEqual (sprintf "%d - %s(%A)" i fname param) expected actual)
{{< /highlight >}}

*Original console tests are in the repo too - all tests pass (checked for Excel 2010).*  

### Testing the Properties  

Something new this version got is the tests of some properties, e.g. the modified duration shouldn't be greater than maturity, the cumulative accrint should be equal to the sum of accrints in corresponding coupon periods etc.  

The best way to do that is through randomized testing with <a title="FsCheck" href="https://github.com/fsharp/FsCheck" target="_blank">FsCheck</a>.  

{{< highlight fsharp >}}
[<Test>]
let ``tbill price is less than 100``() =
    fsCheck (fun (sd: DateTime) t disc' ->
        let md, disc = sd.AddDays (toFloat t * 365.), toFloat disc'

        tryTBillPrice sd md disc    
        ==>
        lazy (Financial.TBillPrice(sd, md, disc) - 100. < PRECISION))
{{< /highlight >}}

The most challenging part is expressing the properties in the way FsCheck can understand easily. For example, each function has ````tryX```` method to check the parameters. That's exactly what we need to be sure the random values make sense. However, when the logic behind such a function is not trivial, the arguments just get exhausted.  

First, the dates - when there're several dates and you know which one comes first, how many days can be between them etc, it makes sense to keep as date the first one only and define the others as ````date + n```` days (as in the example above).  

Second, converting ints to floats usually works better than straightforward generating of floats. Probably, there're other (and better) ways - for example, I was thinking about writing a smart generator both for dates and floats - prices, rates and so on, but the current approach worked quite well and solved the problem with arguments.  

#### Several community projects using FsCheck:  
- <a title="fsharpx" href="https://github.com/fsprojects/fsharpx/tree/master/tests/FSharpx.Tests" target="_blank">fsharpx</a>  
- <a title="Suave" href="https://github.com/SuaveIO/suave/blob/master/Tests/Program.fs" target="_blank">Suave</a>  
- <a title="RProvider" href="https://github.com/BlueMountainCapital/FSharpRProvider/blob/master/tests/Test.RProvider/Test.fs" target="_blank">RProvider</a>  

#### Bonus Reading:  
- <a title="Optimizing with Help of FsCheck" href="http://bugsquash.blogspot.com/2013/06/optimizing-with-help-of-fscheck.html" target="_blank">Optimizing with Help of FsCheck</a>  
- <a title="Gaining FsCheck Fluency through Transparency" href="http://jackfoxy.com/gaining-fscheck-fluency-through-transparency/" target="_blank">Gaining FsCheck Fluency through Transparency</a>  


### Why Does Anyone Care?  

This is the only library replicating Excel behavior, with a few differences, though, as explained <a title="Excel Compatibility" href="http://fsprojects.github.io/ExcelFinancialFunctions/compatibility.html" target="_blank">here</a>. Anyone who needs this functionality can use the library on any platform.  

What was also interesting is to compare the results with other apps. For example, spotted case when Excel for Mac didn't match Windows Excel results, that was a bit surprising. But in <a title="Open Office Diffs" href="http://fsprojects.github.io/ExcelFinancialFunctions/openofficediff.html" target="_blank">Libre Office</a> almost every function is more or less different!  

Some OO functions just returned errors for any input...  

<blockquote class="twitter-tweet" lang="en"><p><a href="https://twitter.com/lu_a_jalla">@lu_a_jalla</a> Ooohh, and under the Apache License as well.   Should not be too hard for us to port to C++.</p>
<p>&mdash; Apache OpenOffice (@ApacheOO) <a href="https://twitter.com/ApacheOO/statuses/425455963693658112">January 21, 2014</a></p></blockquote>
<p><script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>  
	
Seems like they can be fixed now ;)  
