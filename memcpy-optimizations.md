# memcpy/memmove/memset optimizations



Starting with version 7.2, GHC has three new primitives for copying/setting blocks of memory, corresponding to the C functions `memcpy`, `memmove`, and `memset`. GHC optimizes occurrences of these primitives into efficient unrolled loops when possible and calls the corresponding C functions otherwise.


## Implementation



The primitives are implemented as three [Cmm language](commentary/compiler/cmm-type) [CallishMachOp\`s](commentary/compiler/cmm-type#), defined in [compiler/cmm/CmmMachOp.hs](/trac/ghc/browser/ghc/compiler/cmm/CmmMachOp.hs). The code generator generates calls to these `CallishMachOp`s using three utility functions: `emitMemcpyCall`, `emitMemmoveCall`, and `emitMemsetCall`, defined in [compiler/codeGen/StgCmmPrim.hs](/trac/ghc/browser/ghc/compiler/codeGen/StgCmmPrim.hs). The helper functions take an extra parameter that indicates the alignment of the arguments, which is used as a optimization hint by the backends.



The reason the primitives are unrolled in the backends, instead of in the code generator, is to allow us to make use of LLVM's `memcpy`/`memmove`/`memset` intrinsics, which LLVM  optimizes well. In the x86/x86-64 backend we unroll the primitives ourselves. The different native code generator backends can also generate more efficient code than a generic case higher up. Currently only the X86 backend unrolls these primitives though, SPARC and !PowerPC both just call the corresponding C functions.


## Unrolling heuristics



We currently only unroll a primitive if


- the address is at least DWORD aligned, and
- we're operating on at most 128 bytes.


The unrolled primitive uses the widest possible move instruction available on the platform and allowed by the address alignment. The 128 byte threshold is the same as the one used by GCC and LLVM when unrolling the C `memcpy`/`memmove`/`memset` functions.


## User API



These primitives aren't directly exposed to the user at this time. Instead the primitives are exposed to the user through a set of primitive operations on boxed arrays:


- `copyArray#`
- `copyMutableArray#`
- `cloneArray#`
- `cloneMutableArray#`
- `freezeArray#`
- `thawArray#`


The latter four allow the user to efficiently clone an array without first setting all elements to some dummy element, which would be required to e.g. implement `cloneArray#` in terms of `newArray#` and `copyArray#`. The implementation of these primitive operations are in [compiler/codeGen/StgCmmPrim.hs](/trac/ghc/browser/ghc/compiler/codeGen/StgCmmPrim.hs).


## Test API



The main test for this feature of GHC is `cgrun069`, located at `testsuite/tests/ghc-regress/codeGen/should_run/`. The user facing API is tested by `cgrun064` and `cgrun068`.


