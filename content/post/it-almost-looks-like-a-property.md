---
title: It looks almost like a property
date: 2014-01-26
tags: [ csharp ]
categories: [ languages ]
---
*Based on a true story. Only the names, places, and events have been changed.* 

![Character](/images/character.jpg#floatright)

Many people today forget that the tools are just tools. Doesn't matter if it's OO or functional programming, take any language: there're plenty of ways to write awesome and even more - to write terrifying code. 
`lets . compose . these . functions . until . your . type-checker . explodes . btw . did . you . save . everything?`  

Today I'll share a simple code-reducing trick, which I came up with a couple of years ago, but still like it for some reason (if you don't - just deal with it). So what is the problem?  

*Now answer the question - Do you believe there might be anything except the classes?*  
*(No)* No problem. At all. There's a class, with a bunch of fields. <a href="#theend">The end.</a>  
*(Yes)* Ok, let's say the objects of this class suppose to describe the weather conditions. Moving on,  

{{< highlight csharp >}}
class Weather
{
    public double Temp     { get; set; }
    public double Humidity { get; set; }
    public double Pressure { get; set; } 
    public double Rainmm   { get; set; }
}
{{< /highlight >}}

You may say, the temperature is usually reported as a range. Here we go:  

{{< highlight csharp >}}
class Weather
{
    public double MinTemp   { get; set; }
    public double MaxTemp   { get; set; }
    public double WindChill { get; set; } 
    ... // other properties
}
{{< /highlight >}}

Nice, but there're also different observations for the night/morning/afternoon/evening forecasts, which one may want to include into the same forecast:  

{{< highlight csharp >}}
class Weather
{
    public double MinNightTemp { get; set; }
    public double MinDayTemp   { get; set; }
    ... // other properties
}
{{< /highlight >}}

Now add several predictions for future dates, whatever. On a rainy day you may find out the number of fields somehow approached fifty, \*sigh\*, add another one and forget about them.  

And then the Universe decides that it was not fun enough and you should make the predictions more consistent with reality (or at least pretend to do so), for example, by including the information from other sources like <*insert your favorite weather forecast website here, I'd better take an umbrella anyway*>. You quickly run an experiment with the min/max temperatures and expect to get a 10% improvement in accuracy by weighting the observations from different sources... But there're more than 50 properties already, remember?  

{{< highlight csharp >}}
var w1 = 0.8;
var w2 = 1 - w1;
var result = new Weather 
{ 
    MinNightTemp = w1 * orig.MinNightTemp + w2 * ext.MinNightTemp, 
    MaxNightTemp = w1 * orig.MaxNightTemp + w2 * ext.MaxNightTemp,  
    ... // oh, why is that intern on vacation now? who will fill in all this stuff?
}
{{< /highlight >}}

It could be anything else, the key is that there's a set of operations, similar for all the fields, and the fields are somewhat similar too - in our case, they're all of type `double`.  

Let's create an array instead, where the fields are the elements of this array:  

{{< highlight csharp >}}
for(int i = 0; i < n; ++i)
    x[i] = w1 * y[i] + w2 * z[i];
{{< /highlight >}}

But... but now we don't know what is what. Ok, here's the answer:  

{{< highlight csharp >}}
enum P 
{
    MinNightTemp,
    MaxNightTemp,
    ...
}
...   
var res = x[(int)P.MinNightTemp];
{{< /highlight >}}

What about a kind of vector? It looks even better:  

{{< highlight csharp >}}
var x = w1 * y + w2 * z; var res = x[P.MinNightTemp]; // compare to x.MinNightTemp
{{< /highlight >}}

- Saves a lot of typing, decreses the probability of potential mistakes;  
- You don't even need a dictionary - an array is already enough;  
- The enum part can help to preserve the order (in original case that was mandatory requirement), nothing breaks if you add a new feature somewhere in the middle;  
- Every field automatically has its own underlying value, you avoid the mistake of assigning the same value to the different constants (e.g. `MinTempInd = 42` and `MaxTempInd = 42`);  
- There're good chances no one will ever notice, that `Weather` is not a separate class any more. *It looks almost like a property*.  
<p>&nbsp;</p>
<h4 id="theend">Instead of conclusion</h4>  

There're simple yet annoying problems, and to solve them you don't need a dataframe or any complicated data structure or even a separate class. C# is not that bad in the end, no one forces you to write only overly-OOP code.  

That doesn't mean you can't come up with more concise solutions using a different language or solving a different problem. Only that there're ways - in everything - to do the familiar things differently, the ways not everyone notices.  
