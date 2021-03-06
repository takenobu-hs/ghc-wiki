## Bundlings pattern synonyms with types



This section is based upon [\#10653](http://gitlabghc.nibbler/ghc/ghc/issues/10653) and D1258. The feature was implemented in [
96621b1b4979f449e873513e9de8d806257c9493](https://github.com/ghc/ghc/commit/96621b1b4979f449e873513e9de8d806257c9493).



Pattern synonyms allow for constructors to be defined, exported and imported separately from the types which they build.
However, it is sometimes convenient to bundle synonyms with types so that we can more closely model ordinary data constructors.



If we want to refactor to change the internal representation of this maybe-like type to use Maybe.


```wiki
-- Main.hs
module Main where

  import Internal ( A(..))
  ...more stuff using MkA...

-- Internal.hs
module Internal where

  data A = MkA Int | NoA
```


If we modify `Internal.hs` as follows


```wiki
{-# LANGUAGE PatternSynonyms #-}
module Internal where

  newtype A = NewA (Maybe Int)

  pattern MkA n = A (Just n)

  pattern NoA = A Nothing
```


Then local definitions to `Internal` which used `A` would work as before but modules importing `Internal` and `A`
will no longer work as importing `A(..)` will import the type `A` and the constructor `NewA`. We can explicitly import the
new patterns but the usage of pattern synonyms should be transparent to the end user. What's needed is to be able to
bundle the new synonyms with a type such that client code is oblivious to this implementation.


### Proposal



Richard proposes that synonyms are associated at the export of a datatype. Our running example would then look as follows:


```wiki
{-# LANGUAGE PatternSynonyms #-}
module Internal(A(MkA, NoA)) where

newtype A = NewA (Just Int)

pattern MkA n = A (Just n)

pattern NoA = A Nothing
```

### Specification



This proposal only changes module imports and exports. 


#### Definition



We say that "a pattern synonym `P` is associated with a type `T` relative to module `M`" if and only if "`M` exports `T` whilst associating `P`". 


#### Exports



For any modules `M` `N`, we say that "`M` exports `T` whilst associating `P`" just when


- The export has the form `T(c1, ..., cn, P)` where c1 to cn (n \>= 0) are a mixture of other field names, constructors, pattern synonyms and the special token `..`. The special token `..`, which indicates either 

  1. all constructors and field names from `T`'s declaration, if `T` is declared in this module; or 
  1. all symbols imported with `T`, which might perhaps include patterns associated with `T` in some other module.


In case (2), `..` might in fact be a union of sets if `T` is imported from multiple modules with different sets of associated definitions.


#### Imports



For any modules `M` `N`, if we import `N` from `M`,


- The abbreviated form `T(..)` brings into scope all the constructors, methods or field names exported by `N` as well any patterns bundled with `T` relative to `N`. 
- The explicit form `T(c1,...,cn)` can name any constructors, methods or field names exported by `N` as well as any patterns bundled with `T` relative to `N`. 

#### Typing



It proved quite a challenge to precisely specify which pattern synonyms
should be allowed to be bundled with which type constructors.
In the end it was decided to be quite liberal in what we allow. Below is
how Simon described the implementation.


>
>
> Personally I think we should Keep It Simple.  All this talk of
> satisfiability makes me shiver.  I suggest this: allow T( P ) in all
> situations except where `P`'s type is *visibly incompatible* with
> `T`.
>
>
>
> What does "visibly incompatible" mean?  `P` is visibly incompatible
> with `T` if
>
>
> - `P`'s type is of form `... -> S t1 t2`
> - `S` is a data/newtype constructor distinct from `T`
>
>
> Nothing harmful happens if we allow `P` to be exported with
> a type it can't possibly be useful for, but specifying a tighter
> relationship is very awkward as you have discovered.
>
>


Note that this allows \*any\* pattern synonym to be bundled with any
datatype type constructor. For example, the following pattern `P` can be
bundled with any type.


```wiki
pattern P :: (A ~ f) => f
```


So we provide basic type checking in order to help the user out, 
pattern synonyms are defined with definite type constructors, but don't
actually prevent a library author completely confusing their users if
they want to.



A few examples are included for clarification in the final section.


#### Clarification


- Hence, all synonyms must be initially explicitly associated but a module which imports an associated synonym is oblivious to whether they import a synonym or a constructor.

- According to this proposal, only pattern synonyms may be associated with a datatype. But it would be trivial to expand this proposal to allow arbitrary associations. 

#### Examples


```
module N(T(.., P)) where

data T = MkT Int

pattern P = MkT 5

-- M.hs
module M where

import N (T(..))
```


`P` is associated with `T` relative to `N`. M imports `T`, `MkT` and `P`.


```
module N(T(..)) where

data T = MkT Int

pattern P = MkT 5

-- M.hs
module M where

import N (T(..))
```


`P` is unassociated. `M` imports `T` and `MkT`. 


```
module N(T(P)) where

data T = MkT Int

pattern P = MkT 5

-- M.hs
module M where

import N (T(..))
```


`P` is associated with `T` relative to `N`. M imports `T`, and `P`.


```
module N(T(P)) where

data T = MkT Int

pattern P = MkT 5

-- M.hs
module M (T(..)) where

import N (T(..))

-- O.hs
module O where

import M (T(..))
```


`P` is associated with `T` relative to `N`.



As `M` imports `N` and imports `T`, `P` is associated with `T` relative to `M`. Thus `M` exports `T` and `P`.



Therefore when `O` imports `T(..)` from `M`, it imports `T` and `P`. 


```
module N(T(..)) where

data T = MkT Int

-- M.hs
module M(T(P)) where

import N (T(..))

pattern P = MkT 5

-- O.hs
module O where

import M (T(..))
```


This example highlights being able to freely reassociate synonyms. 



`M` imports `T` and `MkT` from `N` but then as `M` associates `P` with `T`, when `O` imports `M`, `T` and `P` are brought into scope. 


#### Typing Examples


```
{-# LANGUAGE PatternSynonyms #-}
module Foo (A(P)) where

data A = A

pattern P :: A
pattern P = A
```


Pattern `P` has type `A` therefore we allow this export.


```
{-# LANGUAGE PatternSynonyms #-}
module Foo (A(P)) where

data A = A

data B = B

pattern P :: B
pattern P = B
```


Pattern `P` has type `B` therefore we do not allow this export.


```
{-# LANGUAGE PatternSynonyms #-}
module Foo (A(P)) where

data A a = A

pattern P :: A Int
pattern P = A
```


Pattern `P` has type `A Int` which is an instance of `A a` therefore we allow
this export


##### Constraints


```
{-# LANGUAGE PatternSynonyms #-}
module Foo (A(P)) where

data A = A

pattern P :: () => (A ~ f) => f
pattern P = A
```


Pattern `P` has no visibly obvious type so we allow it to be bundled with any type constructor.


```
{-# LANGUAGE PatternSynonyms, ViewPatterns #-}
module Foo (Identity(P)) where

data Identity a = Identity a

instance C Identity where
  build a = Identity a
  destruct (Identity a) = a

class C f where
  build :: a -> f a
  destruct :: f a -> a

pattern P :: () => C f => a -> f a
pattern P x <- (destruct -> x)
  where
        P x = build x
```


In this example, `P` is once again polymorphic in the constructor `f` so it can be bundled with any type constructor.



We certainly do not want to allow the following association but we do anyway as the type is not visibly different.


```
{-# LANGUAGE PatternSynonyms #-}
module Foo (A(P)) where

data A = A

data B = B

pattern P :: () => (B ~ f) => f
pattern P = B
```


Things get even more hairy when we remember that classes can have equality constraints.
Consider this quite weird example.


```
{-# LANGUAGE PatternSynonyms #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE ViewPatterns #-}

module Foo ( A(P) ) where

class (f ~ A) => C f a where
  build :: a -> f a
  destruct :: f a -> a

data A a = A a

instance C A Int where
  build n = A n
  destruct (A n) = n


pattern P :: () => C f a => a -> f a
pattern P x <- (destruct -> x)
  where
        P x = build x
```


We should only allow `P` to be associated with `A` due to the superclass constraint
`f ~ A` but it can be bundled with any type constructor as it is not visibly different.


##### Type Families



The final example is when the result type is given by a type family.


```
{-# LANGUAGE PatternSynonyms #-}

module Foo ( A(P) ) where

data A = A

type family F a

pattern P :: F Bool
pattern P = A
```


We can bundle `P` with any type constructor as the type is not visibly different. 


### Unnatural Association



There is some discussion about what should happen with synonyms which target types defined in the prelude. 
It is very uncommon to explicitly import datatypes defined in the prelude, thus this kind of association is very rare in practice
but should be allowed.


### Module Chasing



Simon is a bit worried that once we allow this association then there is no limit to the number of constructors which
can be associated with a type. With normal datatypes, `T(..)` means some subset of the constructors defined where `T` is
defined. With this proposal `T(..)` has no maximal meaning, I don't see any problem with this behaviour as the meaning can still
be determined by the renamer.


### The Privilege Objection



Simon also wonders why it is possible to privilege pattern synonyms in this way and not normal functions for example.
I don't think this is particularly puzzling as at their most general pattern synonyms allow for the definition of unassociated 
data constructors. Being able to later associate them with types only brings their behaviour closer to ordinary data constructors.


### Polymorphic Synonyms



The following is a valid pattern synonym declaration which doesn't have a definite constructor.


```wiki
{-# LANGUAGE PatternSynonyms, ViewPatterns #-}
module Foo where

class C f where
  build :: a -> f a
  destruct :: f a -> a

pattern P :: () => C f => a -> f a
pattern P x <- (destruct -> x)
  where
    P x = build x
```


I propose that we allow such synonyms to be associated with a type `T` as long as it typechecks. I don't expect this to be much used in practice. 


### Associatation at definition



Simon proposed that synonyms are associated at the definition of a datatype. Our running example would look as follows:


```wiki
{-# LANGUAGE PatternSynonyms #-}
module Internal(A(MkA, NoA)) where

newtype A = NewA (Just Int)
    with (MkA, NoA)

pattern MkA n = A (Just n)

pattern NoA = A Nothing
```


The proposal refers to Richard's suggestion rather than Simon's refinement for the following reasons.



Consider two packages `old-rep` and `new-rep` which have  different representations of the same structure. 
The library author wants to smooth the transition for his users by providing a compatibility package `compat-rep`
so that code using the old representation in `old-rep` can work seamlessly with `new-rep`. 



The problem is to define `compat-rep` such that by changing the dependencies of our package, our code to continues to work
but without depending on `old-rep`. More generally, an author may want to write a `*-compat` package for two packages which they do
not control. Having to define these synonyms at the definition site is too restrictive for this case .


