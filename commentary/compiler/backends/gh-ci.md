# The GHC Commentary: GHCi



This isn't a coherent description of how GHCi works, sorry. What it is (currently) is a dumping ground for various bits of info pertaining to GHCi, which ought to be recorded somewhere.


## Debugging the interpreter



The usual symptom is that some expression / program crashes when running on the interpreter (commonly), or gets wierd results (rarely). Unfortunately, finding out what the problem really is has proven to be extremely difficult. In retrospect it may be argued a design flaw that GHC's implementation of the STG execution mechanism provides only the weakest of support for automated internal consistency checks. This makes it hard to debug.



Execution failures in the interactive system can be due to problems with the bytecode interpreter, problems with the bytecode generator, or problems elsewhere. From the bugs seen so far, the bytecode generator is often the culprit, with the interpreter usually being correct.



Here are some tips for tracking down interactive nonsense:


- Find the smallest source fragment which causes the problem.

- Using an RTS compiled with `-DDEBUG`, run with `+RTS -Di` to get a listing in great detail from the interpreter. Note that the listing is so voluminous that this is impractical unless you have been diligent in the previous step.

- At least in principle, using the trace and a bit of GDB poking around at the time of death (See also [Debugging](debugging)), you can figure out what the problem is. In practice you quickly get depressed at the hopelessness of ever making sense of the mass of details. Well, I do, anyway.

- `+RTS -Di` tries hard to print useful descriptions of what's on the stack, and often succeeds. However, it has no way to map addresses to names in code/data loaded by our runtime linker. So the C function `ghci_enquire` is provided. Given an address, it searches the loaded symbol tables for symbols close to that address. You can run it from inside GDB:

