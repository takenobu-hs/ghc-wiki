# Setting Up Cygwin for building GHC (unsupported)



We no longer support Cygwin. This page is here only for historical reasons.


1. Install [ Cygwin](http://www.cygwin.com/)

1. You must install enough Cygwin *packages* to support building GHC. This means selecting at least the following packages in the installation dialogue:
  `openssh`,
  `autoconf`,
  `binutils`,
  `gcc`,
  `make`.
  To see these packages, click on the "View" button in the "Select Packages" stage of Cygwin's installation dialogue, until the view says "Full". Note: Don't use the cygwin git (you'll get various failures, e.g. fork failures, with Windows Vista or Windows 7). Use [
  http://code.google.com/p/msysgit/](http://code.google.com/p/msysgit/) instead.

1. Now set the following user environment variables:

  - Add `c:/cygwin/bin` and `c:/cygwin/usr/bin` to your `PATH`
  - Set `MAKE_MODE` to `UNIX`. If you don't do this you get very weird messages when you type `make`, such as:

    ```wiki
    /c: /c: No such file or directory
    ```
  - Set `SHELL` to `c:/cygwin/bin/bash`. When you invoke a shell in Emacs, this `SHELL` is what you get.
  - Set `HOME` to point to your home directory.  This is where, for example, `bash` will look for your `.bashrc` file. Ditto `emacs` looking for `.emacsrc`


Here are some things to be aware of when using Cygwin:


- Cygwin implements a symbolic link as a text file with some magical text in it.  So other programs that don't use Cygwin's I/O libraries won't recognise such files as symlinks. In particular, programs compiled by GHC are meant to be runnable without having Cygwin, so they don't use the Cygwin library, so they don't recognise symlinks.

- Some script files used in the make system start with "`#!/bin/perl`", (and similarly for `sh`).  Notice the hardwired path! So you need to ensure that your `/bin` directory has at least `sh`, `perl`, and `cat` in it. All these come in Cygwin's `bin` directory, which you probably have installed as `c:/cygwin/bin`.  By default Cygwin mounts "`/`" as `c:/cygwin`, so if you just take the defaults it'll all work ok. (You can discover where your Cygwin root directory `/` is by typing `mount`.) Provided `/bin` points to the Cygwin `bin` directory, there's no need to copy anything.  If not, copy these binaries from the `cygwin/bin` directory (after fixing the `sh.exe` stuff mentioned in the previous bullet).
