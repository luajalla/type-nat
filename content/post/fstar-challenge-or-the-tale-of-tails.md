---
title: F* Challenge or The Tale of Tails
date: 2013-06-07
draft: false
tags: [ mono, fsharp, fstar, tail calls ]
categories: [ mono ]

comments:
- id: 102
  author: leppie
  author_email: xacc.ide@gmail.com
  author_url: http://ironscheme.codeplex.com
  date: '2013-06-07 18:59:44 +0200'
  date_gmt: '2013-06-07 18:59:44 +0200'
  content: Mono is still horribly broken in terms in tail call support.
- id: 112
  author: luajalla
  author_email: lu-a-jalla@ya.ru
  author_url: ''
  date: '2013-06-07 19:30:27 +0200'
  date_gmt: '2013-06-07 19:30:27 +0200'
  content: |-
    Well, to be fair it works for the most-used straightforward cases I needed before. But suddenly trolling scala devs with tco is not that fun anymore )
    And it's frustrating that I don't really understand the <em>why</em> part here as it goes somewhere to implementation details. Looking for a workaround seems to be the most realistic solution.
---
F* is a verification-oriented programming language developed at Microsoft Research. If you already know F#, or OCaml, or Haskell, or another language from ML family, you’ll find it familiar.  

### Resources:  

* F* <a title="F* - Security Distributed Programming with Value-Dependent Types" href="http://research.microsoft.com/en-us/projects/fstar/">project home  
* Rise4Fun – <a title="Rise4Fun - F*" href="http://rise4fun.com/FStar" target="_blank">try it online</a>/go through the <a title="Rise4Fun - F* tutorial" href="http://rise4fun.com/FStar/tutorial/guide" target="_blank">guide</a>  
* F* <a title="F* Download" href="http://research.microsoft.com/en-us/downloads/e9089b8e-8871-46a8-987b-75effdcf70e6/default.aspx" target="_blank">download</a>  

To start experimenting with the language on your machine you need .NET 4.0. With Windows that’s it – just start enjoying verifications. But things are getting more interesting if you’re a mono user, when you want to compile the “right” compiler with blackjack and everything.  

F* Download includes the sources and some binaries to bootstrap the compiler.  
There is a dependency on Z3, we need F# PowerPack libs, fslex, fsyacc and i386 version of dylib for Mac and mono version of `Microsoft.Z3.dll` library (either compile a new one or download <a title="Z3Fs" href="https://github.com/dungpa/Z3Fs/" target="_blank">here</a> or <a title="Z3 Starter" href="https://github.com/luajalla/everything-fun/tree/master/z3" target="_blank">here</a>).  

But first of all, need to admit it works only for the small programs now. That’s why I ain’t gonna list the steps now. They are actually quite straightforward – update the paths [^1] and fix the errors if any (e.g. I replaced the reserved word `const` with `cnst`). You can also define `MONO` symbol and add references to `z3mono.dll`.  

The first wall is `Certify.dll` rule – got StackOverflow here. Given the .dll which is in bin folder by default, everything else compiles, including `fstar.exe`.  

At this point only the simplest programs can be successfully verified. Why only them?  

F* relies on tail call optimizations – lots and lots of recursive functions here. Consider the following simple function, it does nothing usefull but illustrates the problem:  
  
{{< highlight fsharp >}}
let rec f x cont =                                                       
    if   x < 0 then 0
    elif x = 0 then cont x
    else f (x - 1) (fun x -> f x cont)

f 250000 id // SO
{{< /highlight >}}

We can verify that the `.tail` instructions are where expected:  

{{< highlight v >}}
.method public static 
  int32 f (int32 x,
     class [FSharp.Core]Microsoft.FSharp.Core.FSharpFunc`2<int32, int32> cont
     ) cil managed   
{   
 .custom instance void [FSharp.Core]Microsoft.FSharp.Core
    .CompilationArgumentCountsAttribute::.ctor(int32[]) = (
        01 00 02 00 00 00 01 00 00 00 01 00 00 00 00 00
    )
    // Method begins at RVA 0x2050
    // Code size 35 (0x23)
    .maxstack 8
    // loop start
     IL_0000: nop
     IL_0001: ldarg.0
     IL_0002: ldc.i4.0
     IL_0003: bge.s IL_0007
  
     IL_0005: ldc.i4.0
     IL_0006: ret
  
     IL_0007: ldarg.0
     IL_0008: brtrue.s IL_0014
  
     IL_000a: ldarg.1
     IL_000b: ldarg.0
     IL_000c: tail.  // cont x
     IL_000e: callvirt instance !1 class 
           [FSharp.Core]Microsoft.FSharp.Core.FSharpFunc`2<int32, int32>::Invoke(!0)
     IL_0013: ret
  
     IL_0014: ldarg.0
     IL_0015: ldc.i4.1
     IL_0016: sub
     IL_0017: ldarg.1
     IL_0018: newobj instance void Tailcalls/f@4::.ctor(
           class [FSharp.Core]Microsoft.FSharp.Core.FSharpFunc`2<int32, int32>) 
     IL_001d: starg.s cont
     IL_001f: starg.s x
     IL_0021: br.s IL_0000
     // end loop
}  // end of method Tailcalls::f
{{< /highlight >}}

{{< highlight v >}}
.method public strict virtual 
instance int32 Invoke (int32 x) cil managed 
{
    // Method begins at RVA 0x2084
    // Code size 16 (0x10)
    .maxstack 8

    IL_0000: nop
    IL_0001: ldarg.1
    IL_0002: ldarg.0
    IL_0003: ldfld class 
          [FSharp.Core]Microsoft.FSharp.Core.FSharpFunc`2<int32, int32> Tailcalls/f@4::cont
    IL_0008: tail.  // f x cont
    IL_000a: call int32 Tailcalls::f(int32, 
          class [FSharp.Core]Microsoft.FSharp.Core.FSharpFunc`2<int32, int32>)
    IL_000f: ret
}  // end of method f@4::Invoke
{{< /highlight >}}

Does it work? Well, for .NET the answer is yes. But on mono the continuation part makes the stack grow. Manipulations with stack size are not the solution, so the sources modification seems to be a way to go.  

There’s an old <a title="Bug 476785 - Tail call support in F#" href="https://bugzilla.novell.com/show_bug.cgi?id=476785" target="_blank">issue</a> with tail calls when called and caller functions have different number of parameters. I thought that can be the case, because of `FSharpFunc` `Invoke`/`InvokeFast` methods, and created a special type for arguments, so all functions had a single parameter. It isn't.  

I still believe it is possible to get F* working on mono [^2]. Hope to find some spare time to dig into the logic and check how it can be rewritten with a simpler recursion, just extremely busy right now ). I also shared some ideas with Nikhil Swamy from F* team – he experimented with mono builds too and definitely knows much better what can be done. That’s an interesting challenge and, of course, the fresh ideas are very welcome!

<p>&nbsp;</p>

[^1]: it’s better to include the full paths – for example I usually happily forget that <i>fsc</i> on my machine means <i>Fast Scala Compiler</i> and waste the time trying to get why it rejects .fs files.
[^2]: everything comes to electricity at the end, right? ;)