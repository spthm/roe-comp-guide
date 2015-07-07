Installing (and compiling) C/C++ libraries and applications from source
=======================================================================

Notation etc.
-------------

Basic command-line knowledge is assumed, although even if you are very
unfamiliar with the terminal, you should still be able to follow along.

Code blocks (i.e. things which the terminal prints out, or you should be typing
into the terminal), `look like this`, or,

    look like this first line
    and this second line

Lastly, if you see `<sometext>`, this is example or placeholder text, and
should not be copied verbatim!
For example, if your username is spth, and you see the text
`cd /home/<username>`, then you should instead type `cd /home/spth`!


Prerequisites
-------------

Most importantly, the below assumes you are working on a provided Linux
workstation and that you have gcc, the
[GNU Compiler Collection](https://gcc.gnu.org/), installed.
The first of these should imply the second, but to be sure, open up a terminal
and type

```bash
user@machine:~$ gcc --version
gcc (Debian 4.7.2-5) 4.7.2
Copyright (C) 2012 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

user@machine:~$ g++ --version
g++ (Debian 4.7.2-5) 4.7.2
Copyright (C) 2012 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

You should see something _similar_ to the above, but with different
version numbers.
If you do not, please submit a helpdesk ticket.
(In general, you probably want _both_ gcc and g++ installed, which are the C
and C++ compilers, respectively.)

It is also assumed that you are using the bash shell.
As of July 2015, this remains the default for new users.
However, tsch has been the default in the past, and some commands will need to
be altered if that is the case for you.


Initial Set-Up
--------------

You will want somewhere specific to put all of your compiled library files,
headers, and potentially the documentation.
A sensible choice is for them to be accessible via `/home/<username>/local/`;
however, your home directory quota is likely to be rather slim, so actually
placing them at this location is not advised.
Instead, make a directory `local` (or whatever you want to call it) on e.g.
`/disk1` of your machine, which should have plenty of space:

```bash
cd /disk1/<username>
mkdir local
```

This is the location we will _actually_ be installing everything into.
To make it conveniently accessible from your home directory, we're going to
create a symbolic link (symlink) to it,

```bash
cd ~/
ln -s /disk1/<username>/local local
ls -l
```

That last command will list everything in your home directory; look for the
line similar to

```bash
lrwxrwxrwx 1 <username> users   16 Jan 01 12:00 local -> /disk1/<username>/local/
```

which shows that `/home/<username>/local` just points us through to the 'real'
folder, living on `/disk1/<username>/`.

From this point on, it is assumed that the above symlink exists.
If you chose not to, __please modify all file paths accordingly__.

It is also recommended to keep all the source code you compile in one place,
e.g.

```bash
cd /disk1/<username>
mkdir sources
cd ~/
ln -s /disk1/<username>/sources sources
```

where we have, as before, actually created the directory on `/disk1` and simply
symlinked this in our home folder.

Finally, with this directory in place, we need to make sure that the compiler
and linker will actually be able to find things we install here!
For this task, we set some environment variables in our `~/.profile` file.
(As of July 2015, `~/.profile` is used by default; on other systems, it is
typical to instead use `~/.bash_profile`. To verify which of these exist on
your machine, run `ls -a ~/`.)

TODO: A note about .[bash_]profile vs .bashrc, and why --- on Linux --- we
probably want to use .[bash_]profile for this.
NOTE: On Ubuntu, `.profile` is now ignored, and in general, `.profile` is
ignored if either `.bash_profile` or `.bash_login` exist!

Any text editor is fine.
For emacs, run `emacs ~/.profile`, and add the following to the __bottom__ of
the file.
Lines beginning with `#` are comments and not necessary, though if you are not
familiar with writing bash it would be wise to keep them.
Some of these environment variables are
[specific to gcc](https://gcc.gnu.org/onlinedocs/gcc/Environment-Variables.html)
or to
[gnu make](https://www.gnu.org/software/make/manual/html_node/Implicit-Variables.html)
and are noted as such.

```bash
# Set environment variables so that manually installed executables, libraries,
# header files and documentation will be found.

# Executables
if [ -d "$HOME/local/bin" ] ; then
    export PATH="$HOME/local/bin:$PATH"
fi

# Libraries (this might still fail; see comments below).
if [ -d "$HOME/local/lib" ] ; then
    # For link-time. Specific to gcc.
    # Equivalent to compiling with -L $HOME/local/lib
    export LIBRARY_PATH="$HOME/local/lib:$LIBRARY_PATH"

    # For runtime linking to shared library files, i.e. lib*.so files.
    # Equivalent to linking with -R $HOME/local/lib, but fragile: if any
    # -R flags are set THIS WILL BE IGNORED!
    # NOTE: If this path ever changes, all code relying on shared library
    # files will need to be recompiled (actually just have its rpath changed)
    export LD_RUN_PATH="$HOME/local/lib:$LD_RUN_PATH"

    # Account for Autoconf builds where -R is set, overriding LD_RUN_PATH.
    # Specific to GNU make.
    # We're throwing in the link-time search path for good measure.
    export LDFLAGS="-L $HOME/local/lib -R $HOME/local/lib:$LDFLAGS"
fi

# Headers.
if [ -d "$HOME/local/include" ] ; then
    # For compile time. Specific to gcc.
    # Equivalent to -I$HOME/local/include
    export CPATH="$HOME/local/include:$CPATH"
fi

# man and info documentation (likely places, but not exhaustive).
if [ -d "$HOME/local/man" ] ; then
    MANPATH="$HOME/local/man:$MANPATH"
fi

if [ -d "$HOME/local/share/man" ] ; then
    MANPATH="$HOME/local/share/man:$MANPATH"
fi

if [ -d "$HOME/local/info" ] ; then
    INFOPATH="$HOME/local/info:$INFOPATH"
fi

if [ -d "$HOME/local/share/info" ] ; then
    INFOPATH="$HOME/local/share/info:$INFOPATH"
fi
export MANPATH
export INFOPATH
```

For these settings to take effect, either open a new terminal window, or type

```bash
. ~/.profile
```

in your current window.
If you see any errors appear after this, go back and check your did not make
any typos in the above.

A few notes on the above (which may make more sense after you've read
[the aside](#aside-static-vs-shared-libraries-link-time-vs-run-time) on static vs. shared libraries, and runtime linking.
First of all, none of this is ideal!
If you were installing on a system to which you had admin access (and could
therefore run `sudo`), you'd use the package manager, and it would install
all library files and headers to locations which are searched _by default_.
But we're supposing that is impossible.
If your package manager didn't have said package available, but you still had
admin rights, then there are
[still better ways](http://choorucode.com/2014/01/14/how-to-add-library-directory-to-ldconfig-cache/)to set your _shared_ library search paths, using
[ldconfig](http://linux.die.net/man/8/ldconfig).
And you could install to a default search path.

The above should work for installs using autoconf, which have a `./configure`
to be run before you actually do any compiling or installing.
However, if autoconf is not used __and__ the Makefile includes some rpaths,
then the above will probably fail.
rpaths look like either of the following:

```bash
-R some/directory/path
-Wl,-rpath=some/directory/path
````

You program will compile fine, but when you run it, it will be unable to find
any of the libraries you have manually installed (i.e. installed by following
this guide).

The simplest way around this is to add the following to an appropriate location
in the Makefile

```make
-Wl,-rpath=$HOME/local/lib
```

where 'appropriate' depends on the Makefile in question, but in general
anywhere you see `-L` flags is probably okay.
For a more in-depth discussion, see e.g.
[here](http://www.eyrie.org/~eagle/notes/rpath.html)

Setting `LD_LIBRARY_PATH` is another solution, but considered
[bad practice](http://xahlee.info/UnixResource_dir/_/ldpath.html)

### Aside: Static vs shared libraries, link-time vs run-time.

The point of a library is that it is reusable.
Say you have some code which implements some functions you'd end up using all
the time; for example, computing Fourier transform of an input array (don't
actually do this yourself, [a library already exists](www.fftw.org) for that).
Rather than rewriting this code in every application you need it, or - only
slightly less worse - copy-and-pasting it into every bit of source code, you
write it _once_, compile that code into a library, and simply 'link' against
this library whenever you need those functions.

With a __static library__, sometimes __statically linked library__, all
compiled code from the library that your application uses is copied directly
into your application's executable file by the 'linker'.
This is much better than you copy-and-pasting source code, because the linker
can't make typos!
But, it still means your executable file size could be quite large: it contains
a copy of all the library functions it uses.
If you have lots of similar executables, all using the same library functions,
this adds up.
(Note that, in terms of astronomy software, this is still likely to be utterly
insignificant when compared to your data.)
On Linux, these files have the __extension `.a`__, e.g. `libfft.a`.
(For the pedantic: `.a` files are technically _archives_, not libraries.)

Statically-compiled programs are therefore more portable: they already contain
all the code they need to run.
Provided you compiled your file for the correct system (e.g. Windows, Linux)
and architecture (e.g. x86, ARM), your binary executable should run anywhere.
Statically-compiled programs also tend to run slightly faster.

To solve the (binary) code bloat problem, we can use __shared libraries__,
sometimes called __dynamic libraries__.
Here, the library code is _not_ copied into our final executable, only
identifiers for the libraries we rely upon, and identifiers for functions we
want to call from them.
When we run the executable, the runtime linker first checks that it can find
all the libraries we need.
It then loads the executable _and_ all the shared libraries it requires into
memory.
Thus, when calling a function in a shared library, code from the library file
itself is run, and the result is passed back to code in our executable.
(Hence why shared library programs tend to run slower: there's some overhead.)
On Linux, these files have the __extension `.so`__ ('shared object'), often
`.so.x.x`, where the `x.x` is some version number, e.g. `libfft.so`.

Shared libraries have several other benefits, including:
- Bug fixes in the library often only result in a new `.so` file, which will be
entirely compatible with your application; you do not need to recompile!
- If lots of different processes request the same shared library, that shared
library need only be loaded into memory once, saving RAM.

The discussion of `rpath` in the previous section relates to _where_ the
runtime linker will look for shared libraries.
In the software industry, development systems are often very different from
consumer systems, and it therefore became necessary to separately specify where
to look for a library at compile time, vs. where to look for it when an
application is run.
For example, my `libfft.so` might be in `/opt/dev/fft/libs/libfft.so`, but I
know that all users will have it in a standard location, `/usr/local/lib/libfft.so`.
`-L` flags tell the compiler and linker where to look at compile time.
In the proposed example, this is `/opt/dev...`.
For static libraries, this is entirely sufficient.
-`-R` or `-Wl,-rpath=` flags write code _into your application_ that tells the
runtime linker where you look whenever the application is launched.
In the proposed example, this is `/usr/local...`.
The runtime linker also has a list of default locations it will look at.
Without admin privileges, you are not able to alter this list, nor install
libraries to one of these locations.
We must therefore resort to adding the location of our manually-installed
libraries into _every executable_ we compile which requires them.

Setting `LD_RUN_PATH` typically handles this for us, but it is _ignored_ if
the compiler is passed any `-R` or -`Wl,-rpath` flags, which some `Makefiles`
may do!
Hence, sometimes you must add your own.

Avoiding this completely
------------------------

TODO: MOVE TO BELOW NOTATION SECTION.

First of all, it should be noted that many libraries and software packages will
installable via your Linux distro's package manager.
If you see installation instructions containing `apt-get install <packagename>`
or `yum install <packagename>`, then that's the way to go.
Unfortunately, you do not have permission to install this way on your provided
desktop, and will need to submit a helpdesk ticket.
This method of installation is nonetheless recommended, as it avoids the very
real possibility that you will end up in
[dependency hell](https://en.wikipedia.org/wiki/Dependency_hell) (a situation
in which the package you with to compile and install requires other packages
you don't have, which themselves require packages you don't have, and so on).
In theory, it also means your machine will receive updates to those packages
when they are released by your distribution.


Checking if you already have a library installed
------------------------------------------------

Before proceeding with the below, it is also wise to check that you do not
already have access to the library you're about to compile and install.
For this, the `locate` command is very useful.

Suppose you know you need [LAPACK](http://www.netlib.org/lapack/), the Linear
Algebra Package.
Before installing it manually, run

```bash
locate liblapack.*
```

and see what turns up.

TODO: This is not useful in its current form. It does not tell users how to
interpret the (not-always-obvious) result of their locate, does not account
for the fact that the locate database may not have been built (but mine is,
and that requires sudo updatedb, so...?) and, most importantly, does not
tell a user how to go from "I need LAPACK" to "the library to search for is
liblapack.*"


Compiling from Source
---------------------

Assuming then that we definitely do need to compile and install something from
source, you'll first want to download the source code.
For this example, we're going to use the
[GNU Multiple Precision Arithmetic Library](https://gmplib.org/).
This is not an ideal example, as it can be installed on Debian via an `apt-get`
and should therefore probably be done by the IT staff via a helpdesk ticket.
But it's nonetheless a good example, and builds with autoconf, as many libraries
will.

First step is to download the source file.
As of July 2015, the most recent version is 6.0.0a, and we download
`gmp-6.0.0a.tar.bz2`.
Copy this file into `~/sources/`, and then,

```bash
cd ~/sources/
tar -xvf gmp-6.0.0a.tar.bz2
cd gmp-6.0.0a
ls
```

Two files are of note: `README` and `INSTALL`.
It is __always strongly recommended__ to check any `README` or `INSTALL` files
if they are present, since they will often walk you through the installation
process! So,

```bash
less README
```

(You can scroll up and down the file with the keyboard and mouse wheel when
using `less`.)
In this case, the README is not particularly useful as far as installation
goes.
Just hit `q` on your keyboard to exit `less`.
Let's try

```bash
less INSTALL
```

This is much more useful, and immediately tells us to run

```bash
info -f doc/gmp.info
```

(Again, hit `q` on your keyboard to exit `less` before running the above.)
`INSTALL` also contains some quick-and-dirty installation instructions, which
in this case would likely be enough.
However, since more detailed information is available, we're going to look at
it.
After running the info command, scroll down to section "2.1 Build Options",
which provides us with information not in the `INSTALL` file.
There are many options available, the most important of which is `--prefix`.
This will be used for __every__ installation you do in which a `./configure`
script is run!
`--prefix` sets the root installation directory, which in our case is
`$HOME/local/`, so the syntax is

```bash
./configure --prefix=$HOME/local/
```

But we're not done yet!
There are some other options we might want:

- Native compilation: this allows CPU-specific code to be used, meaning faster
code, if you know which CPUs you're compiling for.
- Fat binary: this is similar to the above, but specifically for x86 CPUs (i.e.
all CPUs you are likely to be compiling for!) Enabling this option means that
the final code will include all x86 optimizations, and they will be used if you
happen to be running on a CPU for which optimizations exist.
- C++ Support: GNU MP has a C++ interface, using classes, but this exists in a
separate library to the C interface, and is not compiled unless you request it
(presumably so that the default configuration does not require a C++ compiler).

You may read through the rest, but these are the options we are going to use.
Hit `q` to exit the info screen.
Our configure command is then

```bash
./configure --prefx=$HOME/local/ --enable-cxx --enable-fat
```

This will take a few minutes to run, and output a long list of checks.
In general, it is usually not necessary that every line in the output end in
`... yes`, since it may be checking for this which are not essential.

Once this `./configure` script has finished, it outputs some summary
information.
Additionally, if we scroll back up to a little below the beginning, look for
set of lines similar to

```bash
using ABI="64"
CC="gcc -std=gnu99"
...
CXX="g++"
...
MPN_PATH=" x86_64/fat x86_64 generic"
```

Here we can see the results of our `--enable-` flags: a C++ compiler has been
detected, and the `MPN_PATH` is including `x86_64/fat` before the generic C
code (which will run on any x86 CPU).
Also notice that `./configure` has automatically detected that we are running
on a 64-bit system (although, in 2015, it is highly unlikely you _won't_ be
running on a 64-bit system in x86).

According to the installation instructions, the next steps are

```bash
make
make check
```

Running `make` will output a _lot_ of commands; these are all the compilation
and linking tasks writting into the Makefile - you do not need to do any
compiling yourself, it's all handled by the Makefile generated by autoconf!

In order to speed up the process, we can replace `make` with `make -j N`, where
N is the number of simultaneous tasks you wish to complete.
On a 4-core CPU, you could try e.g. `make -j 6`.
This will accelerate the compilation time, but if anything goes wrong during
compilation it can make the already-dense output even more difficult to parse,
so only use it if you really need to save the time and/or are confident the
build will complete without error.

In this case, the installation instructions strongly recommended `make check`,
which will run a set of tests on the resulting binary files to ensure they were
correctly compiled.
Not all packages will necessary have such testing, you'll need to check the
install documentation.
If `make check` is given, however, you should always run it!
Scrolling up through the output of `make check`, you should see various lines
like

```bash
==================
All 9 tests passed
==================
...
===================
All 63 tests passed
===================
```

showing that the various different tests were passed.
Any errors in the above are somewhat unexpected, as all passed on the
configuration used when writing these instructions (and that configuration
should be similar to yours, possibly with an older version of gcc and g++).
Google is probably your best bet, once you've verified you followed the above
instructions correctly.

Finally, then, we run

```bash
make install
```

which will copy all library files into `$HOME/local/lib/`, and all header files
into `$HOME/local/include`.
To verify this, do

```bash
cd ~/local/
ls lib
ls include
```

And you should see various files, such as `libgmp.so`, `libgmpxx.so` from the
former `ls`, and `gmp.h` and `gmpxx.h` from the latter.

If you are compiling a different package, it may also install into other
directories, e.g. executable files in `$HOME/local/bin` and documentation in
`$HOME/local/doc`, `$HOME/local/man`, `$HOME/local/info`.

In this case, GMP should have installed some info pages, and the environment
variables we added to our `.profile` files should make this accessible.
Run `info gmp` to find out!
(This should result in the GMP manual coming up, for whatever version of GMP
you installed - this guide assumed version 6.0.0.)
