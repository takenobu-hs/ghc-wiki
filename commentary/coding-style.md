


# The GHC Commentary - Coding Style Guidelines for the compiler



This is a rough description of some of the coding practices and style that we use for Haskell code inside `compiler`.  For run-time system code see the [Coding Style Guidelines for RTS C code](commentary/rts/conventions).  Also see the wiki page on [Working Conventions](working-conventions) for issues related to version control, workflow, testing, bug tracking and other miscellany.


## General Style



The general rule is to stick to the same coding style as is already used in the file you're editing. If you must make stylistic changes, commit them separately from functional changes, so that someone looking back through the change logs can easily distinguish them. 



It's much better to write code that is transparent than to write code that is short.



Often it's better to write out the code longhand than to reuse a generic abstraction (not always, of course).  Sometimes it's better to duplicate some similar code than to try to construct an elaborate generalisation with only two instances.  Remember: other people have to be able to quickly understand what you've done, and overuse of abstractions just serves to obscure the *really* tricky stuff, and there's no shortage of that in GHC.


## Comments



There are two kinds of comments in source code, comments that describe the interface (i.e. how is this supposed to be used) and comments that describe the implementation (e.g. subtle gotchas).


### Comments on top-level entities



Every top-level entity should have a Haddock comment that describes what it does and, if needed, why it's there. Example:


```wiki
-- | Returns which registers are read and written by this 
-- instruction, as a (read, written) pair. This info is used
-- by the register allocator.
x86_regUsageOfInstr :: Platform -> Instr -> RegUsage
```


We use Haddock so that the comment is included in the generated HTML documentation.



There's a bit of a broken window effect going on, but please try to follow this rule for new functions you add.


### Comments in the source code



Commenting is good but


- long comments *interleaved with the code* can make the code itself incredibly hard to read, and
- long comments *detached from the code* are easy to miss when you are editing the code itself, and soon become out of date or even misleading.


We have adopted a style that seems to help.  Here's an example:


```wiki
prepareRhs :: SimplEnv -> OutExpr -> SimplM (SimplEnv, OutExpr)
-- Adds new floats to the env iff that allows us to return a good RHS
prepareRhs env (Cast rhs co)    -- Note [Float coercions]
  | (ty1, _ty2) <- coercionKind co      -- Do *not* do this if rhs is unlifted 
  , not (isUnLiftedType ty1)            -- see Note [Float coercions (unlifted)]
  = do  { (env', rhs') <- makeTrivial env rhs
        ; return (env', Cast rhs' co) }

        ...more equations for prepareRhs....

{- Note [Float coercions]
~~~~~~~~~~~~~~~~~~~~~~
When we find the binding
        x = e `cast` co
we'd like to transform it to
        x' = e
        x = x `cast` co         -- A trivial binding
There's a chance that e will be a constructor application or function, or something
like that, so moving the coerion to the usage site may well cancel the coersions
and lead to further optimisation.  
        ...more stuff about coercion floating...
-}
```


Notice that


- **Interleaved with the code** is a short link `Note [Float coercions]`. You can't miss it when you are editing the code, but you can still see the code itself.
- **Detached from the code** is the linked comment, starting with the same string `Note [Float coercions]`.  It can be long, and often includes examples.


The standard format "`Note [Float coercions]`" serves like an URL, to point to an out-of-line comment.  Usually the target is in the same module, but not always.  Sometimes we say


```wiki
    -- See Note [Float coercions] in SpecConstr.lhs
```


Please use this technique.  It's robust, and survives successive changes to the same lines of code.  When you are changing code, it draws attention to non-obvious things you might want to bear in mind.  When you encounter the note itself you can search for the string to find the code that implements the thoughts contained in the comment.


### Comments and examples



When writing a comment to explain a subtle point, consider including an example code
snippet that illustrates the point.  For example, the above `Note [Float coercions]` continues thus:


```wiki
There's a chance that e will be a constructor application or function, or something
like that, so moving the coerion to the usage site may well cancel the coersions
and lead to further optimisation.  Example:

     data family T a :: *
     data instance T Int = T Int

     foo :: Int -> Int -> Int
     foo m n = ...
        where
          x = T m
          go 0 = 0
          go n = case x of { T m -> go (n-m) }
                -- This case should optimise
```


