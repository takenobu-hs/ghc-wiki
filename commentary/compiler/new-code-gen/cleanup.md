## Historical page



This page is a bunch of notes on the new code generator. It is outdated and is here only for historical reasons.It should probably be removed. See [Code Generator](commentary/compiler/code-gen) page for a description of current code generator.


# Cleanup after the new codegen is enabled



The new codegen was enabled by default in [832077ca5393d298324cb6b0a2cb501e27209768](/trac/ghc/changeset/832077ca5393d298324cb6b0a2cb501e27209768/ghc).  Now that the switch has been made, we can remove all the cruft associated with the old code generator.  There are dependencies between some of the components, so we have to do things in the right order.  Here is a list of the cleanup tasks, and notes about dependencies:


## Independent tasks


- Use `BlockId` or `Label` consistently, currently we use a mixture of the two.  Maybe get rid of the `BlockId` module.

- Remove live-var and CAF lists from `StgSyn`, and then clean up `CoreToStg`

- DONE Remove the SRT pass in `simplStg/SRT.lhs`

- DONE remove RET\_DYN from the RTS

- DONE remove `-fnew-codegen`, related `HscMain` bits and the `CodeGen` module.

- DONE remove `CmmOpt.cmmMiniInline`, it is not used any more

- Fix the layering: `cmm` modules should not depend on `codeGen/StgCmm*`

## Towards removing codeGen/Cg\*


- DONE `CmmParse` should produce new `Cmm`. 

  - We will probably want two kinds of `.cmm` file, one that is to be fed through `CmmLayoutStack` and one that isn't.
  - primops will be fed through `CmmLayoutStack`, and will use the native calling convention, with the code generator inserting the copyin/copyout for us.

- DONE Remove all the `Cg*` modules

## Towards removing `OldCmm`


- IN PROGRESS (Simon M): Change the NCG over to consume new `Cmm`.  We possibly also want the generated native code to use the Hoopl Block representation, although that will mean changing branch instructions to have both true and false targets, rather than true and fallthrough as we have now.

- Remove `cmm/CmmCvt` (this will save some compile-time too)

- Remove `cmm/OldCmm*`, `cmm/PprOldCmm` etc.

## Later


- Do the new SRT story (ToDo: write a wiki page about this)
