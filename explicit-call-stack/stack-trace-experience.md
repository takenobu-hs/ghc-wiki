# Experience with existing stack tracing tools



Before we decide what kind of stack trace we want, it is very useful to see what the other existing tools already provide. This will give us some ideas about what is workable in practice.



The tools we will be testing are:


- Hat 2.05, particularly hat-stack and hat-trail (hat stack is just a specialisation of hat-trail).
- Cost Centre Stacks (CCS) with +RTS -xc -RTS, using ghc version 6.6
- Andy Gill's HpcT tracer
- The ghci debugger extended with a simple stack passing transformation. We will try both a full transformation and some kind of partial transformation (where only part of the code is transformed). This is done using a modified version of buddha, which transforms the original program to one which passes stack arguments explicitly. You can find it here: [
  http://www.csse.unimelb.edu.au/\~bjpop/buddha-stack.tar.gz](http://www.csse.unimelb.edu.au/~bjpop/buddha-stack.tar.gz)

## Test cases



The test suite is a selection of programs from the nofig-buggy suite provided by the group at Technical University of Valencia. It is a modified version of the usual nofib benchmark suite. The suite can be obtained like so:


```wiki
   darcs get --partial http://einstein.dsic.upv.es/darcs/nofib-buggy/
```


Since we are interested in stack traces we will limit ourselves to programs with crash with an uncaught exception, such as divide by zero, head of empty list, calls to error, pattern match failure and so forth.



The actual test programs are:


- anna: Julien's strictness analyzer. It has been modified to divide by zero.

### Anna



We test the program on the `big.cor` input file, which causes a divide by zero error.



This is a rather simple bug:


```wiki
utRandomInts s1 s2
   = let seed1_ok = 1 <= s1 && s1 <= 2147483562
         seed2_ok = 1 <= s2 && s2 <= 2147483398

         rands :: Int -> Int -> [Int]
         rands s1 s2
            = let k    = s1 `div` 53668
                  s1'  = 40014 * (s1 - k * 53668) - k * 12211
                  s1'' = if s1' < 0 then s1' + 2147483563 else s1'
                -- BUG: The following line contains a bug
                  k'   = s2 `div` (s1' - s1')
                -- CORRECT -- k'   = s2 `div` 52774
```


The actual bug is in k', and it is local to that definition (it does not depen on values which flow into k').



Nonetheless, utRandomInts seems to be called in a deep context.


## Test results


### Test 1, anna, divide by zero error


#### hat



Changes made to code to get it to work:


```wiki
 darcs whatsnew
{
hunk ./real/anna/TypeCheck5.hs 14
+default ()
+
hunk ./real/anna/TypeCheck5.hs 500
-tcNSdlimit = 2^30
+tcNSdlimit = 2^(30::Int)
}
```


I had to add the type annotation on the literal 30 because defaulting doesn't work in hat.



Commands to prepare for tracing:


```wiki
    hmake -hat Main
```


Commands to see stack trace:


```wiki
   ./Main < big.cor
   hat-stack Main | less
```


Output (edited to make it easier to read and compare with other approaches):


```wiki
(Utils.hs:108)           div
(unknown)                k'
(Utils.hs:119)           rands
(Utils.hs:118)           utRandomInts
(FrontierDATAFN2.hs:243) utRandomInts
(FrontierDATAFN2.hs:75)  fdFs2
(FrontierGENERIC2.hs:57) fdFind
(StrictAn6.hs:383)       fsMakeFrontierRep
(StrictAn6.hs:316)       saNonRecSearch
(unknown)                result
(StrictAn6.hs:167)       saNonRecStartup
(unknown)                callSearchResult
(StrictAn6.hs:59)        saGroups
(unknown)                saResult
(Main.hs:122)            saMain
(Main.hs:120)            strictAnResults
(unknown)                strictAnResults
(Main.hs:166)            maStrictAn
(Main.hs:164)            primIOBind {IO} do
(Main.hs:162)            primIOBind {IO} do
(Main.hs:159)            primIOBind {IO} do
(unknown)                main

```

### Cost centre stacks



