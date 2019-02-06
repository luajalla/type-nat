---
title: LP - Simplex Draft
date: 2013-06-22
tags: [ fsharp, lp, math ]
categories: [ math ]
---
Everybody solves some optimization problems – the airlines schedule flights, companies manage production facilities, salesman still looks for traveling options… When you need to achieve the best outcome minimizing/maximizing a linear cost functions you meet linear programming.  

LP was developed for military purposes in 1939, so it has a long history. Now it is one of the most important problems in operations research and heavily used in different areas directly or as sub-problems.  

### Intro
The standard algorithm here is the simplex method. Lots of information is floating around, but let’s review the basics. There’re several forms for expressing linear programs. Standard form:  

$$\begin{array}{1}
\text{maximize }{c}^\prime x \\\\\\
\text{subject to}\\\\\\
Ax <= b\\\\\
x >= 0
\end{array}$$

We assume that the rows of matrix $A$(*m x n*) are linearly independent.  

The form above can be easily converted to slack form with a help of the new variables (slack variables):  

$$\begin{array}{1}
s = b_i - \sum a\_{ij} x_j \\\\\\
s >= 0
\end{array}$$

So the problem now is  

$$\begin{array}{1}
\text{maximize }{c}^\prime x\\\\\
\text{subject to}\\\\\
Ax = b\\\\\
x >= 0
\end{array}$$

