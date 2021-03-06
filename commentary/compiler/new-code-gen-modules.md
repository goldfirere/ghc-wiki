## Historical page



This page describes state of the new code generator sometime back in 2008. It is completely outdated and is here only for historical reasons. See [Code Generator](commentary/compiler/code-gen) page for a description of current code generator.


# Overview of modules in the new code generator



This page gives an overview of the new code generator, including discussion of:


- the [new Cmm type](commentary/compiler/new-code-gen-modules#the-new-cmm-data-type)
- the [module structure of the new code generator](commentary/compiler/new-code-gen-modules#module-structure-of-the-new-code-generator)


See also [the description of the new code generation pipeline](commentary/compiler/new-code-gen-pipeline).


## The new Cmm data type



There is a new Cmm data type:


- [compiler/cmm/ZipCfg.hs](/trac/ghc/browser/ghc/compiler/cmm/ZipCfg.hs) contains a generic zipper-based control-flow graph data type.  It is generic in the sense that it's polymorphic in the type of **middle nodes** and **last nodes** of a block.  (Middle nodes don't do control transfers; last nodes only do control transfers.)  There are extensive notes at the start of the module.

  The key types it defines are:

  - Block identifiers: `BlockId`, `BlockEnv`, `BlockSet`
  - Control-flow blocks: `Block`
  - Control-flow graphs: `Graph`

- **`ZipDataFlow`** contains a generic framework for solving dataflow problems over `ZipCfg`. It allows you to define a new optimization simply by defining a lattice of dataflow facts (akin to a specialized logic) and then writing the dataflow-transfer functions found in compiler textbooks. Handing these functions to the dataflow engine produces a new optimization that is not only useful on its own, but that can easily be composed with other optimizations to create an integrated "superoptimization" that is strictly more powerful than any sequence of individual optimizations, no matter how many times they are re-run.  The dataflow engine is based on [
  (Lerner, Grove, and Chambers 2002)](http://citeseer.ist.psu.edu/old/lerner01composing.html); you can find a functional implementation of the dataflow engine presented in [
  (Ramsey and Dias 2005)](http://www.cs.tufts.edu/~nr/pubs/zipcfg-abstract.html).

- **[compiler/cmm/ZipCfgCmmRep.hs](/trac/ghc/browser/ghc/compiler/cmm/ZipCfgCmmRep.hs)** instantiates `ZipCfg` for Cmm, by defining types `Middle` and `Last` and using these to instantiate the polymorphic fields of `ZipCfg`.  It also defines a bunch of smart constructor (`mkJump`, `mkAssign`, `mkCmmIfThenElse` etc) which make it easy to build `CmmGraph`.

- **`CmmExpr`** contains the data types for Cmm expressions, registers, and the like. Here is a fuller description of these types is at [Commentary/Compiler/BackEndTypes](commentary/compiler/back-end-types). It does not depend on the dataflow framework at all.  

## Module structure of the new code generator



The new code generator has a fair number of modules, which can be split into three groups:


- basic datatypes and infrastructure
- analyses and transformations
- linking the pipeline


All the modules mentioned are in the `cmm/` directory, unless otherwise indicated.


### Basic datatypes and infrastructure



Ubiquitous types:


- `CLabel` (`CLabel`): All sorts of goo for making and manipulating labels.

- `BlockId` (`BlockId`, `BlockEnv`, `BlockSet`):
  The type of a basic-block id, along with sets and finite maps.

- `CmmExpr` (`CmmType`, `LocalReg`, `GlobalReg`, `Area`, `CmmExpr`):
  Lots of type definitions: for Cmm types (bit width, GC ptr, float, etc),
  registers, stack areas, and Cmm expressions.

- `Cmm` (`GenCmm`, `CmmInfo`, `CmmInfoTable`):
  More type definitions: the parameterized top-level Cmm type (`GenCmm`),
  along with the type definitions for info tables.


Control-flow graphs:


- `ZipCfg` (`Graph`, `LGraph`, `Block`):
  Describes a zipper-like representation for true basic-block
  control-flow graphs.  A block has a single entry point,
  which is a always a label, followed by zero or mode 'middle
  nodes', each of which represents an uninterruptible
  single-entry, single-exit computation, then finally a 'last
  node', which may have zero or more successors.
  `ZipCFG` is polymorphic in the type of middle and last nodes.
- `ZipCfgCmmRep` (`Middle`, `Last`, `CmmGraph`)
  Types to instantiate `ZipCfg` for C--: middle and last nodes,
  and a bunch of abbreviations of types in `ZipCfg` and `Cmm`.

- `MkZipCfg` (`AGraph`, `mkLabel`, `mkMiddle`, `mkBranch`)
  Smart constructors for control-flow graphs (and the constructors have
  non-monadic types).
  Like `ZipCfg`, `MkZipCfg` is polymorphic in the types of middle and last nodes.
- `MkZipCfgCmm` (`mkNop`, `mkAssign`, `mkStore`, `mkCall`, ...)
  Smart constructors for creating middle and last nodes in
  control-flow graphs (and the constructors have non-monadic types).


Calling conventions:


- `CmmInfo` (`cmmToRawCmm`, `mkBareInfoTable`):
  Converts Cmm code to "raw" Cmm.  What this means is: convert a `CmmInfo` data structure describing the info table for each `CmmProc` to a `[CmmStatic]`. 
  `mkBareInfoTable` is the workhorse that produces the `[CmmStatic]`.  It is also used to produce the info table required for safe foreign calls (a middle node).
- `CmmCallConv` (`ArgumentFormat`, `assignArgumentsPos`):
  Implements Cmm calling conventions: given arguments and a calling convention,
  this module decides where to put the arguments.
  (JD: Crufty. Lots of old code in here, needs cleanup.)


Dataflow analysis:


- `CmmTx` (`Tx`, `TxRes`):
  A simple monad for tracking when a transformation has
  occurred (something has changed).
  Used by the dataflow analysis to keep track of when the graph is rewritten.

- `OptimizationFuel` (`OptimizationFuel`, `FuelMonad`, `maybeRewriteWithFuel`)
  We can use a measure of "fuel" to limit the number of rewrites performed
  by a transformation. This module defines a monad for tracking (and limiting)
  fuel use.
  (JD: Largely untested.)

- `DFMonad` (`DataflowLattice`, `DataflowAnalysis`, `runDFM`):
  Defines the type of a dataflow lattice and an analysis.
  Defines the monad used by the dataflow framework.
  The monad keeps track of dataflow facts, along with fuel,
  and it can provide unique id's.
  All in support of the dataflow module.

- `ZipDataflow` (`ForwardTransfers`, `BackwardTransfers`, `ForwardRewrites`, `BackwardRewrites`,
  `zdfSolveFrom`, `zdfRewriteFrom`, etc)
  This module implements the Lerner/Grove/Chambers dataflow analysis frameword.
  Given the definitions of a lattice and dataflow transfer/rewrite functions,
  this module provides all the work of running the dataflow analysis and transformation.
  A number of the phases of the back end rely on this code,
  and hopefully more optimizations will target it in the future.


And a few basic utilities:


- `CmmZipUtil`: (JD: Unused, I believe, but probably should be used in a few places.)
  A few utility functions for manipulating a zipcfg.
- `PprC`:   Prettyprinting to generate C code.
- `PprCmm`: Prettyprinting the C-- code.
- `PprCmmZ`: (JD: Unused, I believe.)
  Prettyprinting functions related to `ZipCfg` and `ZipCfgCmm`.

### Analyses and transformations


- `CmmLint` (`cmmLint`, `cmmLintTop`):
  Some sanity checking on the old Cmm graphs.
  Not sure how effective this is.
- `CmmLiveZ` (`CmmLive`, `livelattice`, `cmmLivenessZ`):
  Liveness analysis for registers (uses dataflow framework).
- `CmmProcPointZ` (`ProcPointSet`, `callProcPoints`, `minimalProcPointSet`,
  `procPointAnalysis`, `splitAtProcPoints`)
  A proc point is a block in a control-flow graph that must be the
  entry point of a new procedure when we generate C code.
  For example, successors of calls and joinpoints that follow calls
  are procpoints.
  This module provides the analyses to find procpoints, as well as
  the transformation to split the procedure into pieces.
  The procpoint analysis doesn't use the dataflow framework,
  but it really should - dominators are the way forward.
- `CmmSpillReload` (`DualLive`, `dualLiveLattice`, `dualLiveness`,
  `dualLivenessWithInsertion`, `insertLateReloads`,
  `removeDeadAssignmentsAndReloads`):
  Inserts spills and reloads to establish the invariant that
  at a safe call, there are no live variables in registers.
- `CmmCommonBlockElimZ` (`elimCommonBlocks`):
  Find blocks in the CFG that are identical; merge them.
- `CmmContFlowOpt` (`branchChainElimZ`, `removeUnreachableBlocksZ`,
  `runCmmOpts`):
  Branch-chain elimination and elimination of unreachable code.
- `CmmStackLayout` (`SlotEnv`, `liveSlotAnal`, `manifestSP`, `stubSlotsOnDeath`):
  The live-slot analysis discovers which stack slots are live
  at each basic block.
  We use the results for two purposes:
  stack layout (manifestSP) and info tables (in CmmBuildInfoTables).
  The function \`stubSlotsOnDeath' is used as a debugging pass:
  it stubs each stack slot when it dies, hopefully causing bad
  programs to fail faster.
- `CmmBuildInfoTables` (`CAFEnv`, `cafAnal`, `lowerSafeForeignCalls`,
  `setInfoTableSRT`, `setInfoTableStackMap`):
  This module is responsible for building info tables.
  Specifically, it builds the maps of live variables (stack maps)
  and SRTs.
  It also has code to lower safe foreign calls into a sequence
  that makes them safe (but suspending and resuming threads very carefully).
  (JD: The latter function probably shouldn't be here.)

### Linking the pipeline


- `CmmCvt`: Converts between `Cmm` and `ZipCfgCmm` representations.
  (JD: The Zip -\> Cmm path definitely works; haven't tried the
  other in a long time -- there's no reason to use it with
  the new Stg -\> Cmm path).
- `CmmCPSZ`: Links the phases of the back end in sequence, along with
  some possible debugging output.

### Dead code


- `CmmCPSGen`, `CmmCPS` (Michael Adams), `CmmBrokenBlock`, `CmmLive`, `CmmPprCmmZ`, `StackColor`, `StackPlacements`