```wiki
(gdb) p ghci_enquire ( 0x50a406f0 )
0x50a406f0 + -48  ==  `PrelBase_Czh_con_info'
0x50a406f0 + -12  ==  `PrelBase_Izh_static_info'
0x50a406f0 + -48  ==  `PrelBase_Czh_con_entry'
0x50a406f0 + -24  ==  `PrelBase_Izh_con_info'
0x50a406f0 +  16  ==  `PrelBase_ZC_con_entry'
0x50a406f0 +   0  ==  `PrelBase_ZMZN_static_entry'
0x50a406f0 + -36  ==  `PrelBase_Czh_static_entry'
0x50a406f0 + -24  ==  `PrelBase_Izh_con_entry'
0x50a406f0 +  64  ==  `PrelBase_EQ_static_info'
0x50a406f0 +   0  ==  `PrelBase_ZMZN_static_info'
0x50a406f0 +  48  ==  `PrelBase_LT_static_entry'
$1 = void
```

>
>
> In this case the enquired-about address is `PrelBase_ZMZN_static_entry`. If no symbols are close to the given addr, nothing is printed. Not a great mechanism, but better than nothing.
>
>

- We have had various problems in the past due to the bytecode generator (compiler/ghci/ByteCodeGen.lhs) being confused about the true set of free variables of an expression. The compilation scheme for `let`s applies the BCO for the RHS of the `let` to its free variables, so if the free-var annotation is wrong or misleading, you end up with code which has wrong stack offsets, which is usually fatal.

- Following the traces is often problematic because execution hops back and forth between the interpreter, which is traced, and compiled code, which you can't see. Particularly annoying is when the stack looks OK in the interpreter, then compiled code runs for a while, and later we arrive back in the interpreter, with the stack corrupted, and usually in a completely different place from where we left off.

>
>
> If this is biting you baaaad, it may be worth copying sources for the compiled functions causing the problem, into your interpreted module, in the hope that you stay in the interpreter more of the time.
>
>

- There are various commented-out pieces of code in Interpreter.c which can be used to get the stack sanity-checked after every entry, and even after after every bytecode instruction executed. Note that some bytecodes (`PUSH_UBX`) leave the stack in an unwalkable state, so the `do_print_stack` local variable is used to suppress the stack walk after them. 

## Useful stuff to know about the interpreter



The code generation scheme is straightforward (naive, in fact). `-ddump-bcos` prints each BCO along with the Core it was generated from, which is very handy.


- Simple `let`s are compiled in-line. For the general case, `let v = E in ...`, the expression `E` is compiled into a new BCO which takes as args its free variables, and `v` is bound to `AP(the new BCO, free vars of E)`.

- `case`s as usual, become: push the return continuation, enter the scrutinee. There is some magic to make all combinations of compiled/interpreted calls and returns work, described below. In the interpreted case, all `case` alts are compiled into a single big return BCO, which commences with instructions implementing a switch tree. 

### Stack management



There isn't any attempt to stub the stack, minimise its growth, or generally remove unused pointers ahead of time. This is really due to laziness on my part, although it does have the minor advantage that doing something cleverer would almost certainly increase the number of bytecodes that would have to be executed. Of course we `SLIDE` out redundant stuff, to get the stack back to the sequel depth, before returning a HNF, but that's all. As usual this is probably a cause of major space leaks.


### Building constructors



Constructors are built on the stack and then dumped into the heap with a single `PACK` instruction, which simply copies the top N words of the stack verbatim into the heap, adds an info table, and zaps N words from the stack. The constructor args are pushed onto the stack one at a time. One upshot of this is that unboxed values get pushed untaggedly onto the stack (via `PUSH_UBX`), because that's how they will be in the heap. That in turn means that the stack is not always walkable at arbitrary points in BCO execution, although naturally it is whenever GC might occur.



Function closures created by the interpreter use the AP-node (tagged) format, so although their fields are similarly constructed on the stack, there is never a stack walkability problem.


### Perspective



I designed the bytecode mechanism with the experience of both STG hugs and Classic Hugs in mind. The latter has an small set of bytecodes, a small interpreter loop, and runs amazingly fast considering the cruddy code it has to interpret. The former had a large interpretative loop with many different opcodes, including multiple minor variants of the same thing, which made it difficult to optimise and maintain, yet it performed more or less comparably with Classic Hugs.



My design aims were therefore to minimise the interpreter's complexity whilst maximising performance. This means reducing the number of opcodes implemented, whilst reducing the number of insns despatched. In particular, very few (TODO How many? Which?) opcodes which deal with tags. STG Hugs had dozens of opcodes for dealing with tagged data. Finally, the number of insns executed is reduced a little by merging multiple pushes, giving `PUSH_LL` and `PUSH_LLL`. These opcode pairings were determined by using the opcode-pair frequency profiling stuff which is ifdef-d out in Interpreter.c. These significantly improve performance without having much effect on the ugliness or complexity of the interpreter.



Overall, the interpreter design is something which turned out well, and I was pleased with it. Unfortunately I cannot say the same of the bytecode generator.


## case returns between interpreted and compiled code



Variants of the following scheme have been drifting around in GHC RTS documentation for several years. Since what follows is actually what is implemented, I guess it supersedes all other documentation. Beware; the following may make your brain melt. In all the pictures below, the stack grows downwards.


### Returning to interpreted code.



Interpreted returns employ a set of polymorphic return infotables. Each element in the set corresponds to one of the possible return registers (R1, D1, F1) that compiled code will place the returned value in. In fact this is a bit misleading, since R1 can be used to return either a pointer or an int, and we need to distinguish these cases. So, supposing the set of return registers is {R1p, R1n, D1, F1}, there would be four corresponding infotables, stg\_ctoi\_ret\_R1p\_info, etc. In the pictures below we call them stg\_ctoi\_ret\_REP\_info.



These return itbls are polymorphic, meaning that all 8 vectored return codes and the direct return code are identical.



Before the scrutinee is entered, the stack is arranged like this:


```wiki
   |        |
   +--------+
   |  BCO   | -------> the return contination BCO
   +--------+
   | itbl * | -------> stg_ctoi_ret_REP_info, with all 9 codes as follows:
   +--------+
                          BCO* bco = Sp[1];
                          push R1/F1/D1 depending on REP
                          push bco
                          yield to sched
```


On entry, the interpreted contination BCO expects the stack to look like this:


```wiki
   |        |
   +--------+
   |  BCO   | -------> the return contination BCO
   +--------+
   | itbl * | -------> ret_REP_ctoi_info, with all 9 codes as follows:
   +--------+
   : VALUE  :  (the returned value, shown with : since it may occupy
   +--------+   multiple stack words)
