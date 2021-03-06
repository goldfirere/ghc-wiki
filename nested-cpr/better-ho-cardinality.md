## better-ho-cardinality



It would be nice to merge the code structure improvements and notes into master, to keep my branch short. But it is based on `better-ho-cardinality`, and that is not suitable for merging because of unexpected regressions even in `nofib` and `rtak`. So I am investigating.



In these tests, it is related to reading and showing data. Small example:


```
main = (read "10" :: Int) `seq` return ()
```


Baseline: 49832, `better-ho-cardinality`: 49968. Unfortunately, the changes to, for example, `GHC.Read` are not small, and probably mostly benign...



Trying to minimize and isolate the problem. After some aggressive code deleting, this is where I ended up:


```
{-# LANGUAGE RankNTypes #-}

data P a
  = Get (Char -> P a)
  | Result a

Get f1     `mplus` Get f2     = undefined
Result x   `mplus` _          = Result x

newtype ReadP a = R (forall b . (a -> P b) -> P b)

instance Monad ReadP where
  return x  = R (\k -> k x)
  R m >>= f = R (\k -> m (\a -> let R m' = f a in m' k))

readP_to_S (R f) = f Result `seq` ()

ppp :: ReadP a -> ReadP a -> ReadP a
ppp (R f1) (R f2) = R (\k -> f1 k `mplus` f2 k)

paren :: ReadP () -> ReadP ()
paren p = p >> return ()

parens :: ReadP () -> ReadP ()
parens p = optional
 where
  optional  = ppp p mandatory
  mandatory = paren optional

foo = paren ( return () )

foo2 = ppp (parens ( return () )) (parens (return ()))

main = readP_to_S foo `seq` readP_to_S foo2 `seq` return ()
```


it is important that both `paren` and `parens` are used more than once; if there is only one use-site, the problem disappears (which made it hard to find). Also, most other changes prevent the increase in allocations: Removing the `Monad` instance and turning its methods into regular functions; adding `NOINLINE` annotations to `paren` or `parens`; changing `foo2` to `ppp foo foo`; even removing the dead code that is the first line of the `mplus` function.



Further investigation and aggresive patch-splitting shows that the arity change is causing the regression. Pushed everything but that patch to master, that patch now lives in `wip/exprArity`.


