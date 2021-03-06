# Shorter import syntax



This page describes a proposed variant of Haskell's `import` declarations.



Trac ticket is [\#10478](http://gitlabghc.nibbler/ghc/ghc/issues/10478).


## Motivation



There are several styles of importing identifiers from other modules in use in Haskell today. To review, unqualified imports of the form,


```wiki
import Data.Map
```


bring into scope every identifier exported by the `Data.Map` module. This means that names such as `Map` and `member` can be used without any additional ceremony. However, this `import` statement also brings into scope identifiers such as `null` that, if used, will conflict with the `null` implicitly imported from `Prelude`.



To avoid such conflicts, one may write an import such that a subset of the names exported by the target module (in this case `Data.Map`) are brought into scope.


```wiki
import Data.Map (Map, member)
```


But now if the programmer does wish to refer to the `null` exported by `Data.Map`, they must write `Data.Map.null`. This is a cumbersome name, so there is a way of providing an alias for the `Data.Map` portion of that fully qualified name.


```wiki
import qualified Data.Map as M
```


Now the programmer may write `M.Map`, `M.member`, and `M.null` to refer to those names exported by the `Data.Map` module. But they may no longer simply write `Map` to refer the name exported by `Data.Map`. If the `Map` name is used often, the noise of the `M.` prefixes can be a nuisance. The annoyance of these prefixes is felt perhaps more acutely when attempting to use operators, particularly those that include a dot in their names!



To distinguish those names the programmer wishes to frequently write from those they are willing to burden with additional syntax, a common idiom is to import a module *twice*.


```wiki
import Data.Map (Map, member)
import qualified Data.Map as M
```


Now, the programmer may write `Map` to refer to `Data.Map.Map`, and `M.null` to refer to `Data.Map.null`.



This pattern applies to common modules such as `Data.Map`, `Data.Set`, `Data.Sequence`, `Data.Text`, `Data.Vector`, etc.


## A Modest Improvement



Without changing the semantics of any existing code, we propose to support the syntax,


```wiki
import Data.Map (Map, member) as M
```


to achieve the exact same result as the last example of the previous section.



The grammar of this new import form is,


```wiki
impdecl -> import modid impspec as modid 
```


In this notation, `import` and `as` are keywords; `modid` is a module name; and `impspec` is a parenthesized list of identifiers that may optionally be prefixed by the word `hiding` (e.g. `import Data.Map hiding (null)`).



This is illegal syntax in Haskell2010.


## The Long View



There is a faction of programmers who favor the exclusive use of either explicit or qualified imports, as in the two-line import of `Data.Map` shown above. This style has the advantage that a reader may easily determine where an identifier comes from without consulting anything other than the source file at hand. Revisiting that example two-line import,


```wiki
import Data.Map (Map)
import qualified Data.Map as M
```


a reader encountering the identifier `Map` may look to the import section of the current file to see it explicitly listed as coming from the `Data.Map` module. Another reader may encounter `M.null` and know that this is unlikely to be the standard `Prelude.null` thanks to the qualification prefix. A consultation with the import section of the current file reveals what the `M.` prefix refers to, and all mysteries are solved.



Other than using two lines and a fair bit of repetition, the aesthetics of this import style have proved divisive within the community, leading to solutions that insert white space between the `import` keyword and the module name if the `qualified` keyword is not present. This is done for the sake of aligning module names in the import section to aid in the reading of that section. More specifically, imports are often sorted by module name, and having the sorted column begin at different textual columns on different lines is arguably a wart. An example of this indentation solution would be,


```wiki
import           Data.Map (Map)
import qualified Data.Map as M
```


The proposed syntax subsumes the vast majority of import scenarios, and may be used in a style that completely omits the `qualified` keyword.


```wiki
import Data.Map (Map) as M
import Data.Set (Set) as S
import Data.Text (Text) as T
```


These sorted imports require no additional work to bring the sorted column into alignment, and cut the number of characters used for imports by half.


## Specification



With the `ShortImports` language extension, all imports are of the following form,


```wiki
import modid maybeImpspec maybeAs
```


We can break down uses (note that the first three are unchanged from Haskell2010; also note that all existing uses of the `qualified` keyword are unchanged),


```wiki
import Data.Map
import Data.Map (Map)
import Data.Map hiding (null)
import Data.Map (Map) as M
import Data.Map hiding (null) as M
```

- The first brings into scope everything exported by `Data.Map`.
- The second brings into scope the identifier `Map`. Everything else exported by `Data.Map` may be referenced by writing its full name (e.g. `Data.Map.null`).
- The third brings into scope everything exported by `Data.Map` *except* the `null` identifier. Thus, `Map` is in scope, but `Data.Map.null` must be written out in its entirety.
- The fourth is new syntax. It brings into scope `Map`. Everything else exported by `Data.Map` may be referenced by writing its full name, or with the alias `M` replacing the `Data.Map` portion of its full name (e.g. `M.null`).
- The fifth is also new syntax. It brings into scope everything exported by `Data.Map` *except* the `null` identifier. Thus, `Map` is in scope, but `null` does not refer to `Data.Map.null`. However, the programmer may write `M.null` to refer to `Data.Map.null`.

## For Students of Haskell



Having many ways to do the same thing can be a point of confusion for those learning a language. In a `ShortImports` world, one may encourage something like the following breakdown of `import` syntax to handle the vast majority of uses:



An `import` statement is of the form,


```wiki
import ModuleName [(unqualifiedImports) [as Alias]]
```


If the statement stops at the module name, everything is brought into scope. If it continues, the next part is a list of names to be brought into scope. After that, an alias for the module may be given in order to use shorter qualified names.


## Potential Confusion



There is an existing import form that looks similar to the proposed syntax,


```wiki
import Data.Map as M
```


The syntactic distinction is in whether or not there is an `impspec` between the first `modid` and the `as` keyword. This is a subtle difference, but fortunately the syntax without the `impspec` is not in common use in publicly available code. An analysis of the Stackage Nightly package set on June 3, 2015 revealed that 0.3% of all `import` statements use this syntax.



With this style of `import`, the identifier `null`, for example is ambiguous. To resolve the ambiguity, the programmer must write `M.null` to refer to `Data.Map.null` or `Prelude.null` to refer to the identifier implicitly imported from `Prelude`. In contrast, when a qualified import is used, the bare `null` identifier continues to refer to `Prelude.null`.


