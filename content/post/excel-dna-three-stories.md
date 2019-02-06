---
layout: post
title: "Excel-DNA: Three Stories"
date: 2013-08-05
tags: [ fsharp, Excel, R ]
categories : [ fsharp ]
comments:
- id: 202
  author: 'F# Weekly #31 2013 | Sergey Tihon&#039;s Blog'
  author_email: ''
  author_url: http://sergeytihon.wordpress.com/2013/08/05/f-weekly-31-2013/
  date: '2013-08-05 05:18:28 +0200'
  date_gmt: '2013-08-05 05:18:28 +0200'
  content: '[...] Natallie Baikevich ‏posted &#8220;Excel-DNA: Three Stories&#8220;.
    [...]'
- id: 212
  author: Lyndsy Simon
  author_email: lyndsy@lyndsysimon.com
  author_url: http://lyndsysimon.com
  date: '2013-08-05 15:01:23 +0200'
  date_gmt: '2013-08-05 15:01:23 +0200'
  content: "<blockquote>The code is of unknown origin and protected, so no way to
    check it and I just gave up.</blockquote>\r\n\r\nYikes!\r\n\r\nPlease don't run
    code from unknown sources. That code could have busily been attempting to gain
    access to every system on your network for 24 hours, happily installing rootkits
    along the way."
- id: 222
  author: luajalla
  author_email: lu-a-jalla@ya.ru
  author_url: ''
  date: '2013-08-05 19:40:27 +0200'
  date_gmt: '2013-08-05 19:40:27 +0200'
  content: Unknown here doesn't mean randomly downloaded/untrusted, just that I don't
    know who wrote it ) Most probably there's a bug inside.
- id: 232
  author: F# and R in Excel | Excel-DNA
  author_email: ''
  author_url: http://excel-dna.net/2013/08/05/f-and-r-in-excel/
  date: '2013-08-05 21:39:19 +0200'
  date_gmt: '2013-08-05 21:39:19 +0200'
  content: '[...] F# and R in Excel [...]'
- id: 242
  author: niggler
  author_email: nirk.niggler@gmail.com
  author_url: ''
  date: '2013-10-10 03:42:16 +0200'
  date_gmt:   '2013-10-10 03:42:16 +0200'
  content: 'You can break VBA protection fairly easily with a hex editor: http://blog.nig.gl/post/63428658404/excel-vba-password-protection-is-useless'
- id: 252
  author: luajalla
  author_email: lu-a-jalla@ya.ru
  author_url: ''
  date: '2013-10-10 15:28:28 +0200'
  date_gmt: '2013-10-10 15:28:28 +0200'
  content: Thanks - was too lazy to look for that :) anyway, fixing VBA is the last
    thing you want to do when there's a simpler solution.
- id: 292
  author: Gerald W
  author_email: gerald.wluka@statfactory.co.uk
  author_url: http://www.fcell.cio
  date: '2014-02-17 18:35:48 +0100'
  date_gmt: '2014-02-17 18:35:48 +0100'
  content: If you want a powerful, robust, commercial solution (integrate .NET into
    Excel spreadsheets, with F#) take a look at Fcell.io
---
### Intro: Simulation

A couple of days ago I found a spreadsheet, potentially quite an interesting one. In theory, it should run a simple simulation (50000 paths by default) written in VBA - I'd say it's several minutes of work in the worst case. However, it took slightly more. One working day, to be precise. I still have no idea what it tried to do, why it ate 97% of CPU and even what exactly it computed, because all that ended with a weird error and crashed Excel. The code is of unknown origin and protected, so no way to check it and I just gave up.  

Now think about an average Excel user. He/she doesn't care about how exactly it works, in which language written or how difficult it was for you to implement. What is important then?  

- *functionality*: everything what comes to mind can become a function;  

- *simplicity*: nobody wants to write VBA (and usually anything else too), but it's always nice to have some useful UDFs at hand;  

- *reliability*: you have a well-tested library already, so why not to call it instead of rewriting in a poor language?  

- *performance*: why wait for a day when there're highly optimized libraries and distributed computing is already invented?  

For those who hasn't tried it yet - check out <a title="Excel-DNA home" href="http://exceldna.codeplex.com/" target="_blank">Excel-DNA</a>. It's an open-source project, which allows to integrate .NET into Excel. And yes, actually not only .NET ;) Documentation, samples, links to related posts can be found on the project page.  

{{% center %}}
![Spreadsheet](/images/dilbert_spreadsheet.gif)
{{% /center %}}

<p>&nbsp;</p>


### Keep Simple Things Simple  

Let's take log-linear interpolation as an example. Simple? Sure. We want to keep calculations clear in case if someone'd like to modify it - took 9 rows for me: functions like `OFFSET` and `MATCH` are not very friendly. And you still need to remember about sorting, extrapolation, duplicate values. What if there're thousands of points?  

Instead we call a UDF, which does all the work, it behaves in a specified way and can be reused in the future. I won't describe how to create an addin - for more information check the project <a title="Excel-DNA home" href="http://exceldna.codeplex.com/" target="_blank">homepage</a>. Briefly, in addition to a standard F# project we need a reference to *ExcelDna.Integration.dll*, *interpolation.dna* file with a reference to our library and a copy of *ExcelDna.xll* (x64 version is also available), renamed to *interpolation.xll*.  

The future UDF has `ExcelFunction` attribute [^1]. For consistency with Excel function names it's called `LOGLINEAR`. You can also add a description, category and so on. All parameters and output here are float arrays.  

{{< highlight fsharp >}}
open ExcelDna.Integration

