# Windows platforms: Cygwin, MSYS, and MinGW



Warning. This page, although still useful, is somewhat out of date. There have been multiple forks in this space. We currently use [
MSYS2](http://sourceforge.net/projects/msys2/) instead of [
MSYS(1)](http://mingw.org/wiki/MSYS) and [
MinGW-W64](http://mingw-w64.org/) instead of the original [
MinGW](http://mingw.org/). And Cygwin is no longer supported for building GHC.



The build system is built around UNIX Makefiles. Because it's not native, the Windows situation for building GHC is a little confusing. This section tries to outline the situation for building GHC on Windows.



If you are already familiar with using UNIX build tools on Windows and simply want to build GHC the jump to [
build preparation](http://hackage.haskell.org/trac/ghc/wiki/Building/Preparation/Windows).


## MinGW



[
MinGW (Minimalist GNU for Windows)](http://www.mingw.org) is a native port of `GCC` to windows such that it can be used to compile and produce native windows programs that have no reliance on third party DLLs. The tools distributed include the GNU Compiler Collection (`GCC`), GNU Binary Utilities (`Binutils`), GNU Debuger (`GDB`) and some other utilities.



MinGW doesn't provide a UNIX shell environment though so by itself can't be used to build GHC as we rely on GNU Make, a UNIX Shell and the standard UNIX utilities.


## Cygwin and MSYS



As you can't use MinGW to "build" GHC since it doesn't provide a shell or GNU Make, another additional project will be needed. For that there are two choices:


- [
  MSYS](http://www.mingw.org/wiki/MSYS) MSYS provides a bare bones UNIX like environment, including GNU Bash, GNU Make, GNU Autoconf, SSH and GNU Coretuils. Its enough so that in combination with MinGW you can build native windows tools using the GNU build chain and utilities. MSYS is managed by the MinGW project.

- [
  Cygwin](http://www.cygwin.com) aims to provide a POSIX API and environment on windows. The price is that executables built against Cygwin must be dynamically linked with the Cygwin DLL. It is also a little more of an abstracted (from Windows) environment to work in. Cygwin does provide far more of a UNIX like environment than MSYS does though, even offering X Windows for example.

## GHC Targets MinGW



We want GHC to compile programs that work on any Win32 system.  Hence:


- GHC does invoke a C compiler, assembler, linker and so on, but we ensure that it only invokes the MinGW tools, not the Cygwin ones. That means that the programs GHC compiles will work on any system, but it also means that the programs GHC compiles do not have access to all of Posix.  In particular, they cannot import the (Haskell) POSIX library; they have to do their input output using standard Haskell I/O libraries, or native Win32 bindings. We will call a GHC that targets MinGW in this way *GHC-mingw*.

- To make the GHC distribution self-contained, the GHC distribution includes the MinGW `gcc`, `as`, `ld`, and a bunch of input/output libraries.  


So *GHC targets MinGW*, not Cygwin. It is in principle possible to build a version of GHC, *GHC-cygwin*,  that targets Cygwin instead.  The up-side of GHC-cygwin is that Haskell programs compiled by GHC-cygwin can import the (Haskell) Posix library. *We do not support GHC-cygwin, however; it is beyond our resources.*



While GHC *targets* MinGW, that says nothing about how GHC is *built*.  We use both MSYS and Cygwin as build environments for GHC; both work fine, though MSYS is rather lighter weight.



In your build tree, the compiler you build uses the `gcc` that you specify using the `--with-gcc` flag when you run `configure` (see below). The makefiles are careful to use the right gcc, either via the in-place ghc or directly, to compile any C files, so that we use correct `gcc` rather than whatever one happens to be in your path.  However, the makefiles do use whatever `ld` and `ar` happen to be in your path. This is a bit naughty, but (a) they are only used to pull together .o files into a bigger .o file, or a .a file, so they don't ever get libraries (which would be bogus; they might be the wrong libraries), and (b) Cygwin and MinGW use the same .o file format.  So its ok.


## File names



Cygwin, MSYS, and the underlying Windows file system all understand file paths of form `c:/tmp/foo`. However:
 


- MSYS programs understand `/bin`, `/usr/bin`, and map Windows's lettered drives as `/c/tmp/foo` etc.  The exact mount table is given in the doc subdirectory of the MSYS distribution. When it invokes a command, the MSYS shell sees whether the invoked binary lives in the MSYS `/bin` directory.  If so, it just invokes it.  If not, it assumes the program is no an MSYS program, and walks over the command-line arguments changing MSYS paths into native-compatible paths. It does this inside sub-arguments and inside quotes. For example, if you invoke

  ```wiki
  foogle -B/c/tmp/baz
  ```

  the MSYS shell will actually call `foogle` with argument `-Bc:/tmp/baz`.

- Cygwin programs have a more complicated mount table, and map the lettered drives as `/cygdrive/c/tmp/foo`. The Cygwin shell does no argument processing when invoking non-Cygwin programs.
