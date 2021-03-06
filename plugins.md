# GHC Plugins


## Intent



We would like to support dynamically-linked Core-to-Core plug-ins, so that people can add passes simply by writing a Core-to-Core function, and dynamically linking it to GHC. This would need to be supported by an extensible mechanism like attributes in mainstream OO languages, so that programmers can add declarative information to the source program that guides the transformation pass. Likewise the pass might want to construct information that is accessible later. This mechanism could obviously be used for optimisations, but also for program verifiers, and perhaps also for domain-specific code generation (the pass generates a GPU file, say, replacing the Core code with a foreign call to the GPU program). 


## Future Work



Annotation related: see [Plugins/Annotations](plugins/annotations).


## (OUTDATED) Implementation Speculation



**Here be dragons! ** This is just the result of my very preliminary thinking about how we might implement this in GHC. I am by no means an expert and so this section is certainly neither authoritative or correct! - Max Bolingbroke, March 08



Plugins as implemented are discussed on [NewPlugins](new-plugins).


### General Design Goals



It would be desirable to ensure that Haskell programs that contain information for a particular plugin still compile when that plugin is not available, in the sprit of other pragma-carried information.



We would like the plugin install process to be totally painless, ideally by reusing the existing Cabal / package.info mechanisms.


### Detailed Discussion Of Implementation Issues


- [Plugins/Phases](plugins/phases)
- [Plugins/Annotations](plugins/annotations)

### User Interface



The user would write code something like the following:


```wiki

module Main where

{-# PLUGIN NVidia.GHC.GPU NVidiaGPUSettings { useTextureMemoryMb = 256 } #-}

{-# PLUGIN NVidia.GHC.GPU doSomethingExpensive NVidiaGPUFunctionSettings { maxStackDepth = 1024 } #-}
doSomethingExpensive :: Int -> ImpressiveResult
doSomethingExpensive = ....

main = print (map doSomethingExpensive [1..10])

```


That is, they can include two types of pragmas. Ambient configuration ones analagous to GHC\_OPTIONS:


```wiki
{-# PLUGIN plugin_name some_data_structure #-}
```


And binder-specific ones analagous to INLINE:


```wiki
{-# PLUGIN plugin_name binder some_data_structure #-}
```


An alternative would be to allow some of the GHC command line arguments to filter through to the plugin:


```wiki
ghc --make Main.hs ... -pgpu-texture-memory-mb=256 ...
```

### Plugin Writer Interface



The plugin writer would provide:


- Loadable library that contains their plugin code: this would be placed in a magic GHC subdirectory, referenced somehow in package.conf or something. It would probably be best if we had plugins distributed as source and cabal-built / installed via the usual package mechanism so that they can deal with many architectures / GHC versions.
- *Optionally*, a component of their plugin library which contains the definitions for the data types they are going to want to use in their pragmas, such as NVidiaGPUSettings.


We should provide at least the following hooks to the plugin library:


- Initial configuration: this is when it gets fed the modules ambient PLUGIN pragma information
- Ability to install extra core passes (their library would be fed the core syntax tree either as [ExternalCore](external-core) or as GHCs core data type (via the GHC-as-a-library stuff))
- Ability to add notify GHC about extra things to link in (e.g. GPU code compilation artifacts). This does not necessarily play well with separate compilation: would have to do something like add info to .hi so the eventual linking stage can find out about any extra artifacts that might be required. This actually causes problems right now when you e.g. compile a module that uses FFI separately: GHC will need to be fed an appropriate linker command line along with the .o file names rather than inferring it from the .hi files
- Any more you can think of....


SPJ believes that external core would not protect against versioning in GHCs core representation as it will change with core itself and so we should use core directly for simplicity. This also allows reuse of GHCs substantial internal utility library for manipulating core representations by the plugin authors.



Concretely, the plugin writer would implement an interface that looked something like this:


```wiki

-- The pragma supplied options must be of type dynamic because they are instantiated by GHC. The plugin author
-- will want to check that the data the user supplied is something they are prepared to accept as a pragma
-- Command line options will be supplied as a simple string list of those things supplied to GHC with a -p prefix
-- Note that this design means that the type of command line plugin option are limited to:
--  * -pfoo
--  * -pfoo-arg (NOT -pfoo arg)
--  * -pfoo=n
--  * -pfoo;
-- Furthermore we cannot warn about unrecognized -p args unless we have plugins report back unrecognized args. 
-- Reporting the error at that late stage will also be problematic, so we probably shouldn't bother.
-- We should provide a library that is able to parse these for plugin writers so they don't reinvent the 
-- wheel (maybe just using GHCs CmdLineParser)
-- An alternative design for functionPragmaOptions would be to attach the pragma data structures to Notes 
-- in the Core syntax tree. However, SPJ feels that would interact too much with other compiler stages, that 
-- have a habit of moving Notes around.
newtype PluginOptions = 
  PluginOptions { ambientPragmaOptions :: [Dynamic]
                , functionPragmaOptions :: Map Id [Dynamic]
                , commandLineOptions :: [String] }
initialize :: PluginOptions -> IO GHCPluginInstance
             
data GHCPluginInstance = GHCPluginInstance {
  -- We could either have them change the pipeline explicitly, 
  -- by providing a CoreDoPluginPass constructor in CoreToDo:
  installCorePasses :: [CoreToDo] -> [CoreToDo]
  -- Or (possibly less fragile and operational, but more complicated) just something like:
  providedCorePasses :: [(CorePassPlacementSpec, PluginCorePassName, PluginCorePass)]
  -- For details see note [Declarative Core Pass Placement]

  -- This very simple approach will probably be sufficient
  getAdditionalLinkArtifacts :: IO [FilePath]
}


type PluginCorePass = DynFlags -> UniqSupply -> [CoreBind] -> IO [CoreBind]

-- Note: [Declarative Core Pass Placement]
-- If we went the declarative route for core pass placement we would have something like this:
type CorePassPlacementSpec = [CorePassPlacement]
data CorePassPlacement
  = CertainlyBefore CorePassName
  | CertainlyAfter CorePassName
  | IdeallyBefore CorePassName
  | IdeallyAfter CorePassName
  | InPhase Int -- Phase 0, 1 or 2 as they currently exist in GHC
  | WhenOptimizationLevelAtLeast Int -- Level 0, 1 or 2 as they currently exist in GHC

type CorePassName = String
type PluginCorePassName = CorePassName
-- Alternatively we could expose a CoreToDo style of thing as the core
-- pass names, e.g:
type PluginCorePassName = String
data CorePassName 
  = SimplifyCorePass
  | FloatInwardsCorePass
  | CSECorePass
  | ...
  | PluginCorePass PluginCorePassName
  deriving (Eq)             
-- This would let us recover some compile time safety while still allowing plugins to refer
-- to passes installed by other plugins, which may be desirable.
-- End Note: [Declarative Core Pass Placement]

```


Alternative design that would let the plugin authors reuse GHCs parsing infrastructure and also let us warn about unrecognized command line flags in command line parsing as usual:


```wiki
-- The actions encapsulated within the OptKind will be run by
-- the GHC command line parser: you can use the action to accumulate flags that will
-- be output by the runCommandLineM thing and eventually fed back to the initialize function
runCommandLineM :: CommandLineM a -> (a, CommandLineOptions)
dynamicFlags :: [(String, OptKind CommandLineM)]

-- These will be defined by the particular plugin author: clearly this makes it hard for GHC to 
-- type check the plugin, but we'll ignore the type safety of dynamic linking here!
newtype CommandLineM = ...
newtype CommandLineOptions = ...
```

### GHC Stuff



The compiler needs to bring the plugin writer and users contributions together. A possible game plan might be:


1. Save any command line options beginning with -p so we can give them to plugins later.
1. Suck in Haskell source as normal, but also parsing PLUGIN pragmas
1. Load up any plugins we see referenced by PLUGIN pragmas, discarding those we don't know about (on the basis that PRAGMAS are meant to be extra information that isn't essential to compile the program, though we might want to at least warn if we can't find one!). This could happen via [
  hs-plugins](http://www.cse.unsw.edu.au/~dons/hs-plugins/).
1. Give the plugins a chance to add some "implicit" imports so that the renamer can see the pragma data types when going through the pragmas. Again this is motivated by the idea that programs that use plugins should still compile if they are absent.
1. Rename, type check as normal, though TcBinds.mkPragFun will need to be extended to splat binder-specific PLUGIN pragmas on those binders
1. Desugarer will have to do something to make the pragma data structure syntax trees into an actual in-memory value. Essentially, we want to evaluate the expression in a context where it just has access to the constructors of the plugins imported data type library. We could reuse some of the GHCi apparatus to do this, see e.g. HscMain.compileExpr, HscMain.hscStmt and Linker.linkExpr. We then wrap this value up into an existential type that has a Dynamic instance and put it into the pragma on the Core (so the typechecker will have to make sure this part of the pragma has such an instance).
1. Give the plugin a chance to install core passes into the pipeline. For each one:

  1. Feed the core to the dynamically linked pass function.
  1. The plugin will encounter CoreSyn Notes that include information about various binders OR will get a map of pragma information on a per-function basis. It can use Dynamic to try and cast the value this contains to an appropriate type from the accompanying pragma data type library
  1. Give the returned core to the next core2core stage.
1. Codegen etc as normal
1. Add to the output file links to anything the plugin specified. Somehow record the extra library files to eventually link in, e.g. via a .hi file


As you can see there are a lot of design choices we might make. Feel free to add your own thoughts on this feature to the page.