![Simplex](/images/simplex.png#floatright)

Geometrical interpretation for 2 variables is very intuitive: all $x_1$ and $x_2$ satisfying the constraints are feasible solutions, and an optimal solution – maximum or minimum – is at a vertex. This feasible region, for n variables – in n-dimensional space, is called a simplex. The algorithm terminates when it reaches an optimal objective value at some vertex.  

Why write a custom implementation? For me Excel solver is usually a solution [^1], works like a charm – when it’s not a Mac version. The naïve version of algorithm is quite simple to implement, it’s also a nice refresher and managed to solve the problem, so why not?  

Consider the following problem:  

$$\begin{array}{1}
\text{minimize }x_1 + 5x_2 - 2x_3 \\\\\\
\text{subject to} \\\\\\
x_1 + x_2 + x_3 <= 4 \\\\\\
x_1 <= 2 \\\\\\
x_3 <= 3 \\\\\\
3x_2 + x_3 <= 6 \\\\\\
x_1, x_2, x_3 >= 0
\end{array}$$

The solution is very straightforward:  
<strong>1.</strong> Convert to slack form  

$$\begin{array}{l}
\text{minimize } x_1 + 5x_2 - 2x_3\\\\\\
\text{subject to }\\\\\\
x_1 + x_2 + x_3 + x_4 <= 4\\\\\\
x_1 + x_5 <= 2\\\\\\
x_3 + x_6 <= 3\\\\\\
3x_2 + x_3 + x_7 <= 6\\\\\\
x_1,\ x_2,\ x_3,\ x_4,\ x_5,\ x_6,\ x_7 <= 0
\end{array}$$

<strong>2.</strong> We see that the cost can be reduced with increasing $x_3$ (it’s also obvious that max $x_3$ value is equal to 3). 
If the cost is already optimal the solution is found.  

For each $j$ the reduced cost is defined as  

$$\begin{array}{1}
\bar{c_j} = c_j - {c_b}^\prime B^{-1}A_j\\\\\\
\text{where }c_b\text{ – the vector of basic variables costs.}
\end{array}$$

<strong>3. </strong> Look for a direction of cost decrease, when moving into the new direction we replace a basic variable with a new one; go to the next iteration.  

### Implementation
The first version I came up with was entirely immutable – with all the costs of creating new sets, matrices etc. But there’s no sense to recreate the whole matrix when only one column is changed for the next iteration. This version is listed below – not that functional, more dangerous, with a mutable state, but cares about time/memory. Note, that this implementation doesn’t handle some corner-cases as it was not a part of my goal.  

{{< highlight fsharp >}}
#r "MathNet.Numerics.dll"
#r "MathNet.Numerics.FSharp.dll"

open MathNet.Numerics.LinearAlgebra.Double
open MathNet.Numerics.LinearAlgebra.Generic

type SimplexResult =
    | Success of Vector<float>
    | Error   of string

/// Simplex method implementation
let simplexImpl (A: _ Matrix) b (c: _ Vector) (x: _ Vector, Ib: _[], In: _[]) =
    let cb   = Seq.map c.At Ib |> DenseVector.ofSeq
    let m, n = A.RowCount, A.ColumnCount

    // 1. start with basic matrix
    let B = Seq.map A.Column Ib |> DenseMatrix.ofColumns m m
    
    let rec calc (Binv: _ Matrix) iter =
        // 2. reduce costs and check optimality conditions
        let p = cb * Binv * A
        c.MapIndexedInplace (fun i ci -> ci - p.[i])

        match Seq.tryFindIndex (fun i -> c.[i] < 0.) In with
        | Some jind ->
            let j = In.[jind]
            // 3. unboundness check
            let u = Binv * A.Column j
            if Seq.forall (fun ui -> ui <= 0.) u then Error "cost unbounded"
            else
                // 4. improvement
                let l, theta =
                    Seq.mapi (fun i ui -> i, x.[i] / ui) u
                    |> Seq.filter (fun (_, di) -> di > 0.)
                    |> Seq.minBy snd
               
                // 5. update solution
                x.MapIndexedInplace (fun i xi -> if i = l then theta else xi - theta * u.[i])

                // 6. update basis, indices and cost
                B.SetColumn(l, A.Column j)
                In.[jind] <- Ib.[l]
                Ib.[l]    <- j
                cb.[l]    <- c.[j]

                let Binv = inverse B
                calc Binv (iter + 1)
        | _ -> 
            // fill solution vector x0, x1, ..., xn
            let res = DenseVector.zeroCreate n
            Seq.iteri (fun i ib -> res.[ib] <- x.[i]) Ib
            Success res

    calc (B.Inverse()) 1
{{< /highlight >}}

For simplicity we use the naïve initialization – all given variables become nonbasic, the slack ones – basic and their values are equal to constraints vector $$b$$. The full method returns the solution vector and cost function value:  

{{< highlight fsharp >}}
/// Naive initialization function - simply set basic x to b
let initSimplex (A0: _ Matrix) (b0: _ Vector) (c0: _ Vector) = (…)

/// Revised Simplex implementation: min cx, Ax <= b, x >= 0, b >= 0
let simplex A0 b0 (c0: _ Vector) =
    let A, b, c, x, Ib, In = initSimplex A0 b0 c0
    match simplexImpl A b c (x, Ib, In) with
    | Success xs ->
        let x0 = xs.[ .. c0.Count-1]
        let cx = c0 * x0
        Some (x0, cx)
    | _ -> None
{{< /highlight >}}

And now time for my favorite part of the method and one the best ways to improve performance – reuse the data you already have to avoid recomputations whenever it’s possible.
In this case it’s about inverting the basis matrix. We know that $B$ at the next iteration is the same as current, except one column.
There’s also the current $B^{-1}$. The naïve simplex + math = revised simplex method.  

{{< highlight fsharp >}}
/// invert matrix given inverse of one column different matrix
let inv (a: _ Matrix) (aprev: _ Matrix, u: float Vector, l) iter =
    // recompute from scratch, because errors accumulate
    if iter % 20 = 0 then a.Inverse() 
    else
        let ul   = u.[l]
        let lrow = aprev.Row l
        Matrix.mapRows (fun i row -> 
            if i = l then row / ul else row - lrow * u.[i] / ul) aprev
{{< /highlight >}}

### Sample

{{< highlight fsharp >}}
// standard problem form   
let A = matrix [[1.; 1.; 1.]
                [1.; 0.; 0.]
                [0.; 0.; 1.]
                [0.; 3.; 1.]]

let c = vector [1.; 5.; -2.]
let b = vector [4.; 2.; 3.; 6.]
    
simplex A b c |> printfn "%A" //[0.0;0.0;3.0],-6.0
{{< /highlight >}}

Complete snippet version is available at <a title="Simplex method" href="https://github.com/luajalla/snippets/blob/master/Simplex.fsx" target="_blank">github.</a>  



### Summing up  

- mutability may be evil, but sometimes it’s a way to go;  
- reusing the computation results and the parts of datastructures ftw.  

### What to look at
- <a title="MS Solver Foundation" href="http://msdn.microsoft.com/en-us/devlabs/hh145003" target="_blank">MS Solver Foundation</a> – quite a nice tool to check too (with msi downloads, but I tried it a long time ago with mono – and it worked).  
- some theory: Introduction to Linear Optimization book (Dimitris Bertsimas and John N. Tsitsiklis) or anything else.  
<p>&nbsp;</p>
<p>&nbsp;</p>

[^1]: one of the side-effects of my work are numerous spreadsheets, and Solver proved to be incredibly helpful.