# Installing packages in your test compiler



You have your compiler built, so you can use the inplace compiler to compile test programs.  This lives in `$(TOP)/inplace/bin/ghc-stage2` (see [Commentary/SourceTree](commentary/source-tree)).  



Every GHC, including the inplace one, comes with the [boot libraries](commentary/libraries).  But sometimes a test case will require the installation of some packages from Hackage.  There are two routes.


## Plan A: use [
cabal install](http://hackage.haskell.org/trac/hackage/wiki/CabalInstall)



This method is quick and easy, but can fail if your `cabal` program is out of date with respect to the GHC version you are building.  Here's how to install a library against a GHC build tree:


```wiki
cabal install --with-compiler=<inplace-ghc> --package-db=<inplace-package-db> <package>
```


where `<inplace-ghc>` is the path to your inplace GHC (`$(TOP)/inplace/bin/ghc-stage2`; make sure it's absolute), `<inplace-package-db>` is the path to your inplace package database (`$(TOP)/inplace/lib/package.conf.d`; again, make sure it's absolute) and \<package\> is the name of the package.



Points to note:


- The testsuite driver doesn't search the user package database for extra packages (it calls 'ghc-pkg --no-user-package-db describe \<package\>), therefore we install the package in the inplace package database.


Plan A can fail, because sometimes GHC changes require corresponding Cabal changes (this happened in GHC 6.12, and after the GHC 7.6.1 release). If the format changed you might get a message like


```wiki
cabal: failed to parse output of 'ghc-pkg dump'
```


But incompatibility can result in other problems installing packages. In that case you need to use the Cabal code that comes with the new version of GHC (ie the one in your build tree).  So use Plan B.


## Plan B: use the Cabal library bundled with GHC



This method is slightly more work, but might work in case plan A failed.



Go to a directory where you are happy to keep the newly-downloaded code.


```wiki
cabal unpack <package>
cd <package>
<inplace-ghc> --make Setup.lhs
./Setup configure --with-compiler=<inplace-ghc> --global
./Setup build
./Setup register --inplace
```


Points to note here:


- The first step can be done manually, by downloading a zip file from Hackage, or by doing grabbing the source from the appropriate repository.  For example:

  ```wiki
  darcs get http://darcs.haskell.org/packages/parallel
  ```

  Nevertheless, `cabal unpack` should work for any Hackage package, even if Cabal has changed a bit.  (Because fetching and unpacking is one of Cabal's less sophisticated operations.)

- It is important to compile `Setup.lhs` with your shiny new *inplace* GHC, not your installed GHC.  Your inplace GHC has the most up-to-date Cabal library, and that is what you want to link `Setup.lhs` against.

- The `--global` flag instructs Cabal to register the package in the database in your build tree, rather than the one in your home directory (`~/.ghc/...`).  In fact, `--global` is actually unnecessary (it's the default), but just in case the default changes in the future it's a good idea to get in the habit of saying whether you want `--global` or `--user`.


 


- The `--inplace` flag to register tells Cabal not to copy the compiled package, but rather to leave it right where it is, and register this location in the package database in your GHC build tree.
