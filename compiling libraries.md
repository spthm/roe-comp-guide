Installing (and compiling) C/C++ libraries and applications from source
=======================================================================

Notation etc.
-------------

Basic command-line knowledge is assumed, although even if you are very
unfamiliar with the terminal, you should still be able to follow along.

Code blocks are things which the terminal prints out, or things you should be typing into the terminal.
They, `look like this`, or

    look like this first line
    and this second line

Lastly, if you see `<sometext>`, this is example or placeholder text, and
should not be copied verbatim!
For example, if your username is abc, and you see the text
`cd /home/<username>`, then you should instead type `cd /home/abc`!


Avoiding this guide completely
------------------------------

First of all, it should be noted that many libraries and software packages will be installable via your Linux distro's package manager.
If you see installation instructions containing `apt-get install <packagename>` or `yum install <packagename>` or `dnf install <packagename>`, would be the way to go.
Unfortunately, you do not have permission to install this way on your provided desktop, and will need to submit a helpdesk ticket.
This method of installation is nonetheless recommended, as it avoids the very real possibility that you will end up in [dependency hell](https://en.wikipedia.org/wiki/Dependency_hell) (a situation in which the software you wish to install requires other packages you don't have, which themselves require packages you don't have, and so on).
In theory, it also means your machine will receive updates to those packages
when they are released.


Prerequisites
-------------

Most importantly, the below assumes you are working on a provided Linux
workstation and that you have gcc, the
[GNU Compiler Collection](https://gcc.gnu.org/), installed.
The first of these should imply the second, but to be sure, open up a terminal
and type

```
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
You can check this by opening up a terminal and running `ps -p $$`.
Example output is

```
user@host:~$ ps -p $$
 PID TTY           TIME CMD
7001 pts/14    00:00:00 bash
```

The `bash` under `CMD` is the relevant bit of info.
It this says something else, e.g. `tcsh`, some parts of this guide may not quite work for you.


Initial Set-Up
--------------

You will want somewhere specific to put all of your compiled library files,
headers, and potentially the documentation.
A sensible choice is for them to be accessible via `/home/<username>/local/`;
however, your home directory quota is likely to be rather slim, so actually
placing them at this location is not advised.
Instead, make a directory `local` (or whatever you want to call it) on e.g.
`/disk1` of your machine, which should have plenty of space:

```
cd /disk1/<username>
mkdir local
```

This is the location we will _actually_ be installing everything into.
To make it conveniently accessible from your home directory, we're going to
create a symbolic link (symlink) to it,

```
cd ~/
ln -s /disk1/<username>/local
ls -l
```

That last command will list everything in your home directory; look for the
line similar to

```
lrwxrwxrwx 1 <username> users   16 Jan 01 12:00 local -> /disk1/<username>/local/
```

which shows that `/home/<username>/local` just points to the 'real'
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

where we have, as before, actually created the directory in `/disk1/<username>/sources` and simply symlinked this in our home folder.

Finally, with this directory in place, we need to make sure that the compiler and linker will actually be able to find things we install here!
For this task, we set some environment variables in our `~/.profile` file.
(As of July 2015, `~/.profile` is used by default; on other systems, it is
typical to instead use `~/.bash_profile`. To verify which of these exist on
your machine, run `ls -a ~/.* and pick the one which is already present.)

On Ubuntu, `.profile` is now ignored, and in general, `.profile` is
ignored if either `.bash_profile` or `.bash_login` exist!

See the [.bash_profile vs. /bashrc](bash_profile_vs_rc.md) article if you want to know more about these 'dot files'.

Any text editor is fine.
For [emacs](http://zoo.cs.yale.edu/classes/cs210/help/emacs.html#dummies), run `emacs ~/.profile`, and add the following to the __bottom__ of the file.
Lines beginning with `#` are comments and not necessary, though if you are not familiar with writing bash it would be wise to keep them.
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
    # Equivalent to compiling with -I$HOME/local/include
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

If you have done this and run into trouble while trying to compile something, or you want to understand _what_ you've done, read [the article on static and shared libraries, and runtime linking](static_vs_shared_libraries.md).
In particular, if you find your application compiles, but then complains about not being able to find libraries when you run it, see the 'Runtime linking' section.

In any case, it is worth noting that none of the above is ideal!
If you are using your own machine, where you can run `sudo`, you should always install libraries using the package manager where possible (`apt-get` on Debian and Ubuntu, `dnf` or `yum` on Redhat and Fedora, to name the most common options).
And, if your package manager does not have the library you want, there are still better procedures.
Again, see [the article on static and shared libraries, and runtime linking](static_vs_shared_libraries.md).

You may come across setting `LD_LIBRARY_PATH` as another solution, but it is considered
[bad practice](http://xahlee.info/UnixResource_dir/_/ldpath.html)


Checking if you already have a library installed
------------------------------------------------

Before proceeding with the below, it is also wise to check that you do not
already have access to the library you're about to compile and install.
For this, the `locate` command is very useful.

Suppose you know you need [LAPACK](http://www.netlib.org/lapack/), the Linear Algebra Package.
Before installing it manually, run

```bash
user@host:~$ locate *liblapack*
/usr/lib/liblapack.a
/usr/lib/liblapack.so
/usr/lib/liblapack.so.3
/usr/lib/liblapack.so.3.0
/usr/lib/lapack/liblapack.so.3
/usr/lib/lapack/liblapack.so.3.0
```

Your output may be different, but you can see that here, we already have this library installed.
Further, `/usr/lib` is a default location for installed libraries.
Anything in there should be automatically found during compilation.
Unforunately, you cannot (and should not) place libraries there yourself.

Other default locations include

```
/usr/lib32
/usr/lib64
/usr/local/lib
/usr/local/lib32
/usr/local/lib64
```

So if you spot anything in those directories, it's a good bet you already have the library.

Working out what to search for using `locate` is tricker.
Often, if a library is called X, then the library will be called `libX.so` or `libX.a`.
Sometimes this is not exactly the case, in which case your best bet is to search (the internet) for information about library X and its corresponding `lib<something>.so` or `lib<something>.a` files.


Compiling from Source with autoconf
-----------------------------------

Assuming then that we definitely do need to compile and install something from source, you'll first want to download the source code.
For this example, we're going to use the [GNU Multiple Precision Arithmetic Library](https://gmplib.org/).
This is not an ideal example, as it can be installed on Debian via an `apt-get` and should therefore probably be done by the IT staff via a helpdesk ticket.
But it's nonetheless a good example, and builds with autoconf (it contains a `configure` script), as many libraries will.

First step is to download the source file.
As of July 2015, the most recent version is 6.0.0a, and we download
`gmp-6.0.0a.tar.bz2`.
Save (or move) this file into `~/sources/`, and then,

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
In this case, the `README` is not particularly useful as far as installation
goes.
Just hit `q` on your keyboard to exit `less`.
Let's try

```bash
less INSTALL
```

This is much more useful.
Note that it tells us to run the `configure` script present in the folder.
If you are compiling a package which does _not_ have a `configure` script, and only has a `Makefile`, you'll need to edit it manually.

The info file immediately tells us to run

```bash
info -f doc/gmp.info
```

(Again, hit `q` on your keyboard to exit `less` before running the above.)
`INSTALL` also contains some quick-and-dirty installation instructions, which in this case would likely be enough.
However, since more detailed information is available, we're going to look at it.

After running the info command, scroll down to section "2.1 Build Options",
which provides us with information not in the `INSTALL` file.
There are many options available, the most important of which is `--prefix`.
__This will be used for every installation you do where a `./configure`
script is run!__
`--prefix` sets the installation directory, which in our case is `$HOME/local/`, so the syntax is

```bash
./configure --prefix=$HOME/local/
```

But we're not done yet!
There are some other options we might want:

- Native compilation: this allows CPU-specific code to be used, meaning faster code, if you know which CPUs you're compiling for.
- Fat binary: this is similar to the above, but specifically for x86 CPUs (i.e. all CPUs you are likely to be compiling for!) Enabling this option means that the final code will include all x86 optimizations, and they will be used if you happen to be running on a CPU for which optimizations exist.
- C++ Support: GNU MP has a C++ interface, using classes, but this exists in a separate library to the C interface, and is not compiled unless you request it (presumably so that the default configuration does not require a C++ compiler).

You may read through the rest, but these are the options we are going to use.
Hit `q` to exit the info screen.
Our configure command is then

```bash
./configure --prefx=$HOME/local/ --enable-cxx --enable-fat
```

This will take a few minutes to run, and output a long list of checks.
In general, it is usually not necessary that every line in the output end in `... yes`, since it will typically checking for more things than are required.

Once this `./configure` script has finished, it outputs some summary information.
Scroll back up to a little below the beginning, and look for set of lines similar to

```bash
using ABI="64"
CC="gcc -std=gnu99"
...
CXX="g++"
...
MPN_PATH=" x86_64/fat x86_64 generic"
```

Here we can see the results of our `--enable-` flags: a C++ compiler has been detected, and the `MPN_PATH` is including the optimized x86 code, `x86_64/fat`.
Also notice that `./configure` has automatically detected that we are running on a 64-bit system (`ABI="64"` - although, in 2015, it is highly unlikely you _won't_ be running on a 64-bit system).

According to the installation instructions (`less INSTALL`), the next steps are

```bash
make
make check
```

Running `make` will output a _lot_ of commands; these are all the compilation and linking tasks written into the `Makefile` - you do not need to do any compiling yourself, it's all handled by the `Makefile` generated by autoconf!

In order to speed up the process, we can replace `make` with `make -j N`, where N is the number of simultaneous tasks you wish to complete.
On a 4-core CPU, you could try e.g. `make -j 4`.
This will accelerate the compilation time, but if anything goes wrong during
compilation it can make the already-dense output even more difficult to parse, so only use it if you really need to save the time and/or are confident the build will complete without error.

In this case, the installation instructions strongly recommended `make check`, which will run a set of tests on the resulting binary files to ensure they were correctly compiled.
Not all packages will necessary have such testing, you'll need to check the
install documentation.
If `make check` is provided, you should always run it!

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
Any errors in the above are somewhat unexpected, as all passed on the configuration used when writing these instructions (and that configuration should be similar to yours, possibly with an older version of gcc and g++).
Google is probably your best bet, once you've verified you followed the above instructions correctly.

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

And you should see `libgmp.so`, `libgmpxx.so` from the former `ls`, and `gmp.h` and `gmpxx.h` from the latter.

If you are compiling a different package, it may also install into other
directories, e.g. executable files in `$HOME/local/bin` and documentation in
`$HOME/local/doc`, `$HOME/local/man`, `$HOME/local/info`.

In this case, GMP should have installed some info pages, and the environment
variables we added to our `.profile` files should make this accessible.
Run `info gmp` to find out!
(This should result in the GMP manual coming up, for whatever version of GMP
you installed - this guide assumed version 6.0.0.)


<!-- Compiling from Source with a Makefile

In the previous section, we ran a `./configure` script, which generated a `Makefile` for us.
When we ran `make`, the automatically-generated `Makefile` was used.
Sometimes, however, no `./configure` script is present and we must manually modify the provided `Makefile` so that it compiles on our system.

This is particularly quite common for astrophysics software packages.
In that vein, we're going to install the cosmological simulation code `Gadget-2`.

First, go to the code's [home page](http://wwwmpa.mpa-garching.mpg.de/gadget/) and [download the `.tar` file](http://wwwmpa.mpa-garching.mpg.de/gadget/gadget-2.0.7.tar.gz).
Save or copy that file into `~/sources/`, then,

```bash
cd ~/sources/
tar -xvf gmp-6.0.0a.tar.bz2
cd Gadget-2.0.7/
ls
```

From `ls`, you should see that there is a `README`.
Check the `README` with `less README`.
Not so helpful this time, as far as installing goes.
So, let's

```bash
cd Gadget2
ls
```

There is a file called `Makfile` here.
Open it up with `emacs Makefile`, or whatever text editor you want.
There are a whole bunch of options here, which are explained in the accompanying documentation for those interested.
Scroll down past all the `OPT += D<something>=` lines until you hit

```Makefile
CC       =  mpicc
OPTIMIZE =  -O2 -Wall -g
MPICHLIB =  -lmpich
```

These are default options for the compiler and [MPI](https://en.wikipedia.org/wiki/Message_Passing_Interface) library.
Just below this are various predefined `SYSTYPE` values.
Most are commented out.
We're going to add our own.
Comment out `SYSTYPE="MPA"` by putting a `#` in front of that line.
Then, above that line, add `SYSTYPE="IFA"`.

If you scroll down a little further, you'll see

```Makefile
ifeq ($SYSTYPE,"MPA")
CC        =  mpicc
OPTIMIZE  =  -O3 -Wall
GSL_INCL  =  -L/usr/common/pdsoft/include
GSL_LIBS  =  -L/usr/common/pdfsoft/lib -Wl,"-R /usr/common/pdsoft/lib"
[...]
```

The above are the settings for when `SYSTYPE` is `"MPA"`.
We're going to make our own for when `SYSTYPE` is `"IFA"`.
First thing, comment-out the line `OPT   +=  -DHAVE_HDF5` by putting a '#' at the start of it (line 43).
We're going to skip using the [HDF5](https://www.hdfgroup.org/HDF5/) library here.

Now, the `Makefile` needs to know where the `GSL` and `FFTW` libraries are stored.
By default, you probably won't have either of them.
[GSL](http://www.gnu.org/software/gsl/)  is the GNU Scientific Library and [FFTW](http://www.fftw.org/) is a library for computing discrete Fourier transforms.
Both should be available through the package manager.
Check you don't already have them with,

```
locate *libgsl*
locate *libfftw*
```

You'll probably find GSL is installed.
Further, if you read the documentation for Gadget-2, you'll find it requires FFTW2 (which is over a decade old).

So, modify your `Makefile` to look like this,

```Makefile
#--------------------------------------- Adjust settings for target computer

ifeq ($SYSTYPE,"IFA")
CC        = mpicc
OPTIMIZE  = -O3 -Wall
GSL_INCL  = -I/home/<username>/local/include
GSL_LIB   = -L/home/<username>/local/lib -Wl,"-R /home/<username>/local/lib"
FFTW_INCL = -I/home/<username>/local/include
FFTW_LIBS = -L/home/<username>/local/lib -Wl,"-R /home/<username>/local/lib"
MPICHLIB  =
HDF5INCL  =
HDF5LIB   =
endif

ifeq ($SYSTYPE,"MPA")
CC
[...] -->
