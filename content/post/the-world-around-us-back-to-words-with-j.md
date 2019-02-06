---
title: "The World Around Us: back to words with J"
date: 2013-08-25
tags: [ J, discoveries ]
categories: [ languages ]
comments: []
---

{{% admonition quote "Isaac Asimov" %}}
The most exciting phrase to hear in science, the one that heralds new discoveries, is not "Eureka!" (I've found it!), but "That's funny..."  
{{% /admonition %}}

Sometimes everybody wants to get off the beaten track and see the world. See how many exciting discoveries are waiting for you there.  

![J Logo](/images/jwlogo.png#floatright)

I want to bring your attention to a language, which changed the vision of some things for me. This language is J.  

J is an array programming language; it is also functional and even object-oriented. And *extremely concise*. If you ever called F# 'cryptic' because of its operators or started to hate a Scala dev writing ````/:```` instead of ````fold```` – stop right here, because this language is definitely not for you.  

If you have a suspicion that J is for aliens, relax – it’s just *different*. In terms of grammar J is literally closer to English than C or Java. Who said ‘token’? It’s time to remember the good old ‘word’. Functions are verbs, objects – nouns, statements - sentences etc.  

Now let’s experiment!  

- J download is <a title="J Download" href="http://www.jsoftware.com/stable.htm" target="_blank">here</a>;  
- Getting started <a title="Getting Started with J" href="http://www.jsoftware.com/jwiki/Guides/Getting%20Started" target="_blank">materials</a>;  
- Why J <a title="Why J" href="http://www.jsoftware.com/jwiki/Guides/Getting%20Started" target="_blank">page</a>.  

It’s really funny, that *whys* start with:  

> J is a very rich language. You could study and use it for years, and still consider yourself a beginner.  

You’ll get it later =).  

But what trapped me for a weekend? I suspect it's all about unusual code and the new ways of thinking. Maybe something is wrong with me, but J is really fun to write and read! (though the latter requires more effort...)  

### Some operators   

````*:```` - square  
````/```` - ‘insert’, ````f/ x y z = x f y f z````  
````% ```` - division  
````_```` - unary minus  
````>:```` - increment  
````#```` - ‘tally’, the number of elements  
````@```` - ‘atop’, composition  
````@.```` – ‘agenda’/switch conjunction  
````=:```` - assignment  
```` ` ```` - ‘tie’ conjunction  
````|.```` – reverse the order  
````|:```` - transpose matrix  
````+/ .*```` - matrix multiplication  

### Conjunctions  

First of all, note, that the rightmost function is applied first, so `3*2+1 = 3*(2+1) = 9`. How do we calculate, say, the sum of squares?  

Square numbers (output is provided in comments starting with NB.):  

{{< highlight apl >}}
 *: 1 2 3
NB. 1 4 9
{{< /highlight >}}

J has a special operator `/`, called ‘insert’, I prefer to think about it as ‘reduce’:  

{{< highlight apl >}}
+/ 1 2 3
NB. 6
{{< /highlight >}}

The equivalent is `1 + 2 + 3`.  
So the sum of squares can be written as  

{{< highlight apl >}}
+/ *: 1 2 3
NB. 14
{{< /highlight >}}

<p>Conjunction <code>&amp;</code> allows to ‘fix’ the left or right argument:</p>

{{< highlight apl >}}
(^&2) 1 2 3
NB. 1 4 9
(2&^) 1 2 3
NB. 2 4 8
{{< /highlight >}}

### Monads and Dyads  

Those who like the forbidden M-word may be confused. But APL-style definition is actually older: *monad* is a function taking a single argument on the right (like `*:` above). *Dyad* takes two arguments, on the left and on the right (````40 + 2````).  

### Hooks and Forks  

When two verbs come together, we call it a **hook**:  
`(f g) y = y f (g y)` 

Let’s normalize the list of elements so that their sum is equal to 1 (note, that division is denoted with `%`). See the naïve variants and a hook:  

{{< highlight apl >}}
1 2 5 % (1 + 2 + 5)
1 2 5 % +/ 1 2 5
(% +/) 1 2 5
NB. 0.125 0.25 0.625
{{< /highlight >}}

Three verbs form a **fork**:  
`(f g h) y = (f y) g (h y)`  

My favorite fork example is mean:  

{{< highlight apl >}}
(+/ % #) 1 2 3 4
NB. 2.5
{{< /highlight >}}


The same as  

{{< highlight apl >}}
(+/ 1 2 3 4) % (# 1 2 3 4)
+/ 1 2 3 4 % # 1 2 3 4
{{< /highlight >}}


### Gerund  

The things are getting even more interesting: meet a *gerund* (the result of `` ` ``)! The sense is similar to English: derived from a verb, but functions as a noun.  

{{< highlight apl >}}
+ ` -
NB.    ┌─┬─┐
       │+│-│
       └─┴─┘
{{< /highlight >}}

We can choose one of the ‘boxes’ with switch conjunction `@.`:  

{{< highlight apl >}}
+ ` - @. 0
NB. +
+ ` - @. 1
NB. -
{{< /highlight >}}

Look at the abs implementation:  

{{< highlight apl >}}
abs =: + ` - @. (< & 0)
abs 42
NB. 42
abs _42
NB. 42
{{< /highlight >}}

Gerund can be used together with ‘insert’, e.g. the following <del>statement</del> sentence is equivalent to `(50 * 2 % 100)`:  