```


A machine code return will park the returned value in R1/F1/D1, and enter the itbl on the top of the stack. Since it's our magic itbl, this pushes the returned value onto the stack, which is where the interpreter expects to find it. It then pushes the BCO (again) and yields. The scheduler removes the BCO from the top, and enters it, so that the continuation is interpreted with the stack as shown above.



An interpreted return will create the value to return at the top of the stack. It then examines the return itbl, which must be immediately underneath the return value, to see if it is one of the magic stg\_ctoi\_ret\_REP\_info set. Since this is so, it knows it is returning to an interpreted contination. It therefore simply enters the BCO which it assumes it immediately underneath the itbl on the stack.


### Returning to compiled code.



Before the scrutinee is entered, the stack is arranged like this:


```wiki
                        ptr to vec code 8 ------> return vector code 8
   |        |           ....
   +--------+           ptr to vec code 1 ------> return vector code 1
   | itbl * | --        Itbl end
   +--------+   \       ....   
                 \      Itbl start
                  ----> direct return code
```


The scrutinee value is then entered. The case continuation(s) expect the stack to look the same, with the returned HNF in a suitable return register, R1, D1, F1 etc.



A machine code return knows whether it is doing a vectored or direct return, and, if the former, which vector element it is. So, for a direct return we jump to `Sp[0]`, and for a vectored return, jump to `((CodePtr*)(Sp[0]))[ - ITBL_LENGTH - vector number ]`. This is (of course) the scheme that compiled code has been using all along.



An interpreted return will, as described just above, have examined the itbl immediately beneath the return value it has just pushed, and found it not to be one of the ret\_REP\_ctoi\_info set, so it knows this must be a return to machine code. It needs to pop the return value, currently on the stack, into R1/F1/D1, and jump through the info table. Unfortunately the first part cannot be accomplished directly since we are not in Haskellised-C world.



We therefore employ a second family of magic infotables, indexed, like the first, on the return representation, and therefore with names of the form stg\_itoc\_ret\_REP\_info. (Note: itoc; the previous bunch were ctoi). This is pushed onto the stack (note, tagged values have their tag zapped), giving:


```wiki
   |        |
   +--------+
   | itbl * | -------> arbitrary machine code return itbl
   +--------+
   : VALUE  :  (the returned value, possibly multiple words)
   +--------+
   | itbl * | -------> stg_itoc_ret_REP_info, with code:
   +--------+
                          pop myself (stg_itoc_ret_REP_info) off the stack
                          pop return value into R1/D1/F1
                          do standard machine code return to itbl at t.o.s.
```


We then return to the scheduler, asking it to enter the itbl at t.o.s. When entered, stg\_itoc\_ret\_REP\_info removes itself from the stack, pops the return value into the relevant return register, and returns to the itbl to which we were trying to return in the first place.



Amazingly enough, this stuff all actually works! Well, mostly ...


## Unboxed tuples: a Right Royal Spanner In The Works



The above scheme depends crucially on having magic infotables stg\_{itoc,ctoi}\_ret\_REP\_info for each return representation REP. It unfortunately fails miserably in the face of unboxed tuple returns, because the set of required tables would be infinite; this despite the fact that for any given unboxed tuple return type, the scheme could be made to work fine.



This is a serious problem, because it prevents interpreted code from doing IO-typed returns, since IO t is implemented as `(# t, RealWorld# #)` or thereabouts. This restriction in turn rules out FFI stuff in the interpreter. Not good.



Although we have no way to make general unboxed tuples work, we can at least make IO-types work using the following ultra-kludgey observation: `RealWorld#` doesn't really exist and so has zero size, in compiled code. In turn this means that a type of the form `(# t, RealWorld# #)` has the same representation as plain t does. So the bytecode generator, whilst rejecting code with general unboxed tuple returns, recognises and accepts this special case. Which means that IO-typed stuff works in the interpreter. Just.



If anyone asks, I will claim I was out of radio contact, on a 6-month walking holiday to the south pole, at the time this was ... er ... dreamt up.


