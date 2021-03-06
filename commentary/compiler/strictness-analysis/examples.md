# Strictness analysis: examples



Consider:


```wiki
f g True = 3
f g False = g 1 2

...f (\ x -> let foo = somethingExpensive in \ y -> ...)...
```


We want to make sure to figure out that f's argument is demanded with type L1X(L1X(LMX)) -- that is, it may or may not be demanded, but if it is, it's always applied to two arguments. This shows why `deferType` shouldn't just throw away the argument info: in this case, the `(\ x -> ...)` expression has a nonstrict demand placed on it, yet we still care about the arguments.



On the other hand, in:


```wiki
foo x y = 
  case x of
     A -> \ z -> x*z
     B -> \ z -> x+z
```


we want to say that if the result of `foo` has demand `S` placed on it (i.e., not a call demand), the body of `foo` has demand `S` placed on it, not `S(LMX)`. So this case needs to be treated differently from the one above.