{{< highlight apl >}}
* ` % / 50 2 100
NB. 1
{{< /highlight >}}


### More Verbs!  

`#.` – dyad ‘base’ or monad ‘from base 2’  

{{< highlight apl >}}
#. 1 0 0 1
NB. 9
3 #. 2 1
NB. 7
{{< /highlight >}}


`[` and `]` – ‘same’, id or argument  

{{< highlight apl >}}
[ 2
NB. 2
1 [ 2
NB. 1
] 2
NB. 2
1 ] 2
NB. 2
{{< /highlight >}}


`+/\`  – running sum (the same works for * etc)  

{{< highlight apl >}}
+/\ 2 3 4
NB. 2 5 9
acc =: /\
+ acc 2 3 4
NB. 2 5 9
* acc 2 3 4
NB. 2 6 24
{{< /highlight >}}

`{.` - take (`n {. L`)  

{{< highlight apl >}}
2 {. 3 4 5
NB. 3 4
{{< /highlight >}}

<p><code>}.</code> - drop (<code>n }. L</code>)</p>
{{< highlight apl >}}
2 }. 3 4 5
NB. 5
{{< /highlight >}}


<p><code>{</code> - fetch</p>

{{< highlight apl >}}
1 { 3 4 5 6
NB. 4
_1 { 3 4 5 6
NB. 6
{{< /highlight >}}


`/:` - sorting  

{{< highlight apl >}}
/: 10 2 5 3
NB. 1 3 2 0
{{< /highlight >}}

Remember fork, id and fetch?  

{{< highlight apl >}}
(/: { ]) 10 2 5 3
NB. 2 3 5 10
{{< /highlight >}}


`q:` - prime factors  


{{< highlight apl >}}
q: 20
NB. 2 2 5
{{< /highlight >}}


`p:` - generate nth prime  


{{< highlight apl >}}
p: 3
NB. 7
{{< /highlight >}}

sum 100 primes:  


{{< highlight apl >}}
+/ p: i. 100
NB. 24133
{{< /highlight >}}


### Modelling Transitions  

Having several states and markov-process like transitions probabilities, what is the probability of one of these states at some time? Credit rating migrations can be taken as an example of such transitions.  

So we define matrix and initial state probabilities:  


{{< highlight apl >}}
T =: 3 3 $ 0.6 0.3 0.1 0.1 0.5 0.4 0 0.1 0.9
x =: 0.3 0.2 0.5
{{< /highlight >}}


At time `y` the result is `T^y` (`x u^:n y` is equivalent to `x u x u …(n times) y`). What is the probability of state 3?  

{{< highlight apl >}}
T3 =: T +/ .*^:2 T
+/ x * (2 {|: T3)
NB. 0.6668
+/ x * [ 2 {|: T3
NB. 0.6668
2 { x +/ .* T3
0.6668
{{< /highlight >}}

The first two options transpose matrix, select the column with index 2 and calculate dot-product with x. The last one uses matrix multiplication verb.  

### Present Value  

Given the stream of cashflows and rate (in %), we want to calculate PV.  
Cashflows: `100 50` 
Rate: `5%`  
`PV = 100 + 50 / (1 + 5%) = 147.61905`  

Splitting by steps:  
- transform %  


{{< highlight apl >}}
pv =: >:@(] % 100"_)
pv 5
NB. 1.05
{{< /highlight >}}


- compute discount factor  

{{< highlight apl >}}
pv =: %@>:@(] % 100"_)
pv 5
NB. 0.952381
1 % 1.05
0.952381
{{< /highlight >}}


- reverse cashflows list and sum the discounted values `(50 * df^1 + 100 * df^0)`:  

{{< highlight apl >}}
|. 100 50
NB. 50 100
0.952381 #. |. 100 50
NB. 147.619
{{< /highlight >}}


- combine everything  

{{< highlight apl >}}
pv =: %@>:@(] % 100"_) #. |.@[
100 50 pv 5
NB. 147.619
{{< /highlight >}}


<pre class="fssnip"><span class="l"> 1: </span><span class="i">pv </span><span class="o">=: %@&gt;:@</span><span class="i">(</span><span class="o">] %<span class="i"> 100"_)</span><span class="o"> #. |.@[</span>
<span class="l"> 2: </span><span class="i">100 50 pv 5</span>
<span class="c">    147.619</span>
</span></pre>  

### Bonus  

If you’re still able to follow this post, here's a couple of references:  
- <a title="J Reference Card" href="http://www.jsoftware.com/jwiki/HenryRich?action=AttachFile&amp;do=view&amp;target=J602_RefCard_color_letter_current.pdf" target="_blank">J Reference Card</a> (2 pages to your rescue);  
- financial <a title="J Financial Phrases" href="http://www.jsoftware.com/jwiki/JPhrases/Finance" target="_blank">phrases;</a>  
- <a title="J Piano Tuning Essay" href="http://www.jsoftware.com/jwiki/Essays/PianoTuning" target="_blank">essay</a> on piano tuning;  
- linear recurrences and <a title="J Matrix Powers" href="http://www.jsoftware.com/jwiki/Essays/Linear%20Recurrences" target="_blank">matrix powers</a>;  
- <a title="Simplex Method" href="http://www.jsoftware.com/jwiki/Stories/RichardBrown." target="_blank">simplex method</a> in J.  

Happy crazy coding!  

<p>&nbsp;</p>
<p>&nbsp;</p>
