---
title: "Interop Adventure: From .NET to Matlab and Back"
tags: [ .net, matlab ]
categories: [ interop ]
date: 2015-02-01
---
A lot of exciting things start with the word *interop*, at least the chances to make several surprising discoveries are pretty high. So, we decided to call some Matlab functions from .NET, what could possibly go wrong?  

*This post as a short summary of our discoveries by trial and error method.*

### Why would anyone need that?  

Easy - because something we needed was implemented in Matlab and that was something using a bunch of mathy packages, which we didn't want to spend years rewriting. The calls performance wasn't a big issue either. On the other hand, it worked quite well and produced expected results [^1] and should have been a benchmark for the code which we actually moved to .NET.

### What are the options?

There's not that many options to choose from, so if you don't want to deal with C or COM (and don't mind buying an extra package) you go for the NE Builder. There's also an F# [Matlab Type Provider](http://bayardrock.github.io/Matlab-Type-Provider/), which wasn't an option for us though. The nice thing about the builder is that it generates "type-safe" APIs and the libraries can be deployed on the machines without Matlab installation, the only requirement is MCR (Matlab Compiler Runtime) - which is free.


### Data conversion

Type safe APIs are great. When they are type safe. And they definitely were... to some extent. You start with writing an interface and compiling a library, carefully type-checking Matlab sources in your head [^2] and writing down the output, then run the builder - and hopefully grab the compiled dlls. If something breaks at this stage - it's pure luck, because the error'd be more or less comprehensible.  

Attention now goes to:  
- types (careful with number conversions and out parameters),  
- names (function and its parameters),  
- project definition (the functions should be mapped to the API in the build project).  

The errors probably mean that a function is missing, or its parameter, or it was renamed... However, you still can successfully build the library which will happilly crash at runtime. And here the fun begins.    

There're some docs about [types](http://ch.mathworks.com/help/mps/dotnet/conversion-between-matlab-types-and-net-types.html) and [data](http://ch.mathworks.com/help/mps/dotnet/data-conversion-with-c-and-matlab-types.html) conversion. Unfortunately, I don't have Matlab installed on my machine now, so can't check that, but seems like some things changed for the better in the most recent version (2014b).  

#### Structs and Classes

The first thing to remember is that the names of fields/properties should match *exactly*, including case. If you have a property 'Name' in C# and call 'name' in Matlab, you'll get a runtime error. Just as if you call non-existent property. Obviously, all the checks for existing fields like ````isfield(Person, 'Name')```` become useless and need to be replaced.

It's also a good idea to define the data structures with structs and not classes, otherwise dealing with function is quite painful.

#### Numerical Types

Just be very careful about the numbers. It's easy when you have only doubles, but everything else requires extra attention:  

	int32(5) + [0.1 1 10].^2
    Error using  + 
    Integers can only be combined with integers of the same class, or scalar doubles.  
	
But these are ok:

    5 + [0.1 1.0].^2  
	int32(5) + 0.1.^2
	
Interesting, but all the examples work in [octave](http://octave-online.net/).

#### Arrays

> Note: Multidimensional arrays of above C# types are supported. Jagged arrays are not supported.  

> When a null is passed from C# to MATLAB, it will always be marshaled into [] in MATLAB as a zero by zero (0 x 0) double.  

Depending on a use case multidimensional arrays might be not useful at all, but passing around jagged arrays actually somehow worked, though the dimensions were mixed up: you pass [A x B x C] and get [B x C x A] in Matlab, but nothing ````permute(matrix, [3 1 2])```` couldn't fix. And, of course, the opposite when passing the results back. Also, passing nulls didn't work, so to avoid all the NRE we had to create empty arrays, like 1x1x0, that didn't kill anyone.  

What you don't want to use is a Matlab function returning 3d+ array. At least if there's a chance that the last dimensions are singleton, because Matlab 'ignores' them. Everything is fine when you have 1000x1000x1000 or 2x3x4, but 1x1x1 is the same as 1x1, so .NET wrapper (which expects 3d, i.e. 1x1x1) will fail. 

### When you finally got the data right

There's a bunch of Matlab functions which you can't use in a deployed application, e.g. ````addpath````, you'll have to remove them or check for ````isdeployed```` flag and use only when it's false.  

Setting up Matlab builds on a build server was another special kind of pain, when it turned out that if the installation path (which happened to be Program Files something) contained a space, Matlab couldn't resolve the dependencies... 

But - with some patience, of course - it worked out, happy end! Hope that your way will be more peaceful than this one ;) 


[^1]: as you might guess, there were more results than 'expected', so we ended up fixing the Matlab part too. And the fact that the Optimization toolbox was changed between different Matlab versions was quite exciting too.
[^2]: pleease, please always add a comment describing function parameters in the code, at least the matrix dimensions! Unless you hate all the people.