---
title: Trying out Deedle with Bones and Regression
date: 2013-11-02
tags: [ fsharp, data ]
categories: [ fsharp ]
---
I usually don't need to run a regression anywhere, but it's kind of chasing me recently, starting with the Asset Pricing class and several variations of returns regressions (signed up to look at the familiar things from a different point of view... well, I definitely succeeded: have you ever thought about drawing the returns, prices and discount factors in space, all at once? [^1]. But I 'cheated' and completed the assignments with R.  

Though that was only the beginning - my cousin, MD student, was measuring the deflection of bones and other samples with different loads. And this time I decided to try out <a href="http://bluemountaincapital.github.io/Deedle/" title="Deedle: Exploratory data library for .NET" target="_blank">Deedle</a> and help her to explore the data.  

*Hint for Mono users: if XS doesn't load the main Deedle project, you can manually update the fsproj file (delete the reference to FSharp.Core), reload it, add references to Math.NET and FSharp.Data libs - it'll work nicely.*  

Let's start with loading experimental data. No more string splits and manual parsing!  

{{< highlight fsharp >}}
#r "FSharp.Data.dll"
#load "Deedle.fsx"

open System
open Deedle
// load data from a csv file
let ds = Frame.ReadCsv(__SOURCE_DIRECTORY__ + "/deflection.csv")
val ds : Frame<int,string> = 
        p         sample type length<mm> width<mm> height<mm> deflection<mm> 
  0  -> 0.98      bone        47         7.3       4.4        0.06 
  1  -> 1.96      bone        47         7.3       4.4        0.12 
{{< /highlight >}}


Then we checked out some properties of different samples groups - say, the average sample deformation. For simplicity we'll use only the bones group, load ("p") and deflection columns.  

{{< highlight fsharp >}}
// choose several columns and group the data by sample type
let bySample = 
    ds.Columns.[["sample type"; "p"; "deflection<mm>"]] 
    |> Frame.groupRowsByString "sample type"
val bySample : Frame<(string * int),string> =
                  sample type p         deflection<mm> 
  bone      0  -> bone        0.98      0.06           
            1  -> bone        1.96      0.12           
  ...      ...    ...         ...                      
  duralumin 6  -> duralumin   0.98      0.03           
  ...      ...    ...         ...                      

// average deflection by sample type
bySample.Columns.[["sample type"; "deflection<mm>"]] |> Frame.meanLevel Pair.get1Of2
val it : Frame<string,string> =
               deflection<mm>    
  bone      -> 0.228333333333333 
  duralumin -> 0.116666666666667 
  ...       -> ...               

// select the data for bones
let bones = (Frame.nest bySample).["bone"]
val bones : Frame<int,string> =
       sample type p         deflection<mm> 
 0  -> bone        0.98      0.06           
 1  -> bone        1.96      0.12           
 ...-> ...         ...       ...    
{{< /highlight >}}

You may notice that some of the values in the table are missing (the handwriting can be completely unparsable!), by default they are omited, but we can always specify how we want this data to be filled using `Direction` or a custom function.  

{{< highlight fsharp >}}
// note that there're missing values in this dataset
let deflections = bones?``deflection<mm>``
val deflections : Series<int,float> =
 0  -> 0.06      
 1  -> 0.12      
 ...-> ...       
 12 -> <missing> 
Series.mean deflections
 val it : float = 0.2283333333 
// omit missing values
deflections |> Series.dropMissing |> Series.mean
 val it : float = 0.2283333333 
// fill missing values by copying forward
deflections |> Series.fillMissing Direction.Forward |> Series.mean
 val it : float = 0.2542857143 
{{< /highlight >}}

Now let's check if there's any relation between the deflection and load. In theory, it's supposed to be linear and we're going to test that with a linear regression.  

{{< highlight fsharp >}}
/// Find slope and intercept with linear regression
let linearRegression xs ys = (...)
// drop rows with missing values
let bonesreg = Frame.dropSparseRows bones
val bonesreg : Frame<int,string> =
        sample type p    deflection<mm> 
   0 -> bone        0.98 0.06           
 ... -> ...         ...  ...            
   5 -> bone        5.88 0.41           

let load = Series.values bonesreg?p
let defl = Series.values bonesreg?``deflection<mm>``
let slope, intercept = linearRegression load defl   
val slope : float = 0.07142857143 
val intercept : float = -0.01666666667 
{{< /highlight >}}


![Chart](/images/deedle_chart_small.png#floatright)

Does this line make a good fit? Here is a chart with a couple of samples from this dataset.  

On the other hand a classical metric like R^2 can help to answer this question too, especially when it's extremely simple to add a new column to the dataframe and perform some operations.  

The new library is tried out, the lab is completed - everyone is happy ^_^  

{{< highlight fsharp >}}
bonesreg?prediction <- intercept + slope * bonesreg?p 
bonesreg?residualsq <- bonesreg?prediction - bonesreg?``deflection<mm>`` 
                       |> Series.mapValues (fun x -> x*x)
bonesreg
val it : Frame<int,string> =
       sample type p    deflection<mm> prediction         residualsq           
  0 -> bone        0.98 0.06           0.0533333333333334 4.44444444444441E-05 
  1 -> bone        1.96 0.12           0.123333333333333  1.11111111111111E-05 
 ...-> bone        ...  ...            ...                ...                  

let sdvs = Frame.sdv bonesreg
val sdvs : Series<string,float> =
  sample type    -> <missing>           
  p              -> 1.83341211951923    
  deflection<mm> -> 0.131059782796503   
  prediction     -> 0.130958008537088   
  residualsq     -> 1.72132593164778E-05

// compute the metrics:
let rsquare = let x = sdvs.["prediction"] / sdvs.["deflection<mm>"] in x * x
val rsquare : float = 0.9984475063 
let df = Frame.countRows bonesreg - 2 |> float
val df : float = 4.0 
let tvalue = sqrt (rsquare / (1. - rsquare) * df)
val tvalue : float = 50.71981861 
let se = (Series.sum bonesreg?residualsq) / df |> sqrt
val se : float = 0.005773502692 
{{< /highlight >}}
   

Yes, it's that simple.  

<br/><br />
<br/><br />

[^1]: to be honest, still don't get why anyone would need that, maybe that's the point where being a PhD helps?  
