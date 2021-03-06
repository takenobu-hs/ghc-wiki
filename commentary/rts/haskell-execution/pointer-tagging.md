# Pointer Tagging



Paper: [
Faster laziness using dynamic pointer tagging](http://research.microsoft.com/en-us/um/people/simonpj/papers/ptr-tag/ptr-tagging.pdf)



In GHC we "tag" pointers to heap objects with information about the object they point to.  The tag goes in the low 2 bits (3 bits on a 64-bit platform) of the pointer, which would normally be zero since heap objects are always [word](commentary/rts/word)-aligned.


## Meaning of the tag bits



The way the tag bits are used depends on the type of object pointed to:


- If the object is a **constructor**, the tag bits contain the *constructor tag*, if the number of
  constructors in the datatype is less than 4 (less than 8 on a 64-bit platform).  If the number of
  constructors in the datatype is equal to or more than 4 (resp 8), then the tag bits have the value 1, and the constructor tag
  is extracted from the constructor's info table instead.

- If the object is a **function**, the tag bits contain the *arity* of the function, if the arity fits
  in the tag bits.

- For a pointer to any other object, the tag bits are always zero.

## Optimisations enabled by tag bits



The presence of tag bits enables certain optimisations:


- In a case-expression, if the variable being scrutinised has non-zero tag bits, then we know
  that it points directly to a constructor and we can avoid *entering* it to evaluate it.
  Furthermore, for datatypes with only a few constructors, the tag bits will tell us *which*
  constructor it is, eliminating a further memory load to extract the constructor tag from the
  info table.

- In a [generic apply](commentary/rts/haskell-execution/function-calls#), if the function being applied has a tag value that indicates it has exactly the
  right arity for the number of arguments being applied, we can jump directly to the function, instead of
  inspecting its info table first.


Pointer-tagging is a fairly significant optimisation: we measured 10-14% depending on platform.  A large proportion of this comes from eliminating the indirect jumps in a case expression, which are hard to predict by branch-prediction.  The paper has full results and analysis.


## Garbage collection with tagged pointers



The [garbage collector](commentary/rts/storage/gc) maintains tag bits on the pointers it traverses.  This is easier, it turns out, than *reconstructing* tag bits.  Reconstructing tag bits would require that the GC knows not only the tag of the constructor (which is in the info table), but also the family size (which is currently not in the info table), since a constructor from a large family should always have tag 1.  To make this practical we would probably need different closure types for "small family" and "large family" constructors, and we already subdivide the constructor closures types by their layout.



Additionally, when the GC eliminates an indirection it takes the tag bits from the pointer inside the indirection.  Pointers to indirections always have zero tag bits.


## Invariants



Pointer tagging is *not* optional, contrary to what the paper says.  We originally planned that it would be: if the GC threw away all the tags, then everything would continue to work albeit more slowly.  However, it turned out that in fact we really want to assume tag bits in some places:


- In the continuation of an algebraic case, R1 is assumed tagged
- On entry to a non-top-level function, R1 is assumed tagged


If we don't assume the value of the tag bits in these places, then extra code is needed to untag the pointer.  If we can assume the value of the tag bits, then we just take this into account when indexing off R1.



This means that everywhere that enters either a case continuation or a non-top-level function must ensure that R1 is correctly tagged.  For a case continuation, the possibilities are:


- the scrutinee of the case jumps directly to the alternative if R1 is already tagged.
- the constructor entry code returns to an alternative.  This code adds the correct tag.
- if the case alternative fails a heap or stack check, then the RTS will re-enter the alternative after
  GC.  In this case, our re-entry arranges to enter the constructor, so we get the correct tag by
  virtue of going through the constructor entry code.


For a non-top-level function, the cases are:


- unknown function application goes via `stg_ap_XXX` (see [Generic Apply](commentary/rts/haskell-execution/function-calls#)).  
  The generic apply functions must therefore arrange to correctly tag R1 before entering the function.
- A known function can be entered directly, if the call is made with exactly the right number of arguments.
- If a function fails its heap check and returns to the runtime to garbage collect, on re-entry the closure
  pointer must be still tagged.
- the PAP entry code jumps to the function's entry code, so it must have a tagged pointer to the function
  closure in R1.  We therefore assume that a PAP always contains a tagged pointer to the function closure.


In the second case, calling a known non-top-level function must pass the function closure in R1, and this pointer *must* be correctly tagged.  The code generator does not arrange to tag the pointer before calling the function; it assumes the pointer is already tagged.  Since we arrange to tag the pointer when the closure is created, this assumption is normally safe.  However, if the pointer has to be saved on the stack, say across a call, then when the pointer is retrieved again we must either retag it, or be sure that it is still tagged.  Currently we do the latter, but this imposes an invariant on the garbage collector: all tags must be retained on non-top-level function pointers.



Pointers to top-level functions are not necessarily tagged, because we don't always know the arity of a function that resides in another module.  When optimisation is on, we do know the arities of external functions, and this information is indeed used to tag pointers to imported functions, but when optimisation is off we do not have this information.  For constructors, the interface doesn't contain information about the constructor tag, except that there may be an unfolding, but the unfolding is not necessarily reliable (the unfolding may be a constructor application, but in reality the closure may be a CAF, e.g. if any of the fields are references outside the current shared library).


## Compacting GC



Compacting GC also uses tag bits, because it needs to distinguish between a heap pointer and an info pointer quickly.  The compacting GC has a complicated scheme to ensure that pointer tags are retained, see the comments in [rts/sm/Compact.c](/trac/ghc/browser/ghc/rts/sm/Compact.c).


## Dealing with tags in the code



Every time we dereference a pointer to a heap object, we must first zero the tag bits.  In the RTS, this is done with the inline function (previously: macro) `UNTAG_CLOSURE()`; in `.cmm` code this is done with the `UNTAG()` macro.  Surprisingly few places needed untagging to be added.


