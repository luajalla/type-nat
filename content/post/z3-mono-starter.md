---
title: Z3 Mono Starter
date: 2013-05-24
draft: false
tags: [ z3, mono, fsharp, quotations ]
categories: [ mono ]
comments:
- id: 12
  author: dungpa
  author_email: phananhdung309@yahoo.com
  author_url: ''
  date: '2013-05-24 05:40:02 +0200'
  date_gmt: '2013-05-24 05:40:02 +0200'
  content: "I have a Github repo to play around with Z3 and F#,. There is a lot of
    examples, you might want to take a look https://github.com/dungpa/Z3Fs\r\n\r\nI
    once asked a question on how to use Z3 on Mono in SO, which is easier to use with
    binary distribution http://stackoverflow.com/questions/14307192/use-z3-managed-api-on-mono"
- id: 22
  author: luajalla
  author_email: lu-a-jalla@ya.ru
  author_url: ''
  date: '2013-05-24 14:07:03 +0200'
  date_gmt: '2013-05-24 14:07:03 +0200'
  content: "Thank you, looks interesting - I like this custom operators approach!
    \r\nDid you try OCaml bindings? For me it wasn't really a choice as the main goal
    was to get F* working on mono, but how is it comparing to dotnet ones?"
- id: 32
  author: dungpa
  author_email: phananhdung309@yahoo.com
  author_url: ''
  date: '2013-05-24 14:56:27 +0200'
  date_gmt: '2013-05-24 14:56:27 +0200'
  content: "I tried the old version of OCaml binding. IMO, it's more verbose and less
    intuitive than the F# version. Z3 team is working on a new OCaml API at http://z3.codeplex.com/SourceControl/list/changesets?branch=ml-ng
    so hopefully it will get better. \r\n\r\nIn the Z3's Windows installer, there
    is utils/Quotations.fs module which compiles a big subset of F# to SMT and verify
    or prove theorems. It's very similar to your approach here. Unfortunately, it
    currently uses the deprecated API (v3.x). I would like to update it to the latest
    API but never get round to do so :-).\r\n\r\nI'm looking forward to your post
    regarding F* and Mono."
- id: 42
  author: luajalla
  author_email: lu-a-jalla@ya.ru
  author_url: ''
  date: '2013-05-24 22:51:46 +0200'
  date_gmt: '2013-05-24 22:51:46 +0200'
  content: |-
    Yeah, it is. And the most annoying is the context (in all versions) - still not sure what is the best way to hide it.
    The point about Windows explains why I didn't find it on Mac ) Wish there were more hours in a day...
- id: 52
  author: 'F# Weekly #21 2013 | Sergey Tihon&#039;s Blog'
  author_email: ''
  author_url: http://sergeytihon.wordpress.com/2013/05/27/f-weekly-21-2013/
  date: '2013-05-26 21:02:10 +0200'
  date_gmt: '2013-05-26 21:02:10 +0200'
  content: '[...] Natallie Baikevich posted &#8220;Z3 Mono Starter&#8220;. [...]'
- id: 552
  author: Yu
  author_email: baiyu0584@gmail.com
  author_url: ''
  date: '2014-07-19 12:23:17 +0200'
  date_gmt: '2014-07-19 12:23:17 +0200'
  content: "Thanks! This really hepls!\r\n\r\nBy the way, I'm using OS X Mavericks
    10.9.4, and in your third step I need to replace the option -fopenmp by -D_NO_OMP_
    to forbid OpenMP, otherwise it won't compile."
- id: 562
  author: luajalla
  author_email: lu-a-jalla@ya.ru
  author_url: ''
  date: '2014-07-22 17:52:39 +0200'
  date_gmt: '2014-07-22 17:52:39 +0200'
  content: Thank you for letting me know, will update the post! (hasn't tried it on
    Mavericks)
---

