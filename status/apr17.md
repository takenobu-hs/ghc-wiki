# GHC Status Report (April 2017)



By the time of publication GHC will be nearing its 8.2.1
release. While GHC 8.0 was a feature-oriented release, the past year of
GHC development has been focused on stabilization and consolidation of
existing features. This has included an effort to reduce compilation
times, improve code generation, and place the levity polymorphism story
introduced in GHC 8.0 on a stable theoretical footing.



While priorities for the 8.4 release have not yet been discussed, they
will very likely include further focus on compilation time reduction,
improved error messages, and additional improvements to code generation.


## Major changes in GHC 8.2



While the emphasis of 8.2 is on performance, stability, and
consolidation, it also includes a number of new features.


### Libraries, source language, and type system


-   **Indexed Typeable representations.** While GHC has long supported runtime type reflection through the `Typeable` typeclass, its current incarnation requires careful use, providing little in the way of type-safety. For this reason the implementation of types like `Data.Dynamic` must be implemented in terms of `unsafeCoerce` with no compiler verification.

>
>
> GHC 8.2 will address this by introducing indexed type representations, leveraging the type-checker to verify many programs using type reflection. This allows facilities like `Data.Dynamic` to be implemented in a fully type-safe manner. See the [
> paper](https://research.microsoft.com/en-us/um/people/simonpj/papers/haskell-dynamic/)\] for a description of the proposed interface and the [
> Wiki](https://ghc.haskell.org/trac/ghc/wiki/Typeable/BenGamari) for current implementation status.
>
>

-   **Backpack.** Backpack has merged with GHC, Cabal and `cabal-install`, allowing you to write libraries which are parametrized by signatures, letting users decide how to instantiate them at a later point in time. If you want to just play around with the signature language, there is a new major mode `ghc –backpack`; at the Cabal syntax level, there are two new fields `signatures` and `mixins` which permit you to define parametrized packages, and instantiate them in a flexible way. More details are on the Backpack [Wiki page](backpack).

-   **Levity polymorphism.** GHC 8.0 reworked GHC’s kind system introducing the notion of levity to describe a type’s runtime representation. However, the ability to write functions which were polymorphic over levity was purposefully withheld.

>
>
> While GHC’s current compilation model doesn’t allow arbitrary levity polymorphism, GHC 8.2 enables certain classes of polymorphism which were either disallowed or broken in 8.0. See the [
> paper](https://www.microsoft.com/en-us/research/publication/levity-polymorphism/) for details.
>
>

-   **`deriving` strategies.** GHC now provides the programmer with a precise mechanism to distinguish between the three ways to derive typeclass instances: the usual way, the `GeneralizedNewtypeDeriving` way, and the `DeriveAnyClass` way. See the `DerivingStrategies` [
  Wiki page](https://ghc.haskell.org/trac/ghc/wiki/Commentary/Compiler/DerivingStrategies) for more details.

-   **New classes in `base`.** The `Bifoldable`, and `Bitraversable` typeclasses are now included in the `base` library.

-   **Unboxed sums.** GHC 8.2 has a new language extension, `UnboxedSums`, that enables unboxed representation for non-recursive sum types. GHC 8.2 doesn’t use unboxed sums automatically, but the extension comes with new syntax, so users can manually unpack sums. More details can be found in the [
  Wiki page](https://ghc.haskell.org/trac/ghc/wiki/UnpackedSumTypes).

### Runtime system


-   **Compact regions**. This runtime system feature allows a referentially “closed” set of heap objects to be collected into a “compact region,” allowing cheaper garbage collection, sharing of heap-objects between processes, and the possibility of inexpensive serialization. See the [
  paper](http://ezyang.com/papers/ezyang15-cnf.pdf) for details.

-   **Better profiling support.** The cost-center profiler now better integrates with the GHC event-log. Heap profile samples can now be dumped to the event log, allowing heap behavior to be more easily correlated with other program events. Moreover, the cost center stack output (e.g. `.prof` files) can now be produced in a machine-readable JSON format for easier integration with external tooling.

-   **More robust DWARF output.** GHC 8.2 will be the first release with reliable support for DWARF debugging information. A number of bugs leading to incorrect debug information for foreign calls have been fixed, meaning it should now be safe to enable debugging information in production builds.

>
>
> With stable DWARF support comes a number of opportunities for new performance analysis and debugging tools (e.g. statistical profiling, cheap execution stacks). As GHC’s debugging information improves, we expect to see tooling developed to support these applications. See the DWARF status page on the [
> GHC Wiki](https://ghc.haskell.org/trac/ghc/wiki/DWARF/Status) for further information.
>
>

-   **Better support for NUMA platforms.** Machines with non-uniform memory access costs are becoming more and more common as core counts continue to rise. The runtime system is now better equipped to efficiently run on such systems.

-   **Experimental changes to the scheduler** that enable the number of threads used for garbage collection to be lower than the `-N` setting.

-   **Support for `StaticPointers` in GHCi.** At long last programs making use of the `StaticPointers` language extension will have first-class bytecode interpreter support, allowing such programs to be loaded into GHCi.

-   **Reduced CPU usage at idle.** A long-standing regression resulting in unnecessary wake-ups in an otherwise idle program was fixed. This should lower CPU utilization and improve power consumption for some programs.

### Miscellaneous


-   **Compiler Determinism.** GHC 8.0.2 is the first release of GHC which produces deterministic interface files. This helps consumers like Nix and caching build systems, and presents new opportunities for compile-time improvements. See the [
  Wiki](https://ghc.haskell.org/trac/ghc/wiki/DeterministicBuilds) for details.

## Development updates and acknowledgments



Since the 8.0 release nearly a year ago, GHC’s development community has
continued to grow. Over the last year we have seen sixty new developers
submit patches, many of whom have become regular contributors. These
include new generations of researchers, as well as, increasingly,
regular Haskell users from industry and elsewhere. GHC has had an
exciting few months...



Moritz Angermann has been hard at work improving cross-compilation and
bringing remoting support to the `-fexternal-interpreter` feature
introduced in GHC 8.0. He has also contributed a number of patches
improving support for iOS and Android targets and has been pondering how
to enable Template Haskell during cross-compilation.



Ryan Scott and Matthew Pickering have been fixing bugs all over the
compiler, in addition to their respective work on deriving strategies
and the new `COMPLETE` pragma mentioned above.



Tamar Christina has been hard at work as our resident Windows expert. In
addition to introducing split sections support, reworking dynamic
linking, fixing toolchain breakage due to the recent Windows 10
Creator’s Update, and maintaining GHC packaging for the Chocolatey
package manager, he is also an invaluable source of knowledge.



Andrey Mokhov and Jose Calderon have been continuing work on Hadrian,
GHC’s new build system built on Shake. Hadrian is already very usable,
lacking only in support for a some non-standard build configurations,
building documentation, and producing binary and source distributions.
These would be great, fairly self-contained projects for any interested
developer with a Haskell experience. Contact Andrey if any of these
sound intriguing to you.



Michal Terepeta has been performing a long-overdue grooming of GHC’s
`nofib` benchmark suite. This suite is one of the primary windows into
the performance of the compiler, yet has fallen into somewhat of a state
of disrepair. Michal has been fixing broken tests, refactoring the build
system, and generally making `nofib` a nicer place. In addition, he has
been gradually cleaning up native code generator, revisiting GHC’s use
of the `hoopl` library. Also working in the native code generator is
Thomas Jakway, who has been working on improving the register
allocator’s treatment of loops.



Another contributor working on runtime performance is Luke Maurer, who
has contributed a rework of GHC’s treatment of join points, one of the
major features of 8.2. Join points are an essential optimization which
allows GHC to eliminate closure allocation for some types of calls.
Prior to GHC 8.2 the join points optimization was fragile due to the
lack of a sound theoretical footing. With Luke’s work, GHC is much
better at preserving join point opportunities, leading to improved code
generation in many performance critical settings. More details can be
found in his [
paper](https://www.microsoft.com/en-us/research/publication/compiling-without-continuations).



As always, this is just a fraction of the contributions which we’ve seen
in the past months. GHC improves through the efforts of everyone who
offers patches, bug reports, code review, and discussion. If you have
contributed any of these in the past year, thank you!



GHC HQ has also taken a number of steps in the past months to try to
improve contributor workflow. Our [
new proposal process](https://github.com/ghc-proposals/ghc-proposals) officially
began in December and has been extremely active ever since: the
`ghc-proposals` repository now has over fifty pull requests and has
facilitated hundreds of messages of great discussion. Thanks to our
everyone who has participated! GHC also now accepts GitHub pull requests
for small changes to the user guide and library documents, which several
contributors have made use of this. We hope that this channel will make
it easier for users to contribute and perhaps feel compelled to pick up
larger tasks in the future.



As always, if you are interested in contributing to any facet of GHC, be
the runtime system, type-checker, documentation, simplifier, or anything
in between, please come speak to us either on IRC (`#ghc` on
`irc.freeenode.net`) or `ghc-devs@haskell.org`. Happy Haskelling!


## Further reading


-   GHC website:

>
>
> \<[ https://haskell.org/ghc/](https://haskell.org/ghc/)\>
>
>

-   GHC users guide:

>
>
> \<[
> https://downloads.haskell.org/\~ghc/master/users-guide/](https://downloads.haskell.org/~ghc/master/users-guide/)\>
>
>

-   `ghc-devs` mailing list:

>
>
> \<[
> https://mail.haskell.org/mailman/listinfo/ghc-devs](https://mail.haskell.org/mailman/listinfo/ghc-devs)\>
>
>

