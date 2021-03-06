# GND Pre-GHC-7.8



We will ignore the type-safety issues as they have been resolved, instead we'll just look at the module boundary / abstraction issues.



In GHC 7.6 or earlier, assume a library author writes the following `MinList` data type:


```wiki
module MinList (
        MinList, newMinList, insertMinList,
    ) where

data MinList a = MinList a [a] deriving (Show)

newMinList :: Ord a => a -> MinList a
newMinList n = MinList n []

insertMinList :: Ord a => MinList a -> a -> MinList a
insertMinList s@(MinList m xs) n | n > m     = MinList m (n:xs)
                                 | otherwise = s
```


The `MinList` data type has an invariant (that depends on the `Ord` typeclass for the type parameter `a` that `MinList` is instantiated at) that after initialization it doesn't accept any element less than the initial element.



In GHC 7.6 and earlier, we could use GND to violate this invariant:


```wiki
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
module Main where

import MinList

class IntIso t where
    intIso :: c t -> c Int

instance IntIso Int where
    intIso = id

newtype Down a = Down a deriving (Eq, IntIso)

instance Ord a => Ord (Down a) where
    compare (Down a) (Down b) = compare b a

fine :: MinList (Down Int)
fine = foldl (\x y -> insertMinList x $ Down y) (newMinList $ Down 0) [-1,-2,-3,-4,1,2,3,4]

unsafeCast :: MinList (Down Int) -> MinList Int
unsafeCast = intIso

bad :: MinList Int
bad = unsafeCast fine

main = do
    print bad
```


Essentially, through GND we have created the function `unsafeCast :: MinList (Down Int) -> MinList Int`. This is a function we can't write by hand since we don't have access to the `MinList` constructor and so can't "see" into the data type.


## Safe Haskell Pre-GHC-7.8



Due to both the type-safety and abstraction issues, GND was considered unsafe in Safe Haskell.


