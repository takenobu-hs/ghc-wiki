# The magic `oneShot` function


## Synopsis



This documents the magic function


```wiki
GHC.Exts.oneShot :: (a -> b) -> (a -> b)
```


Semantically, it is the identity, but in addition it tell the compiler to assume that `oneShot f` is called at most once, which allows it to optimize the code more aggressively.


## Motivation



The idea first came up in the context of making foldl a good consumer: [ticket:7994\#comment:7](http://gitlabghc.nibbler/ghc/ghc/issues/7994)



With the implementation


```wiki
foldl k z xs = foldr (\v fn -> oneShot (\z -> fn (k z v))) id xs z
```


the regular arity analysis of GHC would be able to clean up after fusing this with a consumer. In this sense, the `oneShot` is an alternative to [CallArity](call-arity) (but they are not mutually exclusive, and we probably want both).



Later, David Feuer creatd more good consumers that would rely on such eta-expansion to produce good code, such as `scanl`. Using `oneShot` in these would make this transformation more reliable, in particular when [CallArity](call-arity) fails.



Nofib itself does not exhibit any case where adding `oneShot` improves performance over Call Arity. But there is still a good chance that such cases occur out there.


## Implementation



GHC already supports one-shot lambdas, see `setOneShotLambda` in `Id.lhs`. We implemented `oneShot` as a built in function, similar to `lazy` etc. The crucial bit is to apply `setOneShotLambda` on the lambda’s binder in the unfolding, and to inline `oneShot` aggressively. This is [\[c271e32eac65ee95ba1aacc72ed1b24b58ef17ad\]](/trac/ghc/changeset/c271e32eac65ee95ba1aacc72ed1b24b58ef17ad/ghc)



Also, we need the `oneShot` information needs to prevail in unfoldings and across interfaces. For that, `IfaceExpr` was extended with a boolean flag on lambda binders in [\[c001bde73e38904ed161b0b61b240f99a3b6f48d\]](/trac/ghc/changeset/c001bde73e38904ed161b0b61b240f99a3b6f48d/ghc).



Finally, `oneShot` is actually used in the libraries. This is [\[072259c78f77d6fe7c36755ebe0123e813c34457\]](/trac/ghc/changeset/072259c78f77d6fe7c36755ebe0123e813c34457/ghc).


## Challenges


### Preservation of `setOneShotLambda` in Core2Core transformations



Previously, the `oneShotInfo` was not very valuable: It was determined by the compiler itself (TODO Where exactly?), so not much is lost if a transformation would reset the flag, as a later phase could re-calculate it.



Now, the `oneShotInfo` can come from the use as well, and resetting it would lose the information irrevocably. Therefore, transformations need to be more careful about preserving it, while keeping it correct.



For example the a CSE run could transform


```wiki
   let f = \x[OneShot] -> ...x...
       g = \x[OneShot] -> ...x...
   in f () + g ()
```


to


```wiki
   let f = \x[OneShot] -> ...x...
   in f () + g ()
```


but now the `OneShot` flag is not true any more. So likely it should be reset. OTOH, this might contradict the user’s intentions. Should CSE be aware of this and avoid CSE’ing these function? Or maybe leave the `OneShot` in place, even if incorrect at first glance, on the grounds that the only effect an incorrect `OneShot` annotation has will be to un-do the CSE?



So far, this is a hypothetical challenge: It seems that the `oneShotInfo` is preserved rather well. We’ll see if something else comes up.


## Wild, unverified ideas



Can the IO state hack be avoided if `oneShot` is used in the right places in library code, e.g. in IO’s definition of `>>=`?