These kind of code snippets are extremely helpful to illustrate the point in a
concrete way.  Other ways of making the comment concrete are:


- Cite a particular Trac ticket that this bit of code deals with
- Cite a test case in the test suite that illustrates it

### Longer comments or architectural commentary



Comments with a broad scope, describing the architecture or workings of more than one module, belong here in the commentary rather than in the code.  Put the URL for the relevant commentary page in a comment in the code itself, and also put URLs for all relevant commentary pages in a comment at the top of each module.


### Commit messages



Please do not use commit messages to describe how something works, or give examples, *even if the patch is devoted to a single change*.  The information is harder to find in a commit message, and (much worse) there is no explicit indication in the code that there is carefully-written information available about that particular line of code.  Instead, you can refer to the Note from the commit message.



Commit messages can nevertheless contain substantial information, but it is usually of a global nature.  E.g. "This patch modifies 20 files to implement a new form of inlining pragma".  
They are also a useful place to say which ticket is fixed by the commit, summarise the changes embodied in the commit etc.  



In short, commit messages describe *changes*, whereas comment explain the code *as it now is*.


## Warnings



We are aiming to make the GHC code warning-free, for all warnings turned on by


```wiki
-Wall
```


The build automatically sets these flags for all source files (see `mk/warnings.mk`).  



The [validate script](testing-patches), which is used to test the build before commiting, additionally sets the `-Werror` flag, so that the code **must** be warning-free to pass validation. The `-Werror` flag is not set during normal builds, so warnings will be printed but won't halt the build.



Currently we are some way from our goal, so some modules have a


```wiki
{-# OPTIONS_GHC -fno-warn-... #-}
```


pragma; you are encouraged to remove this pragma and fix any warnings when working on a module.


## Exports and Imports


### Exports


```wiki
module Foo (
   T(..),
   foo,	     -- :: T -> T
 ) where
```


We usually (99% of the time) include an export list. The only exceptions are perhaps where the export list would list absolutely everything in the module, and even then sometimes we do it anyway. 



It's helpful to give type signatures inside comments in the export list, but hard to keep them consistent, so we don't always do that. 


### Imports



List imports in the following order: 


- Local to this subsystem (or directory) first 
- Compiler imports, generally ordered from specific to generic (ie. modules from utils/ and basicTypes/ usually come last) 
- Library imports 
- Standard Haskell 98 imports last 

  ```wiki
  -- friends
  import SimplMonad

  -- GHC
  import CoreSyn
  import Id
  import BasicTypes

  -- libraries
  import Data.IORef

  -- std
  import Data.List
  import Data.Maybe
  ```


Import library modules from the [boot packages](commentary/libraries) only (boot packages are those packages in the file [packages](/trac/ghc/browser/packages)[](/trac/ghc/export/HEAD/ghc/packages) that have a '-' in the "tag" column). Use `#defines `in `HsVersions.h` when the modules names differ between versions of GHC.  For code inside `#ifdef GHCI`, don't worry about GHC versioning issues, because this code is only ever compiled by the this very version of GHC.



**Do not use explicit import lists**, except to resolve name clashes.  There are several reasons for this:


- They slow down development: almost every change is accompanied by an import list change.

- They cause spurious conflicts between developers.

- They lead to useless warnings about unused imports, and time wasted trying to
  keep the import declarations "minimal".

- GHC's warnings are useful for detecting unnecessary imports: see `-fwarn-unused-imports`.

- TAGS is a good way to find out where an identifier is defined (use `make tags` in `ghc/compiler`,
  and hit `M-.` in emacs).


If the module can be compiled multiple ways (eg. GHCI vs. non-GHCI), make sure the imports are properly `#ifdefed` too, so as to avoid spurious unused import warnings. 


## Compiler versions and language extensions



GHC must be compilable and validate by the previous two major GHC releases, and itself. It isn't necessary for it to be compilable by every intermediate development version. 