[<AutoOpen>]
module Interpolation =

    let private interpolate curve (point : float) =
        let len = Array.length curve
        
        let u =
            if   point < fst curve.[0]       then 1
            elif point > fst curve.[len - 1] then len - 1
            else Array.findIndex (fun (v, _) -> point < v) curve
        
        let (xu, yu), (xd, yd) = curve.[u], curve.[u-1]
        let lnp = log yd + (log yu - log yd) / (xu - xd) * (point - xd)
        exp lnp

    [<ExcelFunction(Name="LOGLINEAR", Description="Log-Linear Interpolation",
Category="Custom", IsThreadSafe = true)>]
    let loglinear (xs: _[]) (ys: _[]) points =
        let curve =
            Seq.zip xs ys
            |> Seq.distinctBy fst
            |> Seq.sort
            |> Seq.toArray

        if curve.Length < 2 then failwith "at least 2 points are required"
        Array.map (interpolate curve) points
{{< /highlight >}}

After loading the add-in in Excel (double-click on *interpolation.xll* in output folder), you can just type `LOGLINEAR` and see the results![^2].  

{{% center %}}
![Excel-Interpolation](/images/interpolation.png)
{{% /center %}}

<p>&nbsp;</p>  


### Make Complex Things Simple  

Well, it's cool, but I still can do this interpolation by hands - and it'll work, you don't need to be super-smart to do a couple of subtractions and multiplications without mistakes. But things are getting more interesting when a bunch of standard functions is not enough.  

How about machine learning with Excel? Of course, it can be handy only for experiments with adequately small amounts of data. But for sure, not something I'd want to write in VBA.  

A lot of my fellow traders use random forests for feature selection (it is actually what I like about RF most) before feeding the data into actual models, neural nets or anything else. So let's take a look at *rtp* addin, which can help to find important features and get rid of useless (and potentially harmful?) ones. Important point: this example is for demo purposes only! I took some Yahoo stock prices - it's not that easy to get good data for free, so can't call it realistic; the same works for features[^3]. If you want a bit more for free, there're also two old kaggle competitions: <a title="Benchmark Bond Trade Price Challenge" href="https://www.kaggle.com/c/benchmark-bond-trade-price-challenge" target="_blank">Benchmark Bond Trade Price Challenge</a>  and <a title="Algorithmic Trading Challenge" href="http://www.kaggle.com/c/AlgorithmicTradingChallenge" target="_blank">Algorithmic Trading Challenge</a>  (the winners <a title="Winning Algorithmic Trading Challenge" href="http://sugiyama-www.cs.titech.ac.jp/~sugi/2013/Kaggle.pdf" target="_blank">used RF</a>).  

Anyway, we are lucky because all data is numeric. Even more than that - we have <a title="R Type Provider" href="https://github.com/BlueMountainCapital/FSharpRProvider" target="_blank">F# R Type Provider</a> and can use R's randomForest package!  

{{< highlight fsharp >}}
open RProvider
open RProvider.``base``
open RProvider.randomForest

open ExcelDna.Integration

[<ExcelFunction(Name = "IMPORTANCE", 
    Description = "Variable importance measures as produced by R's randomForest ")>]
let importance (names : obj[]) (values: float[,]) (ys : float[]) =
    let cols = min names.Length (values.GetLength 1)
    let rows = min ys.Length    (values.GetLength 0)

    let xs = 
        names
        |> Seq.take cols
        |> Seq.mapi (fun j name ->
            string name, Array.init rows (fun i -> values.[i, j]))
        |> namedParams
        |> R.data_frame            

    let rf = R.randomForest(xs, ys)
    (R.importance rf).Value
{{< /highlight >}}

Look at the R-Importance tab: we call the function `{=IMPORTANCE(Names,Data,Result)}` and get a nice set of values - the greater the value, the greener and more important it is. For example, P5 and P6 (close prices for day-5 and day-6 respectively) seem to be not very useful. 

{{% center %}}
![Spreadsheet](/images/excel-importance.png)
{{% /center %}}


Adding features is pretty straightforward too. Say, we want to calculate Simple Moving Average with different offsets:  

{{< highlight fsharp >}}
[<ExcelFunction(Name = "SMA", Description="Simple Moving Average")>]
let sma (values : float[]) (nobj : float) =
    let n   = int nobj
    let len = values.Length

    if len < n || n < 2 then box ExcelError.ExcelErrorNA
    else 
        let res = Array2D.zeroCreate len 1

        Seq.windowed n values
        |> Seq.map Seq.average
        |> Seq.iteri (fun i v -> res.[i + n - 1, 0] <- box v)
        resize res
{{< /highlight >}}


The interesting thing here is `resize` function: it allows you not to select the whole output range when calling a function, but just type `=SMA(Values,5)` in the top cell and result array is automatically resized. The code of these examples is available on <a title="Excel-DNA Samples" href="https://github.com/luajalla/everything-fun/tree/master/exceldna-sample" target="_blank">github</a>.  

<iframe src="https://docs.google.com/spreadsheet/pub?key=0AsEtPrcHNCbXdGFvRjRhYkwxYkhVWmxyZWxxV3dJemc&amp;output=html&amp;widget=true" height="400" width="720" frameborder="0"></iframe>  

Excel-DNA makes it possible to bring all .NET power to the spreadsheets. Just try it ^_^  

<p>&nbsp;</p>
<p>&nbsp;</p>

[^1]: see *interpolation* project <a title="Interpolation" href="https://github.com/luajalla/everything-fun/tree/master/exceldna-sample/interpolation" target="_blank">here</a>.
[^2]: as a reminder, array formulae are entered with Ctrl+Shift+Enter.
[^3]: and also my experience has nothing to do with trading.
