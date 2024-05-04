---
extends: post.html
title: Compiling GCC on OSX
date: 2017-04-12
category: programming
tags: ['environment']
external_links:
    - title: 'GCC official site'
      link: https://gcc.gnu.org
---

> Note from 2024 - don't actually do this. When I first wrote this, I was less familiar with environment management
  systems, plus they were less mature in general. Use [Conda](https://docs.conda.io) instead.

Recently, during the course of trying to install a piece of software, I found myself trying to install an isolated
instance of GCC version 4.7 on OSX, which turned out to be something of an adventure.

## Dependencies
Firstly, after downloading the source code from [one of GCC's FTP mirrors](https://gcc.gnu.org/mirrors.html), I
initially tried configuring and compiling it. I immediately found I needed three other C libraries: GMP, MPC and MPFR.

GCC's documentation claimed that downloading the source code for these into subdirectories inside the GCC directory
would cause them to be built along with GCC. This sounded wonderful, however when I tried it, I found that MPFR wasn't
being picked up properly.
[A StackOverflow thread](http://stackoverflow.com/questions/9297933/cannot-configure-gcc-mpfr-not-found) mentioned a
change in file structure in later versions of MPFR. GCC 4.7 is a fairly old version, and I had tried it with the latest
versions of GMP, MPC and MPFR, since finding contemporary versions of them turned out to be complicated. The lesson
learned from this is that documentation claiming to require dependency version, say, 0.1.2 'or later', should be taken
with a pinch of salt.

Eventually, I found that all of this tedious manual downloading isn't actually necessary, as GCC provides a script in
`contrib/download_prerequisites` which will not only automatically download and link GMP, MPC and MPFR, but also the
correct versions of them.

## Segfaults
Having resolved dependencies properly, I then hit a problem with segfaults during the compilation process. At this
point, we are in a strange fractal scenario, where we are trying to compile a version of GCC with another pre-existing
version of GCC. After a bit of reading, I found that said pre-existing GCC on OSX... isn't actually GCC. It is, in fact,
Clang pretending to be GCC! I was able to resolve the segfaults I'd been getting by setting an environment variable:

    [mwhamgenomics]$ export MACOSX_DEPLOYMENT_TARGET=10.9

This informs OSX's developer tools of the minimum version of OSX required. The underlying function of this environment
variable is still a bit of a mystery to me, but as far as I can tell, it allows libraries to be linked in the right way
for this old version of GCC - you'll notice later on that even having built GCC, it won't run (even `--version`) without
this flag set.

## System headers
Now upon configuring and compiling, I found that the stopping point was locating the C standard library, which GCC
expects to be in `/usr/include`, which does not exist in OSX. This can be solved by supplying a
`with-native-system-header-dir` argument at configure time.

At this point, GCC proceeded to compile. This took several hours, but at last, I had a genuine functioning version of
GCC 4.7 on Mac OSX.
