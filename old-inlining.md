# Warning



This page is very old. It was create at the same time as [
https://ghc.haskell.org/trac/ghc/ticket/3123](https://ghc.haskell.org/trac/ghc/ticket/3123)


# Inlining



Inlining refers to the unfolding of definitions, ie replacing uses of identifiers with the definitions bound to them. Doing this at compile time can expose potential for other optimizations. 


## Unfolding Recursions



Since many definitions in non-trivial programs are recursive, leaving them out alltogether is a serious limitation, especially in view of the encoding of loops via tail recursion. In conventional languages, loop transformations such as loop unrolling are at the heart of optimizing high performance code (for a useful overview, see [
Compiler Transformations for High-Performance Computing, ACM Computing Surveys, 1994](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.41.4885)). As a consequence, many performance-critical Haskell programs contain hand-unrolled recursions, which is error-prone and obscures declarative contents.



There is a tension wrt to the stage in the compilation pipeline that should handle loop unrolling: if we are looking only at removing loop administration overhead and code size, then the applicability of unrolling depends on information that is only available in the backend (such as register and cache sizes); if we are looking at enabling other optimizations, then the applicability of unrolling depends on interactions with code that is located in the Core to Core simplifier (such as rewrite rules for array fusion and recycling). We assume that loop transformations should be considered at both stages, for their respective benefits and drawbacks. This page is concerned with Core-level unfolding of recursive definitions, closing the gap in GHC's inliner.


## An Informal Specification



For the purpose of unfolding/inlining definitions, look at groups of mutually recursive definitions as a whole, rather than trying to think about individual definitions. Compare the existing documentation for [\`INLINE/NOINLINE\`](http://www.haskell.org/ghc/docs/latest/html/users_guide/pragmas.html#inline-noinline-pragma) pragmas.



In the following, let `REC({f g ..})` denote the set of all identifiers belonging to the recursion involving `f`, `g`, .. (`f`, `g`, .. in `REC({f g ..})` or in `{-# INLINE f g .. #-}` are required to belong to the same recursion).



`{-# NOINLINE f #-}`


>
>
> as now: no unfolding of `f`
>
>


`{-# INLINE f #-}`


>
>
> as now: for non-recursive `f` only, unfold definition of `f` at call sites of `f` (might in future be taken as go-ahead for analysis-based recursion unfolding)
>
>


`{-# INLINE f g .. PEEL n #-}`


>
>
> new: unfold definitions of the named identifiers at their call sites **outside** their recursion group `REC({f g ..})`. In other words, **entries into** `REC({f g ..})` via `f`, `g`, .. are unfolded.
>
>


   


>
>
> (for the special case of loops this corresponds to loop peeling)
>
>


`{-# INLINE f g .. UNROLL m #-}`


>
>
> new: unfold definitions of the named identifiers at their call sites **inside** their recursion group `REC({f g ..})`. In other words, **cross-references inside** `REC({f g ..})` via `f`, `g`, .. are unfolded.
>
>


   


>
>
> (for the special case of loops this corresponds to loop unrolling)
>
>


`{-# INLINE f g .. PEEL n UNROLL m #-}`


>
>
> combine the previous two
>
>

>
>
> The numeric parameters are to be interpreted as if each call to `f`, `g`, .. was annotated with both `PEEL` and `UNROLL` limits for the whole recursion group `REC({f g ..})`, starting with the limits from the pragmas (write `f_n_m` for a call to `f` with `PEEL` limit `n` and `UNROLL` limit `m`), to be decreased for every `PEEL` or `UNROLL` action, as follows (`REC({f g})` = {`f` `g` `h`}, in these examples):
>
>

1. 

  ```wiki
     let {-# INLINE f g PEEL n UNROLL m #-}
         f .. = .. f_?_? .. g_?_? .. h_0_0 ..
         g .. = .. f_?_? .. g_?_? .. h_0_0 .. 
         h .. = .. f_?_? .. g_?_? .. h_0_0 .. 
     in ..|f_n_m|..

     --PEEL-->

     let {-# INLINE f g PEEL n UNROLL m #-}
         f .. = .. f_?_? .. g_?_? .. h_0_0 ..
         g .. = .. f_?_? .. g_?_? .. h_0_0 .. 
         h .. = .. f_?_? .. g_?_? .. h_0_0 .. 
     in ..|.. f_(n-1)_0 .. g_(n-1)_0 .. h_0_0 ..|..
  ```

>
>
> Notes: 
>
>

- unfolding produces copies of definition bodies
- the `PEEL` limit at the call site decides the `PEEL` limit for all calls to `REC({f g})` in the inlined copy; this limit decreases with each `PEEL` step
- since peeling unfolds code into call sites from outside the recursion, the `UNROLL` limits of calls to `REC({f g})` are effectively `0` in the inlined copy
- only calls to identifiers named in the `INLINE` pragma can be peeled (`f` and `g` here), calls to other members of the same recursion remain unaffected (`h` here), having effective limits of `0`

1. 

  ```wiki
     let {-# INLINE f g PEEL n UNROLL m #-}
         f .. = .. f_0_m .. g_?_? .. h_0_0 ..
         g .. = .. f_?_? .. g_?_? .. h_0_0 .. 
         h .. = .. f_?_? .. g_?_? .. h_0_0 .. 
     in ..

     --UNROLL-->

     let {-# INLINE f g PEEL n UNROLL m #-}
         f .. = .. .. f_0_(m-1) .. g_0_(m-1) .. h_0_0 .. .. g_?_? .. h_0_0 ..
         g .. = .. f_?_? .. g_?_? .. h_0_0 .. 
         h .. = .. f_?_? .. g_?_? .. h_0_0 .. 
     in ..
  ```

  Notes: 

  - unfolding produces copies of definition bodies
  - the `UNROLL` limit at the call site decides the `UNROLL` limit for all calls to `REC({f g})` in the inlined copy; this limit decreases with each `UNROLL` step
  - peeling conceptually precedes unrolling (`PEEL` limit needs to reach `0` before unrolling commences), to avoid peeling unrolled definitions (this corresponds to an existing restriction of no inlining into definitions to be inlined) 
  - unrolling unfolds copies of the original definitions, not the already unrolled ones, again corresponding to the existing inlining restriction (TODO how to specify this avoidance of unrolling unrolled defs in this form of local rule spec?)
  - only calls to identifiers named in the `INLINE` pragma can be unrolled (`f` and `g` here), calls to other members of the same recursion remain unaffected (`h` here), having effective limits of `0`

>
>
> Peeling and unrolling stop when the respective count annotation has reached `0`. Peeling precedes unrolling, to avoid ambiguities in the size of the peeled definitions. Note that calls into mutual recursion groups is the domain of `PEEL`, while `UNROLL` only applies to calls within mutual recursion groups.
>
>

>
>
> `{-# INLINE f PEEL n #-}`, for `n>0`, corresponds to worker/ wrapper transforms (previously done manually) + inline wrapper, and should therefore also be taken as a hint for the compiler to try the static argument transformation for `f` (the "worker").
>
>

>
>
> Non-supporting implementations should treat these as `INLINE` pragmas (same warning/ignore or automatic unfold behaviour).  This might be easier to accomplish if `INLINE PEEL/UNROLL` were implemented as separate pragmas, even though they are refinements of `INLINE` conceptually.
>
>

>
>
> About the current side-conditions for `INLINE` pragmas:
>
>

- no functions inlined into `f`: 

>
> >
> > >
> > >
> > > still makes sense for `PEEL`, needs to be adapted with an exception for `UNROLL`, in that we want to be able to unroll into the function being unrolled, but we want to use the original body for the unrolling, not an already unrolled one (else unrolling would be exponential rather than linear); this appears to be in line with existing work on `INLINE`
> > >
> > >
> >
>

- no float-in/float-out/cse: 

>
> >
> > >
> > >
> > > similar to existing `INLINE`
> > >
> > >
> >
>

- no worker/wrapper transform in strictness analyser: 

>
> >
> > >
> > >
> > > similar to existing `INLINE`
> > >
> > >
> >
>

- loop breakers: 

>
> >
> > >
> > >
> > > `PEEL/UNROLL` have their own limits, applicable to the whole recursion group, creating intrinsic loop breakers when the counters run out. Every `PEEL` or `UNROLL` action creates calls with smaller counters in the inlined copies, if the calls go into the same recursion.
> > >
> > >
> >
>

## Examples



Examples of hand-unrolled/-peeled loops abound in optimized code. `unroll2.hs` demonstrates the usual worker/wrapper split (+static argument transform), which is part of the standard recommendations for writing code with good performance, and is used all through the standard libraries (here, `PEEL` is more important than `UNROLL`. `unroll0.hs` shows an extreme example with a near trivial loop body, so that the loop administration overhead is relatively expensive (`PEEL` makes the loop body available to the loop combinator worker, `UNROLL` reduces loop administration overhead, and reassociation rules enact constant folding). `unroll1.hs` is an example of loop unrolling activating existing optimization `RULES` (array fusion in this case; note that a [
fixpoint fusion](http://www.haskell.org/pipermail/haskell-cafe/2009-March/057295.html) could be even more beneficial than the finite unrolling fusion here).


## Open Issues


- there is quite a bit of performance to be gained from simple `PEEL/UNROLL`, followed by existing optimizations, but rewrite `RULES` appear insufficiently expressive to handle many of the optimizing transformations involving loops (starting from reassociating the nested expressions arising from unfolding, as indicated in the examples and the `optimization and rewrite rules questions` thread referenced below; but that extends further, eg, one might prefer to express fixpoint fusion of two composed fixpoints (needs `RULES` matching over `case`) without having to write the fixpoints in stream form, or one might want to fuse operations from the end of a loop body with complementary operations from the beginning of the next iteration (TODO could that be hacked around?))

- how to handle programs using implicit recursions (via combinators)? One could `PEEL/UNROLL` the definitions of said combinators, but that would give no control at the call sites (like a single setting for all loops). One possibility is to allow `INLINE PEEL/UNROLL` at the call sites, interpreted as making and unfolding copies of the combinator definitions?

- while, eg, `Hugs` just ignores `INLINE` pragmas, recent `GHC` versions have taken to reporting parse errors instead of warnings when encountering unknown forms of `INLINE`

- how does Core-level recursion unfolding interact with backend-level loop transformations (once those come into existence)?

## Further References



\[0\] GHC mailing list threads, with examples and discussion


- [
  optimization and rewrite rules questions](http://www.haskell.org/pipermail/glasgow-haskell-users/2009-February/016695.html)
- [
  Loop unrolling + fusion ?](http://www.haskell.org/pipermail/glasgow-haskell-users/2009-February/016729.html)
  [
  continued in March](http://www.haskell.org/pipermail/glasgow-haskell-users/2009-March/016732.html)


\[1\] [
An Aggressive Approach to Loop Unrolling](http://citeseer.ist.psu.edu/old/620489.html), 1995



\[2\] [
Compiler Transformations for High-Performance Computing](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.41.4885), ACM Computing Surveys, 1994



\[3\] [
http://en.wikipedia.org/wiki/Loop\_transformation](http://en.wikipedia.org/wiki/Loop_transformation)



\[4\] [
loop unrolling vs hardware](http://www.intel.com/software/products/compilers/flin/docs/main_for/mergedprojects/optaps_for/common/optaps_hlo_unrl.htm)



\[5\] [
Unrolling and simplifying expressions with Template Haskell](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.5.9813), Ian Lynagh, 2003


