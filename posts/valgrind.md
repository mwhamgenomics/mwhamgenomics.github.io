---
extends: post.html
title: 'Debugging C programs with Valgrind'
date: 2017-05-22
category: programming
tags: ['environment', 'c']
external_links:
    - title: 'Valgrind official site'
      link: http://valgrind.org
---

Recently after adding a new feature to
[the fastq filterer used at Edinburgh Genomics](https://github.com/EdinburghGenomics/Fastq-Filterer), we got unexpected
segfaults when running the program on full-size input data. The problem did not occur on smaller test data, making us
think we were mishandling memory somewhere - very possible, since I'm still fairly new to programming in C. To find out
what was going wrong, I used a piece of software called Valgrind.

While advanced usage and interpretation of its optional outputs can get complicated, basic usage of Valgrind is very
simple. You simply attach it to your program by prepending it to the command line you would normally use:

    [mwhamgenomics]$ valgrind <valgrind_args> ./program <program_args>

By default, Valgrind will run a tool called [Memcheck](http://valgrind.org/docs/manual/mc-manual.html), which will look
for any dodgy memory management in your program as it runs and produce a report at the end. With the right arguments,
you can report on memory leaks, how many bytes of memory are affected, where they occur in the call stack, etc. This not
only helped me correct an `malloc` call (embarrassingly, I'd neglected to add 1 to account for a `\0` null terminator),
but also helped me learn about good practice for managing memory in C (allocate memory, then use it, then free it - no
implicit calls to `malloc` and `free` buried inside functions) and taught me some of the nuances of `strtok`, which I
was using to walk along a string (in brief, it returns a pointer to a segment of the string you pass in, so you need to
make sure you don't free the string until you're completely finished with it).

Some of Valgrind's other features include call mapping, inspections for threading and the cache, and attaching to a
[GDB](https://www.gnu.org/software/gdb) session. For something that deals with such a complex system, it turned out to
be easy to use, flexible, reasonably clear and informative, and full-featured.
