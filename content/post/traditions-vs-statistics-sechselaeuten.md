---
title: "Traditions vs Statistics: Sechseläuten"
categories: [ statistics ]
tags: [ Zurich, statistics, fsharp, deedle, R ]
image:
  feature: sechselaeuten/header.jpg
  credit: Böögg (zuerich.com)
  creditlink: https://www.zuerich.com/en/visit/sechselaeuten
date: 2015-04-25
---
What do you expect from this summer? Should you worry about _20:39_ or not? Sunglasses or umbrella? Let's find out! [^1]

### Traditions  

Each country has its own fun traditions and festivals, and Switzerland is not an exception. One of the most exciting events is the spring festival, [Sechseläuten](https://www.zuerich.com/en/visit/history-tradition), which is celebrated in Zürich in April. And it's also a day when you realize, that there're some people in town except the tourists walking along the Bahnhofstrasse on Saturdays! The trees become green, the flags appear here and there, the city becomes lively and colourful. And this year the day was just perfectly sunny. Not the best one for the drivers though because of the [parade of Guilds](https://www.youtube.com/watch?v=VHGKHn-Om-o) (everyone I showed the video especially likes the bear ~1:57 :).  

{{% figure class="center" src="/images/sechselaeuten/sechselaeuten.jpg" title="Böögg (www.zuerich.com)" alt="Böögg" %}}


The culmination of Sechseläuten is the burning of [Böögg](https://www.zuerich.com/en/visit/who-or-what-is-the-boeoegg), a figure of snowman - the winter symbol. There's a belief, that the faster the head explodes, the nicer summer is going to be: say, below 10 min is for sunny and warm weather, above 15 min - for rains and cold... This year it took 20 min 39 s! Doesn't look very nice, huh?  

### Statistics  

So is there any correlation between the Böögg's forecast and actual weather? I couldn't resist checking the facts and downloaded the daily data for the Zürich weather station from the [European Climate Assessment & Dataset](http://eca.knmi.nl/) - it's free and contains a lot of information for the 20th century, including the period we're interested in, I was lucky to find it rather quickly! One can find the yearly Sechseläuten statistics, including the burning times, on the official [Sechseläuten website](http://www.sechselaeuten.ch/sechselaeuten/statistik.asp) (in German).

The criteria for what makes a _nice summer_ are pretty subjective - I picked the mean, min and max temperature, hours of sunshine, cloud cover, humidity and precipitation. Let's assume the _cold summer_ means lower temperatures, or more clouds, or less sunshine, or all of these together.  

Now when we know more or less what we'd like to look at, we need to extract that information from the data - a bunch of simple csv files.  

The weather datasets uses _-9999_ as indication of missing data, and another relevant column contains quality codes. As there's no requirement for each day's data to be present, we'll select only the valid records.

{{< highlight fsharp >}}
type QualityCode =
  | Valid   = 0
  | Suspect = 1
  | Missing = 9
  
// read the dataset from a file, index by DATE and remove unnecessary columns
let readData (fileName: string) = 
    Frame.ReadCsv fileName  
    |> Frame.indexRowsInt "DATE"
    |> Frame.filterRowValues (fun r -> r?Q = float QualityCode.Valid)
    |> Frame.dropCol "SOUID"
    |> Frame.dropCol "Q"

let dataFrames = 
    List.map readData
        [
            "mean_temperature.txt" // TG, in 0.1 C
            "min_temperature.txt"  // TN, in 0.1 C
            "max_temperature.txt"  // TX, in 0.1 C
            "precipitation.txt"    // RR, in 0.1 mm
            "sunshine.txt"         // SS, in 0.1 hrs
            "humidity.txt"         // HU, in 1%
            "clouds.txt"           // CC, in oktas
        ]
{{< /highlight >}}


Cloud cover is in [oktas](http://en.wikipedia.org/wiki/Okta), where 0 means perfectly clear sky and 8 - completely cloudy.  

Now we merge all these datasets, select only the summer months, and transform the temperatures to be in degrees C, sunshine - in hours (per day) and so on:  

{{< highlight fsharp >}}
let weatherData = 
    Frame.mergeAll dataFrames 
    |> Frame.filterRows (fun date _ -> let m = date / 100 % 100 in 6 <= m && m <= 8)
    |> Frame.mapCols (fun col vs -> 
        let series = vs.As() 
        match col with
        | "HU" -> series / 100.0
        | "CC" -> series
        | _    -> series / 10.0)  
{{< /highlight >}}

Finally, we combine the burning times and weather averages for corresponding years:

{{< highlight fsharp >}}
let weatherDataByYear =
    weatherData
    |> Frame.groupRowsByIndex (fun date -> date / 10000)
    |> Frame.applyLevel fst Stats.mean

let fullStats = Frame.join JoinKind.Inner boeoeggStats weatherDataByYear
fullStats.SaveCsv("fullStats.csv", keyNames = ["year"])
{{< /highlight >}}

Now that we have something to look at, let's start with the mean temperatures. We will use R for creating the charts:

{{< highlight fsharp >}}
meanTempPlot = qplot(duration, 
                     TG, 
                     data   = stats, 
                     geom   = c("point", "smooth"), 
                     method = "lm", 
                     color  = TG,
                     main   = "Mean Temperature ~ Duration", 
                     xlab   = "Duration, min", 
                     ylab   = "Temperature, C")

meanTempBoxplot = qplot(duration, 
                        TG, 
                        data = stats, 
                        geom = c("boxplot", "jitter"), 
                        outlier.colour = "red", 
                        fill = I("#3366FFAF"),
                        main = "", 
                        xlab = "Duration, min", 
                        ylab = "Temperature, C")

{{< /highlight >}}

{{% center %}}
![Mean Temperature](/images/sechselaeuten/mean_temperature.png)
{{% /center %}}

It's hard not to notice that there's no dependency between the mean temperature and duration. However, there're some outliers (the red dots): for example, the point in the top-left with the highest temperature is too far from the clustered group. Ironically, that was the most "accurate" prediction, in the year 2003, when it took less than 6 min for the head to explode and the summer was very warm and sunny.  
  
The next two "best" years are in the rows with 20+ min. Even though in 2013 the weather was not that bad despite of 35 min for burning. The data in the table below is sorted by mean temperatures:

{{< highlight fsharp >}}
statsByTG = stats[order(stats$TG),] 
{{< /highlight >}}


| year | time, min	| avg T, C	| min T, C	| max T, C	|precipitation, mm|sunshine, hrs|humidity|cloud cover, oktas|
|------|--------|-------|-------|-------|-------------|--------|--------|-----------|  
| 1956 | 4.00	| 15.07	| 11.12	| 21.13	| 5.70	      |  5.64  | 0.74   | 5.66	    |
| 1978 | 12.00	| 15.38	| 11.42	| 20.27	| 4.52	      |  5.53  | 0.73   | 5.02	    |
| 1980 | 17.00	| 15.63	| 12.10	| 20.24	| 3.91	      |  4.69  | 0.75   | 5.30	    |
| 1972 | 8.00	| 15.77	| 11.86	| 20.70	| 4.84	      |  6.28  | 0.78   | 5.36	    |
| 1965 | 20.00	| 15.84	| 11.77	| 22.21	| 5.17	      |  6.15  | 0.77   | 5.32	    |
| 1966 | 16.00	| 15.84	| 11.62	| 22.12	| 5.24	      |  6.36  | 0.74   | 5.21	    |
| 1968 | 5.00	| 16.03	| 11.72	| 22.48	| 3.91	      |  6.94  | 0.71   | 5.20	    |
| ...  |		| 		|      	|      	|     	      |	       |		|           |
| 2013 | 35.18	| 18.53	| NA   	| NA   	| NA  	      |  8.14  | 0.70   | 4.21	    |
| 1952 | 6.00	| 18.75	| 13.68	| 25.73	| 3.05	      |  8.79  | 0.65   | 4.14	    |
| 1950 | 50.00	| 18.76	| 13.57	| 25.87	| 3.43	      |  8.72  | 0.65   | 4.08	    |
| 1983 | 24.33	| 19.06	| 14.15	| 24.63	| 1.72	      |  6.77  | 0.67   | 4.02	    |
| 1994 | 21.92	| 19.15	| 14.36	| 24.77	| 2.77	      |  7.09  | 0.72   | 4.34	    |
| 2003 | 5.70	| 21.66	| 16.53	| 28.02	| 2.29	      |  9.20  | 0.63   | 3.68	    |

<p>&nbsp;</p>

As we see, the weather tends to be average - very dry, or rainy, or hot summers are exceptions. If you try the random forests (regression) with this data, _% Var explained_ is negative.

{{% center %}}
![Weather](/images/sechselaeuten/weather.png)
{{% /center %}}

One can argue, that for a number of reasons (the quality of weather measures, different structure of the Böögg itself or anything else) the years 1950 and 2014 are not comparable. I tried to run the same experiment with the data for the last 15 years, removing outliers and adding the labels for bad/ok/nice weather to make it a classification problem instead of regression - but the main conclusion is the same. Take a look at the historical averages for each of the summer days (the dark-blue line): the normal range (steel-blue) is quite narrow, even though the lowest and highest points (light-blue) may vary a lot[^2]:

{{% center %}}
![Historical Data](/images/sechselaeuten/historical.png)
{{% /center %}}

### Expectations

The results are pretty reassuring: we can be optimistic and hope that this summer will be great, or at least average - which is also quite nice. The weather is not an easy thing to predict months in advance!

### Reality

Update: The summer 2015 happened to be [extremely](http://allaboutgeneva.com/2015/07/15/swiss-heatwave-2015-new-danger-alert/) [hot](http://www.meteoschweiz.admin.ch/home/aktuell/meteoschweiz-blog.subpage.html/de/data/blogs/2015/8/sehr-warmer-august-extrem-heisser-sommer.html) and beat some temperature [records](https://www.washingtonpost.com/news/capital-weather-gang/wp/2015/09/15/the-summer-of-2015-was-earths-hottest-on-record-nasa-data-show/), not only in Europe. Let's hope we'll have more of a regular summer kind in the future.

And I've recently found out there's also an official study (in German): [Böögg Prognose](http://www.meteoswiss.admin.ch/home/climate/past/climate-of-switzerland/reports-throughout-the-year/boeoegg-prognose.html).

<p>&nbsp;</p>
<p>&nbsp;</p>


[^1]: I realized that I forgot to publish this post on time (the date of event) only several days ago, then I decided to convert the data processing part and use Deedle... Maybe that's because of sunny Milan, but it took some time to figure out which functions to use for relatively simple groupings and aggregations.  

[^2]: To see how you can create similar kinds of plots check [this post](http://rpubs.com/bradleyboehmke/weather_graphic).  
