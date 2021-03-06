# The Problem



The type level numbers of kind `Nat` have no structure,
which limits their use in programs that need to overload
values based on a natural number, or use programmer-defined
type functions.  Consider, for example, a class with a
parameter of kind `Nat`:


```wiki
class C (n :: Nat) where
  someMethod :: ...
```


To define instances of this class, we need to either select
a set of concrete numbers like this:


```wiki
instance C 0 where ...
instance C 1 where ...
```


Or, alternatively, we could provide a completely polymorphic instance:


```wiki
instance C a where ...
```


Usually, neither of these is good enough:  the first one is too restrictive as we
have to enumerate all the instances that will ever be used explicitly, while the
second one is too general and does not give us any static information about the
type that we are using (i.e., we could define the method without using a class).



The same sort of thing happens if we try to use numbers of kind `Nat` as the parametr
to a type function---we often can't define the instances that we need.


# A Solution



We can solve this problem by providing an additional representation of type-level natural numbers,
one that has explicit structure.  We define another kind, `Nat1`, which represents natural numbers
in the traditional unary representation (here we are using GHC's `DataKinds` extension):


```wiki
data Nat1 = Zero | Succ Nat1
```


This kind makes it easy to define class instances or type-functions using natural numbers.
For examples, here is a function that selects a type from a list of types:


```wiki
type family Get (n :: Nat1) (xs :: [*]) :: *
type instance Get Zero     (x `: xs) = x
type instance Get (Succ n) (x `: xs) = Get n xs
```


Such a function might be useful if we were defining some sort of safe interface
to a foreign struct:


```wiki
getField :: Selector n -> Ptr (Struct fields) -> Ptr (Get n fields)
```


(This is just an example---to make this work for real, we'd probably
have to use a type class so that we can determine the sizes of the struct fields.)



Unfortunately, if `getField` was defined with this type signature, we
wouldn't be able to use it with the type-level natural number literals
because `Selector` expects a type of kind `Nat1` and not a `Nat`.



To work around this, we can provide a type-function that relates the
two representations of natural numbers:


```wiki
type family FromNat1 (n :: Nat1) :: Nat
type instance FromNat1 Zero     = 0
type instance FromNat1 (Succ n) = 1 + FromNat1 n
```


By using `FromNat1`, we can give `getField` a type that will allow
for the use of ordinary type-level literals:


```wiki
getField :: Selector (FromNat1 n) -> Ptr (Struct fields) -> Ptr (Get n fields)
```


There is one final detail that needs to be explained: if we were to try the
definitions presented so far without any further modifications, we would get
ambiguity errors when trying to use `getField`.   The reason for this is that
the type variable `n` appears only in arguments to type-functions so GHC
has no way to determine its value from the result of the type function.



The good news is that the function `FromNat1` is injective so, in fact,
it is possible to determine the input from the output.  We modified GHC's
type checker to make it aware of this fact.  These changes are captured
by the following two rules:


```wiki
forall a b.         (FromNat1 a ~ FromNat1 b) => (a ~ b)
forall a. exists b. (1 <= FromNat1 a)         => (a ~ Succ b)
```


Now the function `getField` type-checks as expected:


```wiki
s :: Selector 2

p :: Ptr (Struct [Int,Char,Float])

f :: Ptr Float
f = getField s p
```