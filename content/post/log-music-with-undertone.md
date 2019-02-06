---
title: "Log(music) with Undertone"
date: 2013-12-26
tags: [ fsharp, math, R, music, undertone, matlab ]
categories: [ fsharp ]
---

Whether you're a physicist, biologist, economist, data scientist, whatever,  the chances are you'll meet the Markov chains sooner or later. And today we'll try them on... music!  

![Christmas](/images/christmas-musical.jpg#floatright)

If you're not up to the math part, feel free to [skip it](#music).

<h2 id="math">Math</h2>  

The system states changes are represented with a help of transition matrix, describing the probabilities of possible transitions. This matrix is a key to determine the state of the model after n steps.  

The math behind that is very straightforward - we need just a matrix power. Oh, wait - what if it is fractional, for example, when representing time? There're many ways to compute it, let's take a look at some of them.  

### 1. Eigen vectors  

This method is based on properties of eigen values decomposition:  

$$\begin{array}{1}
\text{V - eigen vectors}\\\\\\
A v_j = \lambda_j v_j\\\\\\
A V = V D \text{, where } D = diag(\lambda_1, ..., \lambda_n)\\\\\\
A^n = V (D ^ n) V^{-1}
\end{array}$$

That's actually how Matlab `mpower` function [looks like](http://www.mathworks.com/help/matlab/ref/mpower.html):  

{{< highlight m >}}
[V,D] = eig(M)
An    = V * (D .^ n) * inv(V) 
{{< /highlight >}}

We can take advantage of Math.NET matrix decompositions to implement this method:  

{{< highlight fsharp >}}
open MathNet.Numerics.LinearAlgebra.Double
open MathNet.Numerics.LinearAlgebra.Generic
open Matrix
open SparseMatrix

let [<Literal>] MAX_ITER = 10000
let [<Literal>] PREC     = 1e-60

/// Matrix power: eigen values decomposition method
/// M^p = V * (D .^ p) / V
let mpowEig (m: SparseMatrix) p =
    let evd  = m.Evd()
    let v, d = evd.EigenVectors(), evd.D()
    mapInPlace (fun x -> x ** p) d
    v * d * v.Inverse()
{{< /highlight >}}

What are the pros/cons?  

**+** the method is rather efficient  
**-** inverse matrix $V$ might not exist [^1]  

$$A = \begin{bmatrix} 0.78 & 0.2 & 0.12\\\\ 0 & 0.78 & 0.32\\\\0 & 0 & 0.78 \end{bmatrix}$$  

$$V = \begin{bmatrix} 1.0000 & -1.0000\\\\ 0.0000 & 0.0000 \end{bmatrix}$$  

**-** some matrices cannot be diagonalised:  

$$A = \begin{bmatrix} 3 & 2\\\\0 & 3 \end{bmatrix}$$  

### 2. Generator matrices  

If we can find a matrix $G$, such as $A = \exp(G)$, then (remember series?)  

$$A^x = \exp(xG) = I + (xG) + (xG)^2 / 2! + (xG)^3 / 3! + ... $$

We are interested in finding a generator $G$ with zero row sums and nonnegative off-diagonal entries. So we'll use Mercator's series:  

$$\log(1 + x) = x - x^2 / 2 + x^3 / 3 - x^4 / 4 + ...$$  

In matrix case it's  

$$G = \log(I + (A - I)) = (A - I) - (A - I)^2 / 2 + (A - I)^3 / 3 - ...$$  

The common concern is that the series methods are not very efficient and may be not accurate enough. It's easy to compare the methods by computing the squared error between the actual and expected values (say, compute $(A^\frac{1}{3})^3$).  
 
Another problem is that a generator matrix (and power too) might have negative elements - not really what you want when dealing with probabilities. We can make them non-negative and keep the rows sums equal to zero... but the power properties won't be preserved. For example, you can find out that $A^7 <> A^{3.5} * A^{3.5}$.

{{< highlight fsharp >}}
/// Generator matrix
let genm (a: SparseMatrix) =
    let size = a.RowCount
    let mid  = constDiag size 1.0
    let a_i  = a - mid

    let rec logm (i, qpow: SparseMatrix) res =
        if i = MAX_ITER then res
        else
           let qdiv = qpow.Divide (float i)
           if sumSq qdiv < PREC then res + qdiv
           else 
               logm (i + 1, -qpow * a_i) (res + qdiv)
    // find log(A - I)
    // update negative off-diagonal entries, the row-sum = 0
    logm (1, a_i) (zeroCreate size size) |> updateneg

/// Exp series approximation
let expSeries (m: SparseMatrix)  =
    let size = m.RowCount
    
    let rec sum (i, k, mpow: SparseMatrix) res =
        if i = MAX_ITER then res
        else
            let mpowi, ki = mpow * m, k * float i
            let mdiv      = mpowi.Divide ki
            if sumSq mdiv < PREC then res + mdiv
            else
                sum (i + 1, ki, mpowi) (res + mdiv) 

    let mid = constDiag size 1.0
    sum (1, 1.0, mid) mid

/// Matrix power: series method
let mpowSeries m (p: float) = SparseMatrix.OfMatrix (genm m * p) |> expSeries
{{< /highlight >}}

**+** Works for non-diagonizable matrices  
**-** Not very efficient  
**-** May be not accurate enough or even fail to converge  

These are just the most simple algorithms - would be interesting to check the others, but let's move on.  

### References  
1. <a href="http://www.cs.cornell.edu/cv/researchpdf/19ways+.pdf">19 Dubious Ways to Compute the Exponential of a Matrix</a>  
2. <a href="http://www.math.ubc.ca/~israel/irw/">Finding Generators for Markov Chains via empirical transition matrices, with applications to credit ratings</a>  

<h2 id="music">Music</h2>

It's not a secret, that Markov models are often used for generative music, here's one of the examples: <a href="http://vikparuchuri.com/blog/making-instrumental-music-from-scratch/">Programming Instrumental Music From Scratch</a> (there're also some interesting references in the comments). We'll do a different thing, though: given notes, we can calculate the transition probabilities and... generate new melodies.  

But everything in its time.  

We need to somehow define the sequences of notes and play them. And thanks to <a href="https://github.com/robertpi/Undertone">Undertone</a> that's pretty easy! First of all, let's download the piano samples - University of Iowa has a collection of recordings <a href="http://theremin.music.uiowa.edu/MISpiano.html">here</a>. You can get them manually or using <a href="https://github.com/robertpi/Undertone/blob/master/src/Undertone/DownloadsSamples.fsx">this script</a>. It's also ok to generate the notes programmatically - but somewhat a bit more realistic is more fun ^_^  

{{< highlight fsharp >}}
// Size: 2/4

/// 1/16 (semiquaver)
[<Literal>] 
let Sq = 0.0625
/// 1/8 (quaver)
[<Literal>] 
let Q1 = 0.125
/// 3/8
[<Literal>] 
let Q3 = 0.375

...

/// Mapping between note enum and file name convention 
let noteToPianoName note =
    match note with
    | Note.C      -> "C" 
    | Note.Csharp -> "Db"
    | Note.D      -> "D"
    | Note.Dsharp -> "Eb"
    | Note.E      -> "E"
    | Note.F      -> "F"
    | Note.Fsharp -> "Gb"
    | Note.G      -> "G"
    | Note.Gsharp -> "Ab"
    | Note.A      -> "A"
    | Note.Asharp -> "Bb"
    | Note.B      -> "B"
    | _ -> failwith "invalid note"

/// Read piano note from .aiff file
let readPianoNote() =
    let cache = Dictionary<_,_> HashIdentity.Structural
    let inline path (noteName, octave) = 
        Path.Combine(dir, "Piano.ff." + noteName + string octave + ".aiff")

    fun (note, octave) ->
        let noteKey = noteToPianoName note, octave
        match cache.TryGetValue noteKey with
        | true, wave -> wave
        | _ -> 
            let wave = IO.read (path noteKey) |> Seq.toArray
            cache.Add(noteKey, wave)
            wave

let makeNote = readPianoNote() 
let getMelody = Seq.collect (fun (noct, time) -> makeNote noct |> Seq.take (noteValue time))
{{< /highlight >}}

The note is defined with its symbol (C = Do, F# = Fis = Fa-diesis = F sharp), octave and time (♪ - 1/8, ♬ - 2 notes * 1/16). The sequence of notes gives a melody, which can be played and even saved later. Can you guess the following one?  

[Original](/sounds/original.mp3)  

Sounds recognizable, but awkwardly stumbling - makes me feel *much* better about my piano skills. I'd recommend to watch [this performace](http://www.youtube.com/watch?v=7cFkae0j_Ns), it's amazing! (and that pianist-feeling suddenly goes down again)). I believe there's a way to make the generated sound smoother and so on, but that's not the goal now.  

{{< highlight fsharp >}}
// Define commonly used notes

/// Sample melody
let ent = [
    D3,Sq; Ds3,Sq
    E3,Sq; C4,Q1; E3,Sq; C4,Q1; E3,Sq; C4,Q3
    C4,Sq; D4,Sq; Ds4,Sq
    E4,Sq; C4,Sq; D4,Sq; E4,Q1; C4,Sq; D4,Q1
    C4,Sq; D3,Sq; Ds3,Sq ...
    C4,Q3
]

Player.Play (getMelody ent)
{{< /highlight >}}

*(Note, the example above has 3 times faster tempo that the one you'll get with downloaded notes)*  

The next thing to do is to calculate the transition probabilities. Consider the following sequence: *E3 - 1/16, C4 - 1/8, E3 - 1/16, C4 - 3/8, C4 - 1/16*.  


The first option is to simply count transitions between notes and then normalize the rows, so the sum are equal to 1:  

$$
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 0 \\\\ C4 & 0 & 0 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 1 \\\\ C4 & 0 & 0 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 1 \\\\ C4 & 1 & 0 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 2 \\\\ C4 & 1 & 0 \end{matrix} \rightarrow$$  
$$
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 2 \\\\ C4 & 1 & 0 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 2 \\\\ C4 & 1 & 1 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 1 \\\\ C4 & 0.5 & 0.5 \end{matrix} $$  

But wait, what about time? There're different notes, we can take that into account too: if the 'smallest' length is 1/16, then we can count, how many such intervals each note takes, multiplying by 16[^2]:  

*E3 - 1/16, C4 - 1/8, E3 - 1/16, C4 - 3/8, C4 - 1/16*  

$$
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 0 \\\\ C4 & 0 & 0 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 1 \\\\ C4 & 0 & 0 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 1 \\\\ C4 & 1 & 1 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 2 \\\\ C4 & 1 & 1 \end{matrix} \rightarrow$$ 
$$
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 2 \\\\ C4 & 1 & 7 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 1 \\\\ C4 & 0.125 & 0.875 \end{matrix}$$  

So, we know the transitions and times when the next note should be played. A uniform random number generator will choose this next note: e.g. if transitions are 0.4 0.2 0.1 0.3 and generated number is 0.65, that's the 3rd one (this operation is called ````choose```` below).  
The note at time `t`, given transition matrix `P` is  

$$n_t = choose(P^t _{n _{t - 1}})$$

{{< highlight fsharp >}}
/// Compute the times and probabilities of transitions between notes
let transitions xs =
    let noteToId = Seq.distinctBy fst xs |> Seq.mapi (fun i (note, _) -> note, i) |> dict

    // # of unique notes and # of notes in melody
    let n, m = noteToId.Count, List.length xs

    let matrix = Array.init n (fun _ -> Array.zeroCreate n)
    let times  = Array.zeroCreate m

    // fill the matrix
    Seq.pairwise xs
    |> Seq.iteri (fun i ((note1, time1), (note2, _)) ->
        let n1, n2 = noteToId.[note1], noteToId.[note2]
        // option #1 - ignore times
        //   matrix.[n1].[n2] <- matrix.[n1].[n2] + 1.
        // option #2 - add extra probability for transition to the same note
        //    matrix.[n1].[n1] <- matrix.[n1].[n1] + time1 * 16.0
        //    matrix.[n1].[n2] <- matrix.[n1].[n2] + 1.0
        // option #3 - take time into account (scale by 16)
        let p = time1 * 16. - 1. in if p > 0. then matrix.[n1].[n1] <- matrix.[n1].[n1] + p
        matrix.[n1].[n2] <- matrix.[n1].[n2] + 1.0
        
        times.[i + 1]    <- times.[i] + time1)
    
    normalizeRows matrix, times, Seq.toArray noteToId.Keys

let matrix, times, idToNote = transitions ent

/// Find index of the next note given probability
let transTo (ps: _[]) = ...

/// Generate music from transition matrices in a file
let genMusic matricesFileName (fstNote, idToNote: _[]) =
    // transition matrices and times
    let ps, ts = readMatrices matricesFileName
    let n      = ps.Count
    let rand   = System.Random()
    
    let rec gen i prevInd res =
        if i >= n-1 then List.rev res
        else
            let j    = transTo ps.[i].[prevInd] (rand.NextDouble())
            let next = idToNote.[j], ts.[i]
            gen (i + 1) j (next :: res)

    gen 0 0 [fstNote] 
{{< /highlight >}}

I tried to compute the transition matrix - and both method failed! Turns out this is the real-life 'unlucky' matrix. To be sure that doesn't work, checked R's <a href="http://cran.r-project.org/web/packages/expm/index.html">expm</a> package:  

{{< highlight R >}}
logm(x, method = "Eigen")
Error in logm(x, method = "Eigen") : non diagonalisable matrix
logm(x, method = "Higham08")
Error in solve.default(X[ii, ii] + X[ij, ij], S[ii, ij] - sumU) :
system is computationally singular: reciprocal condition number = 0
In addition: Warning messages:
1: In sqrt(S[ij, ij]) : NaNs produced
2: In sqrt(S[ij, ij]) : NaNs produced
{{< /highlight >}}

Fortunately, we don't need fraction power here: if we count time in 1/16th, it becomes integer! 1/16 is 1, 3/8 is 6 and so on. Good old multiplication saves the situation.  

Here's several examples of generated melodies - there're only 10 notes are in the game, but the results are already pretty different from the original:  

[Generated-1](/sounds/gen1.mp3)   
[Generated-2](/sounds/gen2.mp3)   
[Generated-3](/sounds/gen3.mp3)   

What if we add an 'extra-probability' for note to stay in the same state?  

*E3 - 1/16, C4 - 1/8, E3 - 1/16, C4 - 3/8, C4 - 1/16*  

$$
\begin{matrix} & E3 & C4 \\\\ E3 & 0 & 0 \\\\ C4 & 0 & 0 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 1 & 1 \\\\ C4 & 0 & 0 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 1 & 1 \\\\ C4 & 1 & 2 \end{matrix} \rightarrow  
\begin{matrix} & E3 & C4 \\\\ E3 & 2 & 2 \\\\ C4 & 1 & 2 \end{matrix} \rightarrow $$
$$  
\begin{matrix} & E3 & C4 \\\\ E3 & 2 & 2 \\\\ C4 & 1 & 9 \end{matrix} \rightarrow
\begin{matrix} & E3 & C4 \\\\ E3 & 0.5 & 0.5 \\\\ C4 & 0.1 & 0.9 \end{matrix} $$  

For this matrix, the generator does exist and we can programmatically compose something like that:  

[Generated-series-1](/sounds/genseries1.mp3)  
[Generated-series-2](/sounds/genseries2.mp3)  

Why stop here? We could define the similar transitions for note length; or add chords - say, generated according to the current tonality; or switch tonalities/modes... Well, there's a whole 'musical math', solfeggio, full of different functions - plenty of space to experiment!  

[Happy Holidays!](/sounds/jingleBells.mp3)

{{< highlight fsharp >}}
let jingleBells = [
    E4,Sq; E4,Sq; E4,Q1
    E4,Sq; E4,Sq; E4,Q1
    E4,Sq; G4,Sq; C4,Sq; D4,Sq
    E4,Q1*2.0
    F4,Sq; F4,Sq; F4,Sq; F4,Sq
    F4,Sq; E4,Sq; E4,Sq; E4,Sq
    E4,Sq; D4,Sq; D4,Sq; E4,Sq
    D4,Q1; G4,Q1
]

Player.Play (getMelody jingleBells)
{{< /highlight >}}


[^1]: the example matrix isn't actually a transition matrix - the row sums should be equal to 1 - it's here just to demonstrate a possible problem.
[^2]: the sequence C4 - 1/8, E3 - 1/16 is almost the same as C4 - 1/16, C4 - 1/16, E3 - 1/16. That gives us 50/50 transitions: after each period of time C4 can either stay as C4 or transform to E3.
