# To discuss


- How to fix [\#11719](http://gitlabghc.nibbler/ghc/ghc/issues/11719). We can't ever infer a type variable to have a higher-rank kind (as would seem necessary in this example). But perhaps we should type-check type patterns in a different manner than ordinary types, just like `tcPat` is distinct from `tcExpr`. Then, we could use bidirectional type-checking to get things to work out. This is a pretty significant refactor, though. Is it worth it? Or do we just wait until we have dependent types?

# Summary of tasks to be completed



... as discussed by Richard and Simon. This page is mostly for our own notes, but others are welcome to read it.



Oct 18:


- New module `KcTyClsDecls` that pulls from `TcTyClsDecls` and `TcHsType`.
- Remove `get_class_tvs` from `kcTyClGroup`, in favor of tracking AT TcTyCons in the class TcTyCon
- Remove all the work in `tcTyClTyVars`. It's redundant. `correct_binders` is done by `mk_req_tcb` in `generalise`, `reportFloatingKvs` is irrelevant for CUSKS (should move to `generalise` or `checkValidTyCon`), and `findDupTyVarTvs` can move to `generalise`.


Aug 18:


- [\#15742](http://gitlabghc.nibbler/ghc/ghc/issues/15742): nail down inferred/specified/required
- [\#15497](http://gitlabghc.nibbler/ghc/ghc/issues/15497): coercion quantification and [
  Phab:D5054](https://phabricator.haskell.org/D5054)
- [\#15621](http://gitlabghc.nibbler/ghc/ghc/issues/15621), [\#14185](http://gitlabghc.nibbler/ghc/ghc/issues/14185): using implication constraints to improve error messages
- [\#15579](http://gitlabghc.nibbler/ghc/ghc/issues/15579): `topNormaliseType`; also [\#14729](http://gitlabghc.nibbler/ghc/ghc/issues/14729)
- `zonkPromote`: the remaining ones are there for a reason; but Simon still unhappy; see RAE/SLPJ Slack channel 31 Aug; and [\#15588](http://gitlabghc.nibbler/ghc/ghc/issues/15588) and [\#15589](http://gitlabghc.nibbler/ghc/ghc/issues/15589).  Stuff about "fully-known type variables".
- [\#15479](http://gitlabghc.nibbler/ghc/ghc/issues/15479): refactoring `tcHsType`
- [\#15577](http://gitlabghc.nibbler/ghc/ghc/issues/15577): surprising coercions
- [\#14887](http://gitlabghc.nibbler/ghc/ghc/issues/14887): telescopes
- [\#15474](http://gitlabghc.nibbler/ghc/ghc/issues/15474): (small) `Any` etc.


July 18:


- [\#14164](http://gitlabghc.nibbler/ghc/ghc/issues/14164): comments, invariant, asserts (Richard)
- [\#14873](http://gitlabghc.nibbler/ghc/ghc/issues/14873): make `typeKind` monadic in the type checker
- [\#15346](http://gitlabghc.nibbler/ghc/ghc/issues/15346): `pushCoDataCon` is wrong somehow
- [\#15192](http://gitlabghc.nibbler/ghc/ghc/issues/15192): `GRefl`: still looking into perf changes
- [\#8095](http://gitlabghc.nibbler/ghc/ghc/issues/8095): coercion zapping
- Better floatEqualites based on level numbers


**Type declarations and signatures** (Dec 17)


- (May 18).  Simon is very suspicious about `zonkPromote` and friends.  Revisit this once we get rid of `decideKindGeneralisationPlan`.  [\#15141](http://gitlabghc.nibbler/ghc/ghc/issues/15141) is a good starting point.

- (May 18) [\#14040](http://gitlabghc.nibbler/ghc/ghc/issues/14040), which I think is not fixed.  But it’s somehow linked to [\#15076](http://gitlabghc.nibbler/ghc/ghc/issues/15076).  And that in turn is caused by [\#14880](http://gitlabghc.nibbler/ghc/ghc/issues/14880).  Which Richard has a patch for that doesn’t quite work yet.  And the fix might cure [\#14887](http://gitlabghc.nibbler/ghc/ghc/issues/14887)

- (May 18): [\#15122](http://gitlabghc.nibbler/ghc/ghc/issues/15122)

- (May 18): [\#15142](http://gitlabghc.nibbler/ghc/ghc/issues/15142): `getInitialKinds` simplification

- Let's fix [\#14880](http://gitlabghc.nibbler/ghc/ghc/issues/14880)!

  - `uo_thing`, [\#9173](http://gitlabghc.nibbler/ghc/ghc/issues/9173), [\#13601](http://gitlabghc.nibbler/ghc/ghc/issues/13601)

- [
  Proposal 83: collapse PolyKinds and TypeInType](https://github.com/ghc-proposals/ghc-proposals/pull/83)
- [
  Proposal 81: Visible dependent quantification](https://github.com/ghc-proposals/ghc-proposals/pull/81)
- [
  Proposal 54: CUSKs (depends on Proposal 81)](https://github.com/ghc-proposals/ghc-proposals/pull/54)
- [
  Proposal 103: treat kind and type variables identically in forall](https://github.com/ghc-proposals/ghc-proposals/pull/103) (depends on Proposal 83)
- Add curly-braces for scoping but no type application (nothing depends on this)
- Scoping only with explicit foralls (perhaps best after Proposal 83, because explicit kind-forall needs TypeInType).  Subsumes [\#14548](http://gitlabghc.nibbler/ghc/ghc/issues/14548).
- Fix [\#14451](http://gitlabghc.nibbler/ghc/ghc/issues/14451) (SCC analysis of type declarations)
- All or nothing rule applies to kind variables too (edited)


**Big things**


- Homogeneous flattener ([\#12919](http://gitlabghc.nibbler/ghc/ghc/issues/12919), [\#13643](http://gitlabghc.nibbler/ghc/ghc/issues/13643)).  [
  Phab:D3848](https://phabricator.haskell.org/D3848).   [
  Phab:D4451](https://phabricator.haskell.org/D4451) is a patch to D3848 that fixes performance
- [
  Phab:4049](https://phabricator.haskell.org/4049) is the bump-TcLevel-in-type-sigs patch (Trac [\#14066](http://gitlabghc.nibbler/ghc/ghc/issues/14066)), which is based on D3848.
- [\#11715](http://gitlabghc.nibbler/ghc/ghc/issues/11715): constraint vs \*

- [\#12088](http://gitlabghc.nibbler/ghc/ghc/issues/12088): scc for kind inference (lower priority)


**Little things**


- BUGS

# Fuller list


- Deliver on [\#13959](http://gitlabghc.nibbler/ghc/ghc/issues/13959) (`substTyVar` etc)
- Change flattener to be homogeneous ([\#12919](http://gitlabghc.nibbler/ghc/ghc/issues/12919), [\#13643](http://gitlabghc.nibbler/ghc/ghc/issues/13643))
- Sort out `mkCastTy`

  - Implement KPush in `splitTyConApp`. ([\#13650](http://gitlabghc.nibbler/ghc/ghc/issues/13650))
  - Some invariants to make sure of: No nested `CastTy`s. No `AppTy (TyConApp ... |> co) ty`. No reflexive coercions.
  - Change the premises to `LRCo` so that there may be an outer coercion. That is:

    ```wiki
    g : (t1 t2 |> co1) ~ (s1 s2 |> co2)
    -----------------------------------
    Left g : t1 ~ s1
    ```

>
> >
> >
> > There is more work to do to make this homogeneous.
> >
> >
>

- Implement homogeneous as per Stephanie's paper

  - An-Refl2 makes me think that the output of `coercionKind` would be hetero. Indeed it would. But we could still have `(~#) :: forall k. k -> k -> Type` because we don't have to abstract over hetero equalities. Note that Wanteds are CoercionHoles, and that we can always homogenize givens. This would also require storing `PredTree`s in `CtEvidence` instead of `PredType`s (because we can't write the type of a hetero coercion.
- Fix [\#11715](http://gitlabghc.nibbler/ghc/ghc/issues/11715) according to Richard's plan
- Generalized injectivity [\#10832](http://gitlabghc.nibbler/ghc/ghc/issues/10832), vis-a-vis Constrained Type Families paper
- Taking better advantage of levity polymorphism:

  - Could `[]` be a data family?
  - Unlifted newtypes
  - Unlifted datatypes
  - generalized classes in base
  - ...
- [\#11739](http://gitlabghc.nibbler/ghc/ghc/issues/11739) (simplify axioms)

  - Also: consider making a closed-type-family axiom into a bunch of top-level axioms using some proof of apartness. It might simplify the step in coercion optimization where we optimize a c.t.f. coercion only to abandon the changes because they break apartness constraints. It would also allow us to delete gobs of code dealing with "branched axioms" vs regular ones.
- Fix all the `TypeInType` bugs
- Clean up pure unifier to make the fact that kind coercions *only* affect type variables by using, e.g., `getCastedTyVar_maybe`.
- It seems that `quantifyTyVars` is duplicating some logic from `simplifyInfer`, in that it removes covars. This should really be done in `kindGeneralize`, because `simplifyInfer` *uses* `quantifyTyVars`. This should be just about possible, but with some twists and turns:

  - H98 constructors are strange in that they have tyvars that aren't mentioned in the type. So be careful here and make sure the type is closed (w.r.t. user-written tyvars) before calling `kindGeneralize`.
  - `tcFamTyPats` needs a hard look
  - So does `tcRule`.
  - Could also separate out `kindGeneralizeKind` and `kindGeneralizeType`. The latter works only over closed types.
  - If we remove the "remove covars" call from `quantifyTyVars`, we should really put it in `decideQuantifiedTyVars`. Perhaps we *don't* need to remove covars in `kindGeneralize` because `solveEqualities` will fail if any covars are around. It is **OK** to remove a covar without removing its kind, because the covar will be solved in the residual implication constraint from `simplifyInfer`.
  - Example of why we need to exclude coercions during generalization:

```wiki
data X where
  MkX :: Proxy a -> Proxy b -> (Refl :: a :~: b) -> X
```

- Remove `quantifyTyVars` call from `simplifyInfer`. Instead call `skolemiseUnboundMetaTyVars` from `simplifyInfer` directly.
- Stable topological sort may not be well specified. But we can always write a deterministic algorithm. Perhaps that should be in the manual.
- Can remove `closeOverKinds` in most places. Otherwise, just gather the kinds of user-written tyvars (e.g. fundep RHS)
- Re-do the fix for [\#13233](http://gitlabghc.nibbler/ghc/ghc/issues/13233). There are two separate problems:

  1. How to ascertain whether or not a primop is saturated during desugaring (or, possibly, earlier). On a call, we thought that we could do this in the desugarer by decomposing nested `HsApp`s, using a little stack data type to denote all the different ways a function could be applied (`HsApp`, `HsWrap` with the right wrapper, sections, tuple-sections, `HsTypeApp`, maybe more) uncovering what the function was underneath, and then checking how many parameters are supplied. But now, I think it's much better to do this in the type-checker, especially because the type-checker already decomposes nested `HsApp`s. (See `TcExpr.tcApp`.) When it discovers what the function is, it can check whether the function is a `hasNoBinding` primop. If so, it can eta-expand as necessary (but only if necessary) and use a new piece of `HsSyn` to denote a saturated primop. (It will be a new invariant that no unsaturated primop passes the type-checker.) This seems better than redoing the stack type in the desugarer. The original problem in [\#13233](http://gitlabghc.nibbler/ghc/ghc/issues/13233) was around levity polymorphism. If we make this change in the type checker, then the existing levity polymorphism checks should just work. We'll have to be careful to make the `HsSyn` structure printable in the way a user expects, so that the levity-polymorphism error message doesn't talk about an argument the user didn't write.
  1. How to make sure that saturated primops stay that way in Core. This would be a new check in Lint as well as new checks on any code that does eta-contraction. It has been suggested that levity-polymorphic primops desugar to a family of levity-monomorphic primops. This surely would work, but there doesn't seem to be benefit over a plan simply to keep primops eta-expanded always. Then, there's no worry about eta-contracting levity-polymorphic arguments.
- Make better use of the `uo_thing` field, including refactoring `noThing` away and improving term-level error messages.

  - Simon also asks that the contents of `uo_thing` should only be `HsSyn`. This would obviate the current zonking/tidying stuff. A quick pass suggests that this will be easy to do.
  - Also see [\#13819](http://gitlabghc.nibbler/ghc/ghc/issues/13819), where the current treatment of `uo_thing` leads to an outright bug.
- Take full advantage of `TcTyCon`, getting rid of the dreaded type-checking knot. ([\#13737](http://gitlabghc.nibbler/ghc/ghc/issues/13737))
- Document why we're not worried about casts in class wanteds. (Short story: any cast should be available for rewriting, and so it will rewrite the kinds.)
- Sort out `matchTypeable` (see email) [\#13333](http://gitlabghc.nibbler/ghc/ghc/issues/13333)
- Fix equality printing: 

  - Remove IfaceEqualityTyCon in favor of a new IfaceEquality constructor of IfaceType, which would be the conversion of a TyConApp
  - Make explicit-kinds print the kinds (duh) and equality-rels control the equality relation (duh)
  - Print \~ for homo in practice; print `~~` for hetero in practice (unless equality-rels)
- Merge fsk and fmv treatment, by returning the list of created fsks from `solveSimpleGivens` (which would now be fmvs) and fill them in after solving the wanteds. This eliminates problems around the fact that zonking in the flattener might zonk fsks back to type family applications and that fsks might lurk in residual constraints.
- Remove `wc_insols`. The distinction isn't paying its way.
- It's terrible if we ever inspect a meta-tyvar in pure code. Something is surely awry. Add an `ASSERT` to `coreView` to stop this from happening and fix any consequences.

## Completed tasks


- Remove pushing from `mkCastTy`. But see bullet above about remaining tasks.
- Remove `solveSomeEqualities`
- Take a look at `tidyToIfaceType`: I don't think it needs to tidy the env.
