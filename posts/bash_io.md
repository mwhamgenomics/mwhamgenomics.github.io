---
extends: post.html
title: 'Bash IO: handling multiple output files'
date: 2016-11-28
category: programming
tags: ['bash', 'linux']
external_links:
    - title: EGCG Fastq-Filterer
      link: 'https://github.com/EdinburghGenomics/Fastq-Filterer'
      description: Source code for the fastq filtering program we developed
    - title: Process substitution on TLDP
      link: 'http://www.tldp.org.LDP/abs/html/process-sub.html'
      description: Notes on process substitution on The Linux Documentation Project
---

In the course of developing
[a C tool for filtering paired-end fastq files](https://github.com/EdinburghGenomics/Fastq-Filterer), we ran into a
performance problem. C is extremely fast, however we encountered a
bottleneck at the point of re-compressing output data. This was because we were compressing in a single thread using C's
`zlib` library. We eventually decided to remove integrated output data compression, and compress instead via an external
tool.

The script we developed takes two input fastqs (R1 and R2) passed as `i1` and `i2`, the
corresponding (filtered and uncompressed) output fastqs are specified as `o1` and `o2`, and a length threshold at which
to trim out a read pair is passed as `threshold`:

    [mwhamgenomics]$ ./fastq_filterer --i1 r1.fastq.gz --i2 r2.fastq.gz --o1 filtered_r1.fastq --o2 filtered_r2.fastq --threshold 36

Now to compress the output data. There is a Linux tool called [Pigz](https://github.com/madler/pigz) that solves our
performance problem, since it allows compression of data in multiple threads. The only problem left is linking this up
via pipes without writing any intermediate files to disk.

If we were only dealing with one output fastq, we'd just be able to pipe it into pigz via stdout, however in this
case, there are two. We could use stdout and stderr, but what if something goes wrong and the filterer has to print an
error message? Fortunately, Bash has solutions for piping multiple channels of data between programs.

## Named pipes
Named pipes use the `mkfifo` command, which creates what appears to be a file on your filesystem. However, this is more
like a Python `StringIO` object, able to stream data instead of writing it to disk. Having created these named pipes, we
can now use parallel commands to stream data to/from them concurrently:

    mkfifo intermediate_r1.fastq
    mkfifo intermediate_r2.fastq

    ./fastq_filterer --i1 r1.fastq.gz --i2 r2.fastq.gz --o1 intermediate_r1.fastq --o2 intermediate_r2.fastq --threshold 36 &
    pigz -c intermediate_r1.fastq > r1_filtered.fastq.gz &
    pigz -c intermediate_r2.fastq > r2_filtered.fastq.gz

    rm intermediate_r1.fastq intermediate_r2.fastq

Here, we have the filterer write its output to named pipes which then pass data to Pigz for compression. Note the use of
`&` in the first two main commands, and the absence of `&` in the third. Having created the named pipes, we want to
start up three commands in parallel: one writing to them, and the other two reading them and passing to pigz. It's also
important to delete the named pipes afterwards. This, then, is a very useful and powerful construct, if a little
verbose.

## Process substitution
Process substitution is an extension of named pipes, and allows a more elegant syntax. Here, you create expressions that
pretend to be files - they can be opened and written to, but underneath, they are Linux pipes. It's probably easiest to
describe how they're used with an example:

    set -o pipefail
    ./filter_fastq_files --i1 r1.fastq.gz --i2 r2.fastq.gz --o1 >(pigz -c > r1_filtered.fastq.gz) --o2 >(pigz -c > r2_filtered.fastq.gz) --threshold 36

Process substitutions are created with the syntax `>(...)` if you're writing to them, or `<(...)` if you're reading. The
filterer opens and writes to them as if they were files, however the `(...)` allows you to specify some processing to
apply to the data - in this case, piping into pigz.

We encountered a couple of difficulties with process substitution. Firstly, it's recommended to use `set -o pipefail` to
ensure that the whole pipeline returns a non-zero exit status if one command fails. Secondly, when I tried to put this
into a script to run on Slurm, I found that Slurm does not like process substitutions. It was, however, able to use
named pipes, so we were still able to solve the problem.

Even with complex IO problems, Bash has powerful functionality for streaming data between commands. Using this, we were
able to link one of our own scripts to multithreaded data compression, significantly improving the speed of our pipeline
while avoiding writing too many intermediate files to disk.
