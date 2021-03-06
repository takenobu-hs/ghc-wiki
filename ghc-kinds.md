


# Kind polymorphism and datatype promotion



This page gives additional implementation details for the `-XPolyKinds` flag. The grand design is described in the paper [
Giving Haskell a Promotion](http://dreixel.net/research/pdf/ghp.pdf). Most of the work has been done and merged into GHC 7.4.1. The relevant user documentation is in \[the user's guide (add link when it's up)\] and on the [
Haskell wiki page](http://haskell.org/haskellwiki/GHC/Kinds). What still doesn't work, or doesn't work correctly, is described here.



Sub-pages


- [GhcKinds/KindPolymorphism](ghc-kinds/kind-polymorphism)
- [GhcKinds/PolyTypeable](ghc-kinds/poly-typeable) A kind-polymorphic version of the `Typeable` class.
- [GhcKinds/KindsWithoutData](ghc-kinds/kinds-without-data)
- [ExplicitTypeApplication](explicit-type-application) proposes a syntax for explicit kind application

---


# Future work


## Promoting data families



Consider this:


```wiki
  data family T a
  data instance T Int = MkT
  data Proxy (a :: k)
  data S = MkS (Proxy 'MkT)
```


Is it ok to use the promoted data family instance constructor `MkT` in
the data declaration for `S`?  No, we don't allow this. It *might* make
sense, but at least it would mean that we'd have to interleave
typechecking instances and data types, whereas at present we do data
types *then* instances.



A couple of people have asked about this


- [
  http://hackage.haskell.org/trac/ghc/wiki/Commentary/Compiler/GenericDeriving\#Digression](http://hackage.haskell.org/trac/ghc/wiki/Commentary/Compiler/GenericDeriving#Digression)
- [
  http://www.reddit.com/r/haskell/comments/u7oxb/is\_it\_possible\_to\_datakindlift\_a\_data\_family/](http://www.reddit.com/r/haskell/comments/u7oxb/is_it_possible_to_datakindlift_a_data_family/)


 


## [
\#5682](http://hackage.haskell.org/trac/ghc/ticket/5682) (proper handling of infix promoted constructors)



Bug report [ \#5682](http://hackage.haskell.org/trac/ghc/ticket/5682) shows a
problem in parsing promoted infix datatypes.



**Future work:** handle kind operators properly in the parser.


## Kind synonyms (from type synonym promotion)



At the moment we are not promoting type synonyms, i.e. the following is invalid:


```wiki
data Nat = Ze | Su Nat
type Nat2 = Nat

type family Add (m :: Nat2) (n :: Nat2) :: Nat2
```


We propose to change this, and make GHC promote
type synonyms to kind synonyms by default with `-XDataKinds`. For instance, `type String = [Char]`
should give rise to a kind `String`.



**Question:** are there dangerous interactions with `-XLiberalTypeSynonyms`? E.g. what's the kind
of *type K a = forall b. b -\> a\`?
*



By extension, we might want to have kind synonyms that do not arise from promotion: `type kind K ...`.
And perhaps even type synonyms that never give rise to a promoted kind: `type type T ...`.


## Generalized Algebraic Data Kinds (GADKs)



**Future work:** this section deals with a proposal to collapse kinds and sorts into a single system
so as to allow Generalised Algebraic DataKinds (GADKs). The sort `BOX` should
become a kind, whose *kind* is again `BOX`. Kinds would no longer be classified by sorts;
they would be classified by kinds.



(As an aside, sets containing themselves result in an inconsistent system; see, for instance,
[
this example](http://www.cs.nott.ac.uk/~txa/g53cfr/l20.agda). This is not of practical
concern for Haskell.)



Collapsing kinds and sorts would allow some form of indexing on kinds. Consider the
following two types, currently not promotable in FC-pro:


```wiki
data Proxy a = Proxy

data Ind (n :: Nat) :: * where ...
```


In `Proxy`, `a` has kind `forall k. k`. This type is not promotable because
`a` does not have kind `*`. This is unfortunate, since a new feature (kind
polymorphism) is getting on the way of another new feature (promoting
datatypes). As for `Ind`, it takes an argument of kind (promoted) `Nat`,
which renders it non-promotable. Why is this? Well, promoted `Proxy` and `Ind`
would have sorts:


```wiki
Proxy  :: forall s. s -> BOX

Ind    :: 'Nat -> BOX
```


But `s` is a sort variable, and `'Nat` is the sort arising from promoting
the kind `Nat` (which itself arose from promoting a datatype). FC-pro has
neither sort variables nor promoted sorts. However, if there are no sorts, and
`BOX` is the **kind** of all kinds, the "sorts" ("kinds", now) of promoted `Proxy`
and `Ind` become:


```wiki
Proxy  :: forall k. k  -> BOX

Ind    :: Nat          -> BOX
```


Now instead of sort variables we have kind variables, and we do not need to promote
`Nat` again.



Kind indexing alone should not require kind equality constraints; we always
require type/kind signatures for kind polymorphic stuff, so then
[
wobbly types](http://research.microsoft.com/en-us/um/people/simonpj/papers/gadt/gadt-rigid-contexts.pdf)
can be used to type check generalised algebraic kinds, avoiding the need for
coercions. While this would still require some implementation effort, it
should be "doable".


