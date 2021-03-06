
This is Joachim’s notepad about his plans to get rid of dynamic arity checks in the generated code (and hence also things like PAPs).


## Status quo



Core is a language where function types to not carry arity information (`Int→Int→Int` could be a binary function return `Int`, or a unary function returning `Int→Int`).



In STG, functions do have an arity, and calls to known functions are efficient in that they use the right calling convention. But because there is no type system any more, when we generate code for a call with, say, two arguments to an unknown function, the code has to handle the case of unary functions (putting one argument to the stack), binary functions (jumping right to them) or higher-arity functions (creating a PAP closure). This code is commoned up in commonly used code `stg_ap_pp_fast`, but even in the best case, there are two jumps involved.



Assume this code could simply rely on the function behind the pointer expecting two arguments, then we could get rid of the indirection via `stg_ap_pp_fast` and just jump there.


## Is it worth it



Before going in the how, the question is: Is it worth it?



Here is a micro benchmark:


```
-- Foo.hs
module Foo where

foo :: (Int -> Int -> Int) -> Int
foo f = go 0 0
  where go a 999999999 = a
        go a n = a `seq` go (f a n) (n+1)
{-# NOINLINE foo #-}

-- Main.hs
import Foo

fun :: Int -> Int -> Int
fun x y = (x*x)+(y*y)

main = print $ foo fun
```


I compiled this:


```
$ ghc -O -c -keep-s-files Foo.hs
$ ghc -O -c -keep-s-files Main.hs
$ ghc -O Foo.o Main.o -o test-slow
$ vim Foo.s # see below
$ ghc -c -O Foo.s
$ ghc -O Foo.o Main.o -o test-fast
```


In the edit step, I replaced


```
        movq %rbx,%rdx
        movq %rax,%rbx
        movq %rdx,-16(%rbp)
        movq %rcx,-8(%rbp)
        addq $-24,%rbp
        jmp stg_ap_pp_fast
        .size s1F8_info, .-s1F8_info
```


with


```
        movq %rbx,%rdx
        movq %rax,%rbx
        movq %rdx,-16(%rbp)
        movq %rcx,-8(%rbp)
        addq $-24,%rbp
        jmp *-2(%rax)
        .size s1F8_info, .-s1F8_info
```


(yes, that works!)



Then I timed the resulting binaries with


```
$ python -m timeit -n 10 -r 3 'import os; os.system("./test-fast")';
10 loops, best of 3: 4.94 sec per loop
$ python -m timeit -n 10 -r 3 'import os; os.system("./test-slow")';
10 loops, best of 3: 5.49 sec per loop
```


and found that the change improves runtime, in this extreme micro-benchmark, in this iteration, by 10% (and less when running it again ☹). Which is not great given the form of the benchmark, but at least a number.



Also, the real code would still have to check `%rax` for whether it is an evaluated function there, or a thunk that first needs entering.


