### Static vs shared libraries, link-time vs run-time.

TODO: Define linker vs compiler in simple terms.

TODO: [Add this reference link](http://docencia.ac.upc.edu/FIB/USO/Bibliografia/unix-c-libraries.html)

Static and shared libraries
---------------------------

The point of a library is that it is reusable.
Say you have some code which implements some functions you end up using all the time; for example, computing Fourier transform of an input array (don't actually do this yourself, [a library already exists](http://www.fftw.org) for that).
Rather than rewriting this code in every application you need it, you write it _once_, compile that code into a library, and simply 'link' against this library whenever you need those functions.

With a __static library__, sometimes called a __statically linked library__, all compiled code from the library that your application uses is copied directly into your application's executable file by the 'linker'.
This means your executable file size could be quite large: it contains a copy of all the library functions it uses.
If you have lots of similar executables, all using the same library functions, this adds up.
On Linux, static library files have the __extension `.a`__, e.g. `libfft.a`.
(For the pedantic: `.a` files are technically _archives_, not libraries.)

To solve the executable file-size bloat problem, we can use __shared libraries__, sometimes called __dynamic libraries__.
Here, the library code is _not_ copied into our final executable.
Instead, our executable contains some small identifiers for the libraries it relies upon, and identifiers for the functions it uses.
When we run the executable, the 'runtime linker' first checks that it can find all the libraries our program needs.
It then loads the executable _and_ all the shared libraries it requires into
memory.
Thus, when a program calls a function in a shared library, code from the library file itself is run, and the result is passed back to our executable.
On Linux, shared library files have the __extension `.so`__ ('shared object', and often `.so.x.x`, where the `x.x` is some version number).

Shared libraries have several other benefits, including:
- Bug fixes in the library often only result in a new `.so` file, which will be entirely compatible with your application; you do not need to recompile!
- If lots of different processes request the same shared library, that shared library need only be loaded into memory once, saving RAM.

Their only major disadvantage is that they may perform slightly slower than an equivalent statically-linked program, since there is some overhead associated with calling a shared library's function and passing the result back to our main program.


Runtime linking
---------------

An `rpath` is the location the runtime linker, mentioned above, will look for shared libraries.
(Software-development systems are often very different from the systems where the code will actually be run, and it therefore became necessary to separately specify where to look for a library at compile time, vs. where to look for it when an application is run.)

For example, my `libfft.so` might be in `/opt/develop/fft/libs/libfft.so`, but I know that all users of my software will have it in a standard location, `/usr/local/lib/libfft.so`.

`-L` flags tell the compiler and linker where to look at _compile_ time.
In the proposed example, this is `/opt/develop...`.
For static libraries, this is entirely sufficient.

`-R` or `-Wl,-rpath=` flags write code _into your application_ that tells the
runtime linker where you look whenever the application is launched.
Such flags are often necessary for applications which use shared libaries.

In the proposed example, the location to search at runtime is `/usr/local...`.
The runtime linker also has a list of default locations it will look at.
If you install a shared library using your Linux distribution's package manager (e.g. `apt-get`, `dnf` or `yum`), it will be placed into one of these default locations.
If your package manager didn't have the package available, but you still had admin rights, then there are [still better ways](http://choorucode.com/2014/01/14/how-to-add-library-directory-to-ldconfig-cache/) to set your _shared_ library search paths, using [ldconfig](http://linux.die.net/man/8/ldconfig) - essentially, you add to the list of default search paths.

If you are without admin privileges, you are not able to install to these locations, nor are you able to add anything to the list of default search paths.
One must therefore add the location of any manually-installed shared libraries into _every single executable_ we compile which requires them.
This can be done by passing `-R <path>` or `-Wl,-rpath=<path>` to the compiler.

Setting `LD_RUN_PATH` typically handles this for us - if we add "~/local/lib" to it, then it will bake `~/local/lib` into every single application we compile as a runtime search path.
However, `LD_RUN_PATH` is _ignored_ if the compiler is passed any `-R` or -`Wl,-rpath=` flags, which some `Makefiles` may do!
In that case, you must manually alter the `Makefile` to ensure that your program is compiled with the necessary runtime search paths, e.g. `-R ~/local/lib`.
Simply add your own `-R your/runtime/search/path` to the `Makefile`.


Troubleshooting 'not found' libraries when _running_ applications
-----------------------------------------------------------------

To elaborate on the last point in the previous section:

The information in [the compiling libraries article](compiling libraries.md) should work for installs using autoconf, which have a `./configure` to be run before you actually do any compiling or installing.
However, if autoconf is not used _and_ the Makefile includes some rpaths, then it will probably fail.
rpaths look like either of the following when passed to the compiler:

```
-R some/directory/path
-Wl,-rpath=some/directory/path
````

and simply

```
-rpath=some/directory/path
```

when passed to the linker.

You program will compile fine, but when you run it, it will be unable to find any of the libraries you have manually installed .

The simplest way around this is to add the following to an appropriate location in the Makefile

```make
-Wl,-rpath=$HOME/local/lib
```

where 'appropriate' depends on the Makefile in question, but in general
anywhere you see `-L` flags is probably okay.
If there are already `-R some/path` or `-Wl,-rpath=some/path` flags present, simply add your own after these.
For a more in-depth discussion, see e.g.
[here](http://www.eyrie.org/~eagle/notes/rpath.html)
