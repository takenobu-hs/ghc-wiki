# Idiom: stub makefiles



It's all very well having a single giant `Makefile` that knows how to
build everything in the right order, but sometimes you want to build
just part of the system.  When working on GHC itself, we might want to
build just the compiler, for example.  In the ancient, recursive **make**
system we would have done `cd ghc` and then `make`.  In the non-recursive
system we can still achieve this by specifying the target with something
like `make ghc/stage1/build/ghc`, but that's not so convenient.



Our second idiom therefore supports the `cd ghc; make` idiom, just as
with recursive make. To achieve this we put tiny stub `Makefile` in each
directory whose job it is to invoke the main `Makefile` specifying the
appropriate target(s) for that directory.  These stub `Makefiles`
follow a simple pattern:


```wiki
dir = libraries/base
TOP = ../..
include $(TOP)/mk/sub-makefile.mk
```


where `mk/sub-makefile.mk` knows how to recursively invoke the giant top-level **make**.


