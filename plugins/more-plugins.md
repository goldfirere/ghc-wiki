
There seems to be high demand for tinkering deeper and deeper into GHC passes.



This page summarizes an informal ICFP 2015 discussion about introducing new plugin types to GHC:



Now we have only typeChecker plugins and core-to-core plugins.



The plan is to extend Plugin type to become a sum type, and allow to extend the list of plugin types with:


1. `CoreSyn` plugin that take `CoreSyn` that is given to code generator;
1. STG plugins;
1. `HsSource` plugins;
1. class defaulting plugins.


I would start by replacing {{plugin :: Plugin}} interface by the new {{Plugins}} type that is a sum type:


```
data PluginDecl = CoreSynPlugin         { coreSynProcess :: CoreSyn -> CoreSyn }
                | STGPlugin             { stgProcess :: STGMach     -> STGMach }
                | SourcePlugin          { preprocess :: HsSource    -> IO HsSource }
                | ClassDefaultingPlugin { tryDefault :: WantedConstraints -> TcS WantedConstraints }

type Plugins = [PluginDecl]
```


While we would ideally hope to replace parts of Hooks API, it may be difficult, since it does much more invasive
changes to the compiler, and thus is useful only when GHC is used as a library.