Following the instructions from [http://www.haskell.org/ghc/docs/latest/html/users\_guide/runtime-control.html\#rts-options-debugging](http://www.haskell.org/ghc/docs/latest/html/users_guide/runtime-control.html#rts-options-debugging)



Program compiled with ghc 6.6 like so:


```wiki
    ghc --make -prof -auto-all Main.hs
```


Program run like so:


```wiki
   ./Main +RTS -xc -RTS
```


Stack trace generated like so:


```wiki
   <GHC.Err.CAF>Main: divide by zero
```


Not very helpful!



Simon M correctly points out that the problem is that the exception is raised inside a CAF. Here is the way div is defined (for Int32, but the other types are similar):


```wiki
    div     x@(I32# x#) y@(I32# y#)
        | y == 0                  = divZeroError
        | x == minBound && y == (-1) = overflowError
        | otherwise               = I32# (x# `divInt32#` y#)
```


Note the reference to `divZeroError` in the first guarded equation. This is defined as:


```wiki
divZeroError :: a
divZeroError = throw (ArithException DivideByZero)
```


Hence it is a CAF. This is exactly the CAF that `GHC.Err.CAF` is refering to in the output from the cost centre stacks above. 



Now, we get muich better results if we raise the exception inside the user's code. For instance, if we change the definition of the local variable `k'` like so:


```wiki
k'   = error "bjpop crash"
```


We get the following output from running the program with `-xc` (edited to make it easier to compare with other traces):


```wiki
Utils.utRandomInts
FrontierDATAFN2.fdFs2
FrontierDATAFN2.fdFind
FrontierGENERIC2.fsMakeFrontierRep
StrictAn6.saNonRecSearch
StrictAn6.saNonRecStartup
StrictAn6.saGroups
StrictAn6.saMain
Main.maStrictAn
Main.main
Main.CAF
```


This is much better, and it is the same stack as produced by hat and the explicit stack passing transformation described below (minus the references to locally defined entities). 



It is worth noting that, had we transformed the libraries with the explicit stack passing transformation (below), we would get the same bad behaviour as the original output from cost centre stacks. 


### Stack passing transformation



I generated this trace by transforming the program using a heavily modified version of buddha. It implements the transformation as described on [
http://hackage.haskell.org/trac/ghc/wiki/ExplicitCallStack](http://hackage.haskell.org/trac/ghc/wiki/ExplicitCallStack), under the heading **Transformation option 1**.



Then I ran the transformed program inside the ghci debugger, and set a breakpoint manually around the call to div (that is, the call to div in the transformed version of the program, not the original version). This works because the transformation adds a new arguments to each function to pass stacks around, and the ghci debugger can view all the arguments to a function when it hits a breakpoint. Obviously it would be automated in practice, but this method is good enough for the purpose of this experiment.



Here is the generated stack (edited to make it easier to read and compare with others):


```wiki
                         (div would be here)
(Utils.hs:108)           k'
(Utils.hs:103)           rands
(Utils.hs:98)            utRandomInts
(FrontierDATAFN2.hs:241) (final_yy, final_xx, finalMemo)
(FrontierDATAFN2.hs:236) fdFs2
(FrontierDATAFN2.hs:74)  (fr, new_memo_additions)
(FrontierDATAFN2.hs:68)  fdFind
(FrontierGENERIC2.hs:56) (data_fn_result, final_memo)
(FrontierGENERIC2.hs:33) fsMakeFrontierRep
(StrictAn6.hs:382)       (next_safe, next_safe_evals)
(StrictAn6.hs:344)       saNonRecSearch
(StrictAn6.hs:315)       result
(StrictAn6.hs:287)       saNonRecStartup
(StrictAn6.hs:166)       callSearchResult
(StrictAn6.hs:133)       saGroups
(StrictAn6.hs:58)        saResult
(StrictAn6.hs:40)        saMain
(Main.hs:119)            strictAnResults
(Main.hs:101)            maStrictAn
(Main.hs:158)            main
```


Notice that it is largely the same thing as the one produced by Hat. The main differences are that the stack passing transformation records entries for local pattern bindings, but it seems that Hat does not. Also there are a couple of spurious looking duplicates in the Hat stack that I don't understand (which may be bugs in Hat?). And Hat seems to make some record of the calls to bind in the do notation. I didn't pass stacks to any standard library functions.