To maintain compatibility, use [HsVersions.h](commentary/coding-style#) (see below) where possible, and try to avoid using \#ifdef in the source itself. 


### `HsVersions.h`



`HsVersions.h` is a CPP header file containing a number of macros that help smooth out the differences between compiler versions. It defines, for example, macros for library module names which have moved between versions. Take a look [compiler/HsVersions.h](/trac/ghc/browser/ghc/compiler/HsVersions.h).


```wiki
#include "HsVersions.h"
```

### Literate Haskell



In GHC we use a mixture of literate (`.lhs`) and non-literate (`.hs`) source. I (Simon M.) prefer to use non-literate style, because I think the `\begin{code}..\end{code`} clutter up the source too much, and I like to use Haddock-style comments (we haven't tried processing the whole of GHC with Haddock yet, though). 


### The C Preprocessor (CPP)



Whenever possible we try to avoid using CPP, as it can hide code from the compiler (which means changes that work on one platform can break the build on another) and code using CPP can be harder to understand.



The following CPP symbols are used throughout the compiler: 


<table><tr><th>**DEBUG**</th>
<td>
Used to enables extra checks and debugging output in the compiler. The ASSERT macro (see `HsVersions.h`) provides assertions which disappear when DEBUG is not defined. 
</td></tr></table>


>
>
> However, whenever possible, it is better to use `debugIsOn` from the `Util` module, which is defined to be `True` when `DEBUG` is defined and `False` otherwise.  The ideal way to provide debugging output is to use a Haskell expression "`when debugIsOn $ ...`" to arrange that the compiler will be silent when `DEBUG` is off (unless of course something goes wrong or the verbosity level is nonzero). When option `-O` is used, GHC will easily sweep away the unreachable code.
>
>

>
>
> As a last resort, debugging code can be placed inside `#ifdef DEBUG`, but since this strategy guarantees that only a fraction of the code is seen be the compiler on any one compilation, it is to be avoided when possible.
>
>

>
>
> Regarding performance, a good rule of thumb is that `DEBUG` shouldn't add more than about 10-20% to the compilation time. This is the case at the moment. If it gets too expensive, we won't use it. For more expensive runtime checks, consider adding a flag - see for example `-dcore-lint`.
>
>


**Trap, pitfall for using the ASSERT macro**:



The ASSERT macro uses CPP, and if you are unwise enough to try to write assertions using primed variables (`ASSERT (not $ intersectsBlockEnv b b')`), one possible outcome is that CPP silently fails to expand the ASSERT, and you get this very baffling error message:


```wiki
Not in scope: data constructor `ASSERT'
```


Now you can Google for this error message :-)


<table><tr><th>**GHCI**</th>
<td>
Enables GHCi support, including the byte code generator and interactive user interface. This isn't the default, because the compiler needs to be bootstrapped with itself in order for GHCi to work properly. The reason is that the byte-code compiler and linker are quite closely tied to the runtime system, so it is essential that GHCi is linked with the most up-to-date RTS. Another reason is that the representation of certain datatypes must be consistent between GHCi and its libraries, and if these were inconsistent then disaster could follow. 
</td></tr></table>


### Platform tests



Please refer to [Platforms and Conventions](commentary/platform-naming) wiki page for an overview of how to handle target specific code in GHC.


## Tabs vs Spaces



GHCs source code is indented with a mixture of tabs and spaces, and is standardised on a tabstop of 8.



The Haskell source code in GHC is free of tabs now. Keep it this way. This is actually enforced by a git hook. Other, non-Haskell-code is indented with a mixture of tabs and spaces, and is standardised on a tabstop of 8. The git commit will prevent you from adding tabs to files that currently do not contain tabs.



In order to avoid angering this git hook, you should set your editor to indent using spaces rather than tabs:


- In Emacs, add `(setq-default indent-tabs-mode nil)` to your `.emacs` file ([
  more discussion](http://cscs.umich.edu/~rlr/Misc/emacs_tabs.htm))
- In Sublime Text, save the following to files at `Packages/User/Haskell.sublime-settings` and `Packages/User/Literate Haskell.sublime-settings`:

```wiki
{
	"tab_size": 8,
	"translate_tabs_to_spaces": true
}
```

- In TextMate, in the tabs pop-up menu at the bottom of the window, select "Soft Tabs", as show in the following screenshot where the blue rectangle is:

>
>
> [](/trac/ghc/attachment/wiki/Commentary/CodingStyle/TextMate-tabs-menu.png)  
>
>

>
>
> Alternatively, open the Bundle Editor and add a new Preference called Indentation to the bundle editor. Give it the following contents:
>
>

```wiki
{	shellVariables = (
		{	name = 'TM_SOFT_TABS';
			value = 'YES';
		},
	);
}
```