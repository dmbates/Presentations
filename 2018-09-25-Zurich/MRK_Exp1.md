---
title: "Re-analysis of data from Masson, Rabe and Kliegl (2017)"
author: "Douglas Bates, Maximillian Rabe & Reinhold Kliegl"
date: "2018-09-28"
output:
  html_document:
    number_sections: true
    toc: true
  pdf_document:
    number_sections: true
    toc: true
---

This is a Julia Markdown (`.jmd`) file that can be executed with the [`Weave` package](https://github.com/mpastell/Weave.jl) for the [`Julia`](https://julialang.org/) programming language.

### Data description and importing the data

The experiment that generated these data is described in Masson, Rabe and Kliegl (2017), *Mem. Cogn.* **45**:480-492 (DOI 10.3758/s13421-016-0666-z).
The data from Experiment 1 were converted to an [`R`](https://www.r-project.org) data frame and saved in the compressed, serialized file, `MRK_Exp1.rds`, using the `saveRDS` function in `R`.

This compressed serialized format can be loaded into Julia using the [`RData`](https:github.com/JuliaData/RData.jl) package.
The [`DataFramesMeta`](https://github.com/JuliaStats/DataFramesMeta.jl) package allows for transformations and selections on a data frame, similar to the `dplyr` package for `R`.
The [`MixedModels`](https://github.com/dmbates/MixedModels.jl) package provides methods of fitting and examining linear and generalized linear mixed-effects models.
It is similar in scope to the [`lme4`](https://github.com/lme4/lme4) package for `R`.

The `InteractiveUtils` and `LinearAlgebra` packages are standard library packages for Julia.
The specific uses for these are to access the `versioninfo` function in `InteractiveUtils` and the `svdvals` function in `LinearAlgebra`.

<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-k'>using</span><span class='hljl-t'> </span><span class='hljl-n'>DataFramesMeta</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-n'>InteractiveUtils</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-n'>LinearAlgebra</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-n'>MixedModels</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-n'>RData</span><span class='hljl-t'>a</span>
</pre>



Load the data from experiment 1 and remove those rows with response time less than 300 ms. or greater than 3000 ms.
Select the columns of interest, which are

| Factor | Description        |
|--------|--------------------|
|   W    | Last trial status  |
|   L    | Last trial quality |
|   Q    | Quality            |
|   F    | Frequency          |
|   P    | Prime              |
|   id   | Subject            |
|   st   | Stimulus           |

The two-level factors in the saved data are coded as ±0.5
For additional stability when forming high-order interaction terms, convert to a ±1 encoding.
(If we form interaction terms numerically from ±1 coded factors the results will also be ±1 coded, no matter how high the interaction order is, and the columns of the model matrix will have the same Euclidean length.)  

<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-n'>d2</span><span class='hljl-t'> </span><span class='hljl-oB'>=</span><span class='hljl-t'> </span><span class='hljl-nd'>@linq</span><span class='hljl-t'> </span><span class='hljl-nf'>load</span><span class='hljl-p'>(</span><span class='hljl-s'>&quot;MRK_Exp1.rds&quot;</span><span class='hljl-p'>)</span><span class='hljl-t'> </span><span class='hljl-oB'>|&gt;</span><span class='hljl-t'>
           </span><span class='hljl-nf'>where</span><span class='hljl-p'>(</span><span class='hljl-ni'>300</span><span class='hljl-t'> </span><span class='hljl-oB'>.≤</span><span class='hljl-t'> </span><span class='hljl-sc'>:rt</span><span class='hljl-t'> </span><span class='hljl-oB'>.≤</span><span class='hljl-t'> </span><span class='hljl-ni'>3000</span><span class='hljl-p'>)</span><span class='hljl-t'> </span><span class='hljl-oB'>|&gt;</span><span class='hljl-t'>
           </span><span class='hljl-nf'>select</span><span class='hljl-p'>(</span><span class='hljl-sc'>:rrt</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-sc'>:P</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-sc'>:F</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-sc'>:Q</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-sc'>:L</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-sc'>:W</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-sc'>:id</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-sc'>:st</span><span class='hljl-p'>)</span><span class='hljl-t'> </span><span class='hljl-oB'>|&gt;</span><span class='hljl-t'>
           </span><span class='hljl-nf'>transform</span><span class='hljl-p'>(</span><span class='hljl-n'>P</span><span class='hljl-t'> </span><span class='hljl-oB'>=</span><span class='hljl-t'> </span><span class='hljl-ni'>2</span><span class='hljl-t'> </span><span class='hljl-oB'>.*</span><span class='hljl-t'> </span><span class='hljl-sc'>:P</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-n'>F</span><span class='hljl-t'> </span><span class='hljl-oB'>=</span><span class='hljl-t'> </span><span class='hljl-ni'>2</span><span class='hljl-t'> </span><span class='hljl-oB'>.*</span><span class='hljl-t'> </span><span class='hljl-sc'>:F</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-n'>Q</span><span class='hljl-t'> </span><span class='hljl-oB'>=</span><span class='hljl-t'> </span><span class='hljl-ni'>2</span><span class='hljl-t'> </span><span class='hljl-oB'>.*</span><span class='hljl-t'> </span><span class='hljl-sc'>:Q</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-n'>L</span><span class='hljl-t'> </span><span class='hljl-oB'>=</span><span class='hljl-t'> </span><span class='hljl-ni'>2</span><span class='hljl-t'> </span><span class='hljl-oB'>.*</span><span class='hljl-t'> </span><span class='hljl-sc'>:L</span><span class='hljl-p'>,</span><span class='hljl-t'> </span><span class='hljl-n'>W</span><span class='hljl-t'> </span><span class='hljl-oB'>=</span><span class='hljl-t'> </span><span class='hljl-ni'>2</span><span class='hljl-t'> </span><span class='hljl-oB'>.*</span><span class='hljl-t'> </span><span class='hljl-sc'>:W</span><span class='hljl-p'>)</span><span class='hljl-t'>
16409×8 DataFrames.DataFrame. Omitted printing of 2 columns
│ Row   │ rrt      │ P       │ F       │ Q       │ L       │ W       │
│       │ Float64  │ Float64 │ Float64 │ Float64 │ Float64 │ Float64 │
├───────┼──────────┼─────────┼─────────┼─────────┼─────────┼─────────┤
│ 1     │ -1.25945 │ 1.0     │ 1.0     │ 1.0     │ 1.0     │ 1.0     │
│ 2     │ -1.97628 │ 1.0     │ -1.0    │ -1.0    │ 1.0     │ -1.0    │
│ 3     │ -2.03666 │ -1.0    │ -1.0    │ -1.0    │ 1.0     │ 1.0     │
│ 4     │ -2.05761 │ -1.0    │ -1.0    │ 1.0     │ 1.0     │ 1.0     │
│ 5     │ -2.07469 │ 1.0     │ -1.0    │ 1.0     │ 1.0     │ -1.0    │
│ 6     │ -1.84502 │ -1.0    │ -1.0    │ -1.0    │ 1.0     │ -1.0    │
│ 7     │ -1.30548 │ -1.0    │ -1.0    │ 1.0     │ 1.0     │ 1.0     │
⋮
│ 16402 │ -1.78571 │ 1.0     │ -1.0    │ -1.0    │ -1.0    │ 1.0     │
│ 16403 │ -1.43885 │ 1.0     │ -1.0    │ 1.0     │ -1.0    │ 1.0     │
│ 16404 │ -1.65289 │ -1.0    │ 1.0     │ -1.0    │ 1.0     │ -1.0    │
│ 16405 │ -1.44092 │ -1.0    │ 1.0     │ 1.0     │ -1.0    │ -1.0    │
│ 16406 │ -2.03666 │ 1.0     │ -1.0    │ 1.0     │ 1.0     │ -1.0    │
│ 16407 │ -1.23457 │ 1.0     │ -1.0    │ 1.0     │ 1.0     │ -1.0    │
│ 16408 │ -1.83824 │ -1.0    │ -1.0    │ 1.0     │ -1.0    │ 1.0     │
│ 16409 │ -1.92308 │ -1.0    │ -1.0    │ 1.0     │ 1.0     │ 1.0     │</span>
</pre>



(The name `linq` echoes that of the MicroSoft Visual Studio [`Language Integrated Query`](https://msdn.microsoft.com/en-us/library/bb397926.aspx) library.
The `@` character indicates this is a call to a macro, not a function in Julia.
See the documentation of the [`DataFramesMeta`](https://github.com/JuliaStats/DataFramesMeta.jl) package for details.)

### Fit a mixed-effects model

A full factorial model in the experimental factors with random-effects for `id` and `st` and potentially correlated random "slopes" for the experimental factors is defined as
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-n'>m1</span><span class='hljl-t'> </span><span class='hljl-oB'>=</span><span class='hljl-t'> </span><span class='hljl-nf'>LinearMixedModel</span><span class='hljl-p'>(</span><span class='hljl-nd'>@formula</span><span class='hljl-p'>(</span><span class='hljl-n'>rrt</span><span class='hljl-t'> </span><span class='hljl-oB'>~</span><span class='hljl-t'> </span><span class='hljl-ni'>1</span><span class='hljl-t'> </span><span class='hljl-oB'>+</span><span class='hljl-t'> </span><span class='hljl-n'>F</span><span class='hljl-oB'>*</span><span class='hljl-n'>P</span><span class='hljl-oB'>*</span><span class='hljl-n'>Q</span><span class='hljl-oB'>*</span><span class='hljl-n'>L</span><span class='hljl-oB'>*</span><span class='hljl-n'>W</span><span class='hljl-t'> </span><span class='hljl-oB'>+</span><span class='hljl-t'> </span><span class='hljl-p'>(</span><span class='hljl-ni'>1</span><span class='hljl-oB'>+</span><span class='hljl-n'>P</span><span class='hljl-oB'>+</span><span class='hljl-n'>Q</span><span class='hljl-oB'>+</span><span class='hljl-n'>L</span><span class='hljl-oB'>+</span><span class='hljl-n'>W</span><span class='hljl-t'> </span><span class='hljl-oB'>|</span><span class='hljl-t'> </span><span class='hljl-n'>st</span><span class='hljl-p'>)</span><span class='hljl-t'> </span><span class='hljl-oB'>+</span><span class='hljl-t'> </span><span class='hljl-p'>(</span><span class='hljl-ni'>1</span><span class='hljl-oB'>+</span><span class='hljl-n'>F</span><span class='hljl-oB'>+</span><span class='hljl-n'>P</span><span class='hljl-oB'>+</span><span class='hljl-n'>Q</span><span class='hljl-oB'>+</span><span class='hljl-n'>L</span><span class='hljl-oB'>+</span><span class='hljl-n'>W</span><span class='hljl-t'> </span><span class='hljl-oB'>|</span><span class='hljl-t'> </span><span class='hljl-n'>id</span><span class='hljl-p'>)),</span><span class='hljl-t'> </span><span class='hljl-n'>d2</span><span class='hljl-p'>)</span><span class='hljl-t'>;</span>
</pre>


and fit as
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-nf'>fit!</span><span class='hljl-p'>(</span><span class='hljl-n'>m1</span><span class='hljl-p'>)</span><span class='hljl-t'>
Linear mixed model fit by maximum likelihood
 Formula: rrt ~ 1 + F + P + Q + L + W + F &amp; P + F &amp; Q + P &amp; Q + F &amp; L + P &amp; L + Q &amp; L + F &amp; W + P &amp; W + Q &amp; W + L &amp; W + &amp;(F, P, Q) + &amp;(F, P, L) + &amp;(F, Q, L) + &amp;(P, Q, L) + &amp;(F, P, W) + &amp;(F, Q, W) + &amp;(P, Q, W) + &amp;(F, L, W) + &amp;(P, L, W) + &amp;(Q, L, W) + &amp;(F, P, Q, L) + &amp;(F, P, Q, W) + &amp;(F, P, L, W) + &amp;(F, Q, L, W) + &amp;(P, Q, L, W) + &amp;(F, P, Q, L, W) + ((1 + P + Q + L + W) | st) + ((1 + F + P + Q + L + W) | id)
     logLik        -2 logLik          AIC             BIC       
 -3.57396102×10³  7.14792204×10³  7.28592204×10³  7.81760743×10³

Variance components:
              Column      Variance     Std.Dev.    Corr.
 st       (Intercept)  0.00320273934 0.056592750
          P            0.00012899963 0.011357800 -0.05
          Q            0.00015932536 0.012622415 -0.36  0.38
          L            0.00003492070 0.005909374 -0.37  0.03  0.03
          W            0.00015751675 0.012550568  0.11 -0.87 -0.01 -0.35
 id       (Intercept)  0.03061291823 0.174965477
          F            0.00002900289 0.005385433 -0.44
          P            0.00012920446 0.011366814 -0.35  0.99
          Q            0.00078842084 0.028078833 -0.41  0.71  0.70
          L            0.00011519270 0.010732786 -0.06  0.16  0.17  0.58
          W            0.00104559433 0.032335651 -0.26 -0.02 -0.05  0.37  0.50
 Residual              0.08571595064 0.292772865
 Number of obs: 16409; levels of grouping factors: 240, 73

  Fixed-effects parameters:
                       Estimate  Std.Error     z value P(&gt;|z|)
(Intercept)            -1.63747  0.0209278    -78.2438  &lt;1e-99
F                     0.0192577  0.0043605     4.41639   &lt;1e-4
P                      0.018828 0.00275244     6.84048  &lt;1e-11
Q                      0.042749 0.00409132     10.4487  &lt;1e-24
L                    0.00161819 0.00265768    0.608872  0.5426
W                    0.00838243 0.00450706     1.85985  0.0629
F &amp; P                0.00720747  0.0024103     2.99028  0.0028
F &amp; Q                0.00139354 0.00243517    0.572254  0.5671
P &amp; Q                -0.0013762  0.0022965   -0.599262  0.5490
F &amp; L                0.00100362 0.00234811    0.427414  0.6691
P &amp; L                0.00238117 0.00231528     1.02846  0.3037
Q &amp; L               -0.00775059 0.00231448    -3.34873  0.0008
F &amp; W              -0.000455143 0.00244687   -0.186011  0.8524
P &amp; W                6.23258e-5 0.00231303   0.0269456  0.9785
Q &amp; W                -0.0017072 0.00231278    -0.73816  0.4604
L &amp; W                0.00532213 0.00231248     2.30148  0.0214
F &amp; P &amp; Q          -0.000302097 0.00229653   -0.131545  0.8953
F &amp; P &amp; L           -0.00132686 0.00231576    -0.57297  0.5667
F &amp; Q &amp; L            0.00263179 0.00231783     1.13545  0.2562
P &amp; Q &amp; L           -0.00401736 0.00231699    -1.73387  0.0829
F &amp; P &amp; W            0.00200139 0.00231414    0.864853  0.3871
F &amp; Q &amp; W            -0.0011834 0.00231131   -0.512002  0.6086
P &amp; Q &amp; W           0.000136195 0.00231456   0.0588428  0.9531
F &amp; L &amp; W            0.00158487 0.00231761     0.68384  0.4941
P &amp; L &amp; W            1.10124e-6  0.0023171 0.000475266  0.9996
Q &amp; L &amp; W            0.00894266 0.00231453      3.8637  0.0001
F &amp; P &amp; Q &amp; L         0.0022095 0.00231484    0.954494  0.3398
F &amp; P &amp; Q &amp; W        0.00137071  0.0023153    0.592022  0.5538
F &amp; P &amp; L &amp; W       -0.00288758  0.0023179    -1.24578  0.2128
F &amp; Q &amp; L &amp; W       -0.00401418 0.00231752     -1.7321  0.0833
P &amp; Q &amp; L &amp; W       -0.00188142 0.00231772   -0.811755  0.4169
F &amp; P &amp; Q &amp; L &amp; W    0.00128903 0.00231502    0.556812  0.5777</span>
</pre>



(By convention, the names of *mutating* functions in Julia - those that modify one or more or their arguments - end in `!` to warn the programmer that arguments can be modified.)

A summary of the optimization process
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-n'>m1</span><span class='hljl-oB'>.</span><span class='hljl-n'>optsum</span><span class='hljl-t'>
Initial parameter vector: [1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0  …  1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0]
Initial objective value:  11981.894272154186

Optimizer (from NLopt):   LN_BOBYQA
Lower bounds:             [0.0, -Inf, -Inf, -Inf, -Inf, 0.0, -Inf, -Inf, -Inf, 0.0  …  0.0, -Inf, -Inf, -Inf, 0.0, -Inf, -Inf, 0.0, -Inf, 0.0]
ftol_rel:                 1.0e-12
ftol_abs:                 1.0e-8
xtol_rel:                 0.0
xtol_abs:                 [1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10  …  1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10, 1.0e-10]
initial_step:             [0.75, 1.0, 1.0, 1.0, 1.0, 0.75, 1.0, 1.0, 1.0, 0.75  …  0.75, 1.0, 1.0, 1.0, 0.75, 1.0, 1.0, 0.75, 1.0, 0.75]
maxfeval:                 -1

Function evaluations:     1224
Final parameter vector:   [0.193299, -0.00201094, -0.0156074, -0.00747159, 0.00489949, 0.0387417, 0.0156643, 0.000135921, -0.0370575, 0.0370108  …  0.0, -0.0594599, -0.0107455, -0.0476173, 0.0294279, 0.0340899, 0.0321956, 0.00542327, 0.083328, 0.02939]
Final objective value:    7147.922043791058
Return code:              FTOL_REACHED</span>
</pre>


shows that the optimization required over 1200 function evaluations on 36 parameters
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-nf'>length</span><span class='hljl-p'>(</span><span class='hljl-n'>m1</span><span class='hljl-oB'>.</span><span class='hljl-n'>optsum</span><span class='hljl-oB'>.</span><span class='hljl-n'>final</span><span class='hljl-p'>)</span><span class='hljl-t'>
36</span>
</pre>



The time required to fit this model is
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-nd'>@time</span><span class='hljl-t'> </span><span class='hljl-nf'>fit!</span><span class='hljl-p'>(</span><span class='hljl-n'>m1</span><span class='hljl-p'>);</span><span class='hljl-t'>
 10.403657 seconds (2.46 M allocations: 131.191 MiB, 0.28% gc time)</span>
</pre>



The estimated covariance matrix factors are
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-n'>m1</span><span class='hljl-oB'>.</span><span class='hljl-n'>λ</span><span class='hljl-p'>[</span><span class='hljl-ni'>1</span><span class='hljl-p'>]</span><span class='hljl-t'>  </span><span class='hljl-cs'># for shown target (st)</span><span class='hljl-t'>
5×5 LowerTriangular{Float64,Array{Float64,2}}:
  0.193299      ⋅             ⋅            ⋅          ⋅ 
 -0.00201094   0.0387417      ⋅            ⋅          ⋅ 
 -0.0156074    0.0156643     0.0370108     ⋅          ⋅ 
 -0.00747159   0.000135921  -0.00255719   0.0185747   ⋅ 
  0.00489949  -0.0370575     0.0173925   -0.0117428  0.0</span>
</pre>


and
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-n'>m1</span><span class='hljl-oB'>.</span><span class='hljl-n'>λ</span><span class='hljl-p'>[</span><span class='hljl-ni'>2</span><span class='hljl-p'>]</span><span class='hljl-t'>  </span><span class='hljl-cs'># for subject (id)</span><span class='hljl-t'>
6×6 LowerTriangular{Float64,Array{Float64,2}}:
  0.597615      ⋅            ⋅          ⋅          ⋅           ⋅     
 -0.00815783   0.0164867     ⋅          ⋅          ⋅           ⋅     
 -0.0134579    0.0364176    0.0         ⋅          ⋅           ⋅     
 -0.0392605    0.0570543   -0.0594599  0.0294279   ⋅           ⋅     
 -0.00204158   0.00572008  -0.0107455  0.0340899  0.00542327   ⋅     
 -0.0283961   -0.0167552   -0.0476173  0.0321956  0.083328    0.02939</span>
</pre>



The singular values of these factors measure the comparative variability in the directions of the principal axes of the covariance structure.
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-n'>svdvals</span><span class='hljl-oB'>.</span><span class='hljl-p'>(</span><span class='hljl-n'>m1</span><span class='hljl-oB'>.</span><span class='hljl-n'>λ</span><span class='hljl-p'>)</span><span class='hljl-t'>
2-element Array{Array{Float64,1},1}:
 [0.194182, 0.0564846, 0.0413341, 0.0191747, 0.0]                  
 [0.599859, 0.115972, 0.0847407, 0.0287916, 0.0209605, 1.72721e-18]</span>
</pre>



By analogy to principal components analysis, a quantity of interest is the cumulative proportion of the variance in the first principal direction, the first two, the first three, etc.
Defining a function to evaluate this
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-k'>function</span><span class='hljl-t'> </span><span class='hljl-nf'>cumulative_variance_proportion</span><span class='hljl-p'>(</span><span class='hljl-n'>a</span><span class='hljl-oB'>::</span><span class='hljl-n'>AbstractMatrix</span><span class='hljl-p'>)</span><span class='hljl-t'>
    cumvar = cumsum(abs2.(svdvals(a)))
    cumvar ./ cumvar[end]
end
cumulative_variance_proportion (generic function with 1 method)</span>
</pre>


and applying it to these covariance factors
<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-n'>cumulative_variance_proportion</span><span class='hljl-oB'>.</span><span class='hljl-p'>(</span><span class='hljl-n'>m1</span><span class='hljl-oB'>.</span><span class='hljl-n'>λ</span><span class='hljl-p'>)</span><span class='hljl-t'>
2-element Array{Array{Float64,1},1}:
 [0.877443, 0.951687, 0.991444, 1.0, 1.0]          
 [0.942633, 0.977866, 0.996677, 0.998849, 1.0, 1.0]</span>
</pre>


shows that for both `st` and `id`, 99% of the variation in these 5- and 6-dimensional random effects is in the first three principal components.

### Version of Julia and characteristics of the computer

<pre class='hljl'>
<span class='hljl-nB'>julia&gt; </span><span class='hljl-nf'>versioninfo</span><span class='hljl-p'>()</span><span class='hljl-t'>
Julia Version 1.0.0
Commit 5d4eaca0c9 (2018-08-08 20:58 UTC)
Platform Info:
  OS: Linux (x86_64-linux-gnu)
  CPU: Intel(R) Core(TM) i5-3570 CPU @ 3.40GHz
  WORD_SIZE: 64
  LIBM: libopenlibm
  LLVM: libLLVM-6.0.0 (ORCJIT, ivybridge)

</span><span class='hljl-nB'>julia&gt; </span><span class='hljl-n'>BLAS</span><span class='hljl-oB'>.</span><span class='hljl-n'>_vendor</span><span class='hljl-t'>
:mkl</span>
</pre>