Have you ever thought how exciting verification tools are? How do you **prove** the **correctness** of a program? Of course, the green tests bring some confidence... but we all know it's not a proof.  
![Pex Logo](/images/pex.png#floatright)

Just look at <a title="Pex" href="http://www.pexforfun.com" target="_blank">Pex</a>, it definitely can't leave anybody cold (recommendation: skip this step for now, it was a trap).  

But this story is not about Pex, but what powers it - Z3, an efficient SMT solver from Microsoft Research. You may ask why you should care (at least I always  do when looking at something new - a tool/language/app/whatever), the answer is simple - because we want the proof, not only a bunch of valid inputs. The work made me a bit paranoid to this point - it takes more time, but the shining correctness allows you not to worry about how much customers can lose just because someone forgot about a not-likely-to-happen workflow, like an unexpected negative number from nowhere -> power -> NaN (surprise, surprise!).  

  
### Resources:    

![Z3 Logo](/images/z3.png#floatright)
* Z3 on <a title="Z3 Home" href="http://z3.codeplex.com/" target="_blank">codeplex</a>, with documentation, papers and slides.  
* Try it online with <a title="Rise4Fun" href="http://rise4fun.com/Z3/tutorial/guide" target="_blank">Rise4Fun</a>.  
* Z3 Constraint Solver on <a title="Nikolaj Bjørner and Leonardo de Moura: The Z3 Constraint Solver" href="http://channel9.msdn.com/blogs/peli/the-z3-constraint-solver" target="_blank">Ch9 </a>.  
* Original home of <a title="Z3: Theorem Prover" href="http://research.microsoft.com/en-us/um/redmond/projects/z3/old/index.html" target="_blank">Z3 project</a>.  
  
Z3 is supported on Windows, OSX, Linux and FreeBSD and has an extensive API:  C, C++, OCaml, .NET languages, Python, Java.  
  
### Z3 + Mono
The standard Z3 distrubution for OSX comes with x64 binaries, but to get it working with Mono we need to compile it ourselves with x86 architecture.  

<strong>1.</strong> Download the sources <a title="Z3 Home" href="http://z3.codeplex.com/" target="_blank">here</a>.  
<strong>2.</strong> Open the folder and generate the makefiles.  

{{< highlight cmd >}}
python scripts/mk_make.py
cd build
{{< /highlight >}}

<strong>3.</strong> We're interested in `config.mk` file. By default the result architecture is x86_x64, that's where the compiler flags help:

{{< highlight make >}}
CXXFLAGS= -D_MP_INTERNAL -m32  -c -fopenmp -mfpmath=sse -O3
 -D _EXTERNAL_RELEASE -fomit-frame-pointer -fPIC -msse -msse2
LINK_FLAGS=-m32
SLINK_FLAGS=-dynamiclib
{{< /highlight >}}

*Update:* according to the Yu's comment you might need to replace the option `-fopenmp` with `-D_NO_OMP_`.  

<strong>4.</strong> Just make it [^1]. The recent Z3 versions require a little change as compiler gives `typename outside of template` error, I just removed `typename` from `fdd.h`.  
  
<strong>5.</strong> Check the architecture

{{< highlight cmd >}}
$ lipo -info libz3.dylib
Non-fat file: libz3.dylib is architecture: i386
{{< /highlight >}}

<strong>6.</strong>  **cd** to `z3/src/api/dotnet` folder and compile Microsoft.Z3.csproj with Mono/.NET 4.0 as a target framework [^2]. Also you'd need to add a config file to map `libz3.dll` to our fresh native `libz3.dylib`.

{{< highlight xml >}}
<?xml version="1.0" encoding="utf-8">
<configuration>
    <dllmap dll="libz3.dll" target="libz3.dylib" os="osx" cpu="x86"/>
</configuration>
{{< /highlight >}}

Now we are ready to try it!

### Examples time  
Let's start with a simple script using official Z3 API, the C# version of examples below can be found in Z3 <a title="Z3 example (C#)" href="http://z3.codeplex.com/SourceControl/latest#examples/dotnet/Program.cs" target="_blank">repository</a>. There's also a nice set of F# examples - I couldn't leave it just in comments - <a title="Z3Fs" href="https://github.com/dungpa/Z3Fs/tree/master/Z3Fs" target="_blank">Z3Fs</a>, a DSL to solve SMT problems by  <a href="https://twitter.com/dungpa">@dungpa</a>.  

For simplicity we assume that Microsoft.Z3.dll, config and libz3.dylib are in the script's folder.  

*Prove that x = y implies g(x) = g(y).*  

{{< highlight fsharp >}}
#r "Microsoft.Z3.dll"  

open System
open Microsoft.Z3
open Microsoft.FSharp.Quotations
open DerivedPatterns
open Patterns  
  
let prove (ctx: Context) f =  
    let s = ctx.MkSolver()  
    s.Assert (ctx.MkNot f)  
    match s.Check() with  
    | Status.UNKNOWN     -> printfn "unknown because %A" s.ReasonUnknown  
    | Status.SATISFIABLE -> printfn "error"  
    | _                  -> printfn "ok, proof: %A" s.Proof  


///Prove that x = y implies g(x) = g(y)
let proveSample() =  
    // create context
    let prms = System.Collections.Generic.Dictionary(dict [ "proof", "true" ])
    let ctx  = new Context(prms)

    // create uninterpreted type
    let U = ctx.MkUninterpretedSort (ctx.MkSymbol "U")
    // declare function g
    let g = ctx.MkFuncDecl("g", U, U)

    // create x and y
    let x = ctx.MkConst("x", U)
    let y = ctx.MkConst("y", U)
    // create g(x), g(y)
    let gx, gy = g.[x], g.[y]

    // assert x = y
    let eq = ctx.MkEq(x, y)

    // prove g(x) = g(y)
    let f = ctx.MkEq(gx, gy)
    printfn "prove: x = y implies g(x) = g(y)"
    prove ctx (ctx.MkImplies(eq, f))

proveSample()
{{< /highlight >}}

Well, it's a pain to write all those `ctx.Mk*`. That's where F# quotations come to hand. Some papers and slides mention Z3 support for quotations, though they seem to be a bit outdated - haven't seen anything similar in the sources. Here is an example of what it can look like.  

*Simplify expression "x + (y - (x + z))"*  

{{< highlight fsharp >}}
/// SimplifiedSpecificCall
let inline (|Func|_|) expr =
    match expr with
    | Lambdas(_, (Call(_, minfo1, _))) -> function
	   | Call(obj, minfo2, args) when minfo1.MetadataToken = minfo2.MetadataToken -> Some args
	   | _ -> None
    | _ -> failwith "invalid template parameter"

/// From quotations to Z3 objects
let z3 expr =
    // create context
    let prms = System.Collections.Generic.Dictionary(dict [ "proof", "true" ])
    let ctx  = new Context(prms)
    let rec unquote expr =
        match expr with
        | Func <@@ (+) @@> [x; y] -> ctx.MkAdd (unquote x, unquote y)
        | Func <@@ (-) @@> [x; y] -> ctx.MkSub (unquote x, unquote y)
        | Lambdas (_, e)          -> unquote e
        | Var var                 -> ctx.MkIntConst var.Name :> ArithExpr
        | x                       -> failwithf "unknown expression %A" x
    unquote expr

/// Simplify expression "x + (y - (x + z))"
let simplifier() =
    let t1 = z3 <@@ fun x y z -> x + (y - (x + z)) @@>  
    let t2 = t1.Simplify()
    printfn "%A -> %A" t1 t2

simplifier()
{{< /highlight >}}

Looks much better, doesn't it?  

*Next time: bringing another f-language to mono...*  
<br/>



[^1]: don't forget to clean up (*.a* and *.obj* files) the folder if you have built x64 lib before.
[^2]: MD-generated makefile & co are available on <a title="Sample on GitHub" href="https://github.com/luajalla/everything-fun/tree/master/z3" target="_blank">github</a>