---
title: "[F#] Dev Life's Little Pleasures"
date: 2013-10-17
tags: [ fsharp, slides ]
categories: [ fsharp, talks ]
---
Different languages have some little things that make the life easier: for devs, their QAs and customers. The app is fast and behaves correctly, the code looks simple and understandable - everyone is happy.  

Here are the <a title="Slides" href="http://www.slideshare.net/luajalla/fsdev-lifes-littlepleasures" target="_blank">slides</a> and a couple of code <a title="Code Samples " href="https://github.com/luajalla/talks/tree/master/fsdevslife" target="_blank">samples</a> from my talk "[F#] Dev Life's Little Pleasures". Briefly, it takes some real-world problems [^1] and provides the simple solutions.  
 
![Captain Kitteh - Mutability](/images/captainkitteh-mutability.png#floatright)

1. **Units of Measure** - need to admit, I was a long time sceptic (c'mon, how can you get kg and m^2 mixed up?)... until was given a spreadsheet with a "correct formula", but its result didn't make sense at all.  

2. **Generic Restrictions** - one of my favourite features! A lot of people don't even know about it. That's unfair. The most complex example I could think of - a static member with out-parameter) is included. But as always - use with care, it might become quite cryptic. 

3. **Immutability as a default** - mostly for the devs with C# or similar background. Great to be sure the things won't change.  

4. **Nulls** and **Options** - the classic.  

5. **Discriminated Unions** and **Pattern-Matching** - invalid states shouldn't have a possibility to appear (and yes, sometimes it's better to forget about the underscores).  

Let the compiler help you!  


[^1]: simplified a bit