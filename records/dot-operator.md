## Details on the dot



Records proposal requires the current Haskell function composition dot operator to have spaces on both sides. No spaces around the dot are reserved for name-spacing: this use and the current module namespace use. No space to the right would be partial application (see 
[
TDNR](http://hackage.haskell.org/trac/haskell-prime/wiki/TypeDirectedNameResolution). The dot operator should bind as tightly as possible.


### Summary



The community needs to deprecate the usage of dot for function composition, perhaps starting with requiring spaces around the function composition dot. We should move to a new ascii operator and a unicode dot operator.


### Partial application



see [
TDNR](http://hackage.haskell.org/trac/haskell-prime/wiki/TypeDirectedNameResolution) syntax discusion for an explanation.


```wiki
(.a) r == r.a
```


.x (no space after the dot), for any identifier x, is a postfix operator that binds more tightly than function application, so that parentheses are not usually required.


```wiki
.a r == r.a
```


When there are multiple operators, they chain left to right


```wiki
(r.a.b.c) == (.c $ .b $ .a r)
```


See below for how partial application can allow for different code styles.



Question: does this now hold?


```wiki
r.a == r.(Record.a) == r.Record.a
```

## Dealing with dot-heavy code


### Identifying the difference between a name-space dot and function composition



Given the dot's expanded use here, plus its common use in custom operators, it is possible to end up with dot-heavy code.


```wiki
quux (y . (foo>.<  bar).baz (f . g)) moo
```


It's not that easy to distinguish from


```wiki
quux (y . (foo>.<  bar) . baz (f . g)) moo
```


What then is the future of the dot if this proposal is accepted? The community needs to consider ways to reduce the dot:



1) discourage the use of dot in custom operators: `>.<` could be discouraged, use a different character or none: `><`
In most cases the dot in custom operators has little to no inherent meaning. Instead it is just the character available for custom operators that takes up the least real-estate. This makes it the best choice for implementing a custom operator modeled after an existing Haskell operator: `.==` or `.<` is normably preferable to `@==` and `@<`.



2) discourage the use of dot for function composition - use a different operator for that task. Indeed, Frege users have the choice between `<~` or the proper unicode dot.
Haskell also has `Control.Category.<<<`



Discouraging the use of the dot in custom operators makes the example code only slightly better. With using a different operator we now have:


```wiki
quux (y <~ (foo>.<  bar).baz (f <~ g)) moo
```


Very easy to distinguish from


```wiki
quux (y <~ (foo>.<  bar) <~ baz (f <~ g)) moo
```


If you are disgusted by `<~` than you can use the very pretty unicode dot. Or we can stick with the category operator `<<<` instead of `<~`:


```wiki
quux (y <<< (foo>.<  bar).baz (f <<< g)) moo
```

### Editor mappings



Users can attempt to always save their code with ascii symbols, but use an editor mapping to display unicode symbols. This is a nice tool to have available, but personally I found the available vim mapper for Haskell to be slightly buggy, and we cannot expect every one to use mappings (but we can expect them to use utf8 aware editors).


### Downside: mixing of 2 styles of code


```wiki
data Record = Record { a::String }
b :: Record -> String

let r = Record "a" in b r.a 
```


It bothers some that the code does not read strictly left to right as in: `b . a . r`. Chaining can make this even worse: `(e . d) r.a.b.c`


#### Solution: Partial Application



Partial application provides a potential solution: `b . .a $ r`



So if we have a function `f r = b r.a` then one can write it points-free: `b . .a`



Our longer example from above: `e . d . .c . .b . .a`



Let us consider real use with longer names:


```wiki
echo . delta . .charlie . .beta . .alpha
```


Note that a move to a different operator for function composition (see discussion above) would make things much nicer:


```wiki
echo <~ delta <~ .charlie <~ .beta <~ .alpha
```

#### Solution: Field selector to the left of the record



We could have an equivalent of the dot where the field is to the left of the record: `b a@r`
Could this also be used in a partial syntax?


```wiki
echo . delta . charlie@ . beta@ . alpha@
```


Can this be shortened to:


```wiki
echo . delta . charlie@beta@alpha@
```


Or would this syntax alway need to be applied?


```wiki
echo . delta $ charlie@beta@alpha@r
```


**See also:** [Dot as Postfix Function Apply](records/declared-overloaded-record-fields/dot-postfix), part of the Declared Overloaded Record Fields proposal, where a record selector is just a function (overloaded). So dot notation is just postfix (reverse) function application (tight-binding), and can be used for any function. -- AntC 22-Feb-2012


