---
layout: post
title: A Title
category: programming
tags: ['bioinformatics']
external_links:
    - title: A link
      link: 'http://en.wikipedia.org/wiki/Main_Page'
      description: A description
    - title: Another link
      link: 'http//localhost:4000'
      description: Another description
---

Part of the functionality of EGCG's [Analysis Driver pipeline](http://github.com/EdinburghGenomics/Analysis-Driver) is to process BCL binary base call files from an Illumina HiSeqX into fastq sequence files. Most of the work is done by a program developed by Illumina called [bcl2fastq](http://support.illumina.com/downloads/bcl2fastq-conversion-software-v217.html), but we found a problem with the fastq files generated: some of their reads were very short - less than 36 bases. It turned out that these were a result of the adapters used in the sequencing, and didn't actually contain any real sequence data. As a result, we had to add a processing stage after running bcl2fastq to scan each fastq and, if a read there or in the fastq's corresponding partner was too short, filter it out.

<div>
    <div class="col-md-6">
        R1.fastq
        <code>
            @read1 len 8
            ATGCATGC
            +
            #--------
            @read2 len 6
            ATGCAT
            +
            #------
            @read3 len 3
            ATG
            +
            #---
            @read4 len 9
            ATGCATGCA
            +
            #---------
        </code>
    </div>

    <div class="col-md-6">
        R2.fastq
        <code>
            @read1 len 9
            ATGCATGCA
            +
            #---------
            @read2 len 3
            ATG
            +
            #---
            @read3 len 8
            ATGCATGC
            +
            #--------
            @read4 len 7
            ATGCATG
            +
            #-------
        </code>
    </div>
</div>
<div>Example of fastq filtering: with a filter threshold of 4, reads 2 and 3 would be filtered out of both fastqs.</div>

At first, we used a program called [Sickle](https://github.com/najoshi/sickle), which does what we required, but has a performance problem, in that it implements a sliding window. Sliding windows are a common statistical technique in bioinformatics where, for each base in a DNA sequence, a 'score' (of base quality, GC content, etc.) is calculated from all the surrounding bases. Apart from us not needing any of this base scoring functionality, this was also slowing the processing - Sickle took about 90 mins to process a pair of 4 Gb fastq files. In an attempt to optimise this stage, we decided to try rolling our own fastq filterer.


### Implementation
Firstly, we decided to compare Sickle with a control. A Python implementation of the functionality above took about 4.5 hours to process the aforementioned pair of 4 Gb fastqs. Now that we now know how to solve the problem, it's just a matter of optimising for speed.


### C
We decided to try a different implementation of the above Python script in C. Despite the popularity of high level languages like Python, Perl, R, etc. in bioinformatics, C has retained popularity where it's important to be able to fly through a huge text file at high speed. In fact, Sickle itself and many other bioinformatics tools like BCFTools are written in C. Our C program would have a series of tasks: read in a pair of corresponding entries from two fastq files, check both sequences and, if both were long enough, add them to a pair of corresponding output files.

(Note that this will not discuss actual programming in C - only the challenges and solutions involved. I will, however, have a separate article where I go into the C code)

Firstly, we had to read in data from a file (actually two files), which is possible with C's `stdio` library or `zlib`, depending on whether you need to read compressed files. Data files are almost always compressed in bioinformatics, so we used `zlib`.

Next, we have to read these files line by line. Again, this is possible using respective functions from `stdio` and `zlib`. An issue to consider here is memory management, which is completely hidden from the user in high-level languages like Python and Ruby, but must be done manually by the programmer in C. The functions that `stdio` and `zlib` provide make it very easy to read `x` number of characters from a file - you just allocate enough memory to read in `x` characters. However, what if we find a line in the file that is longer than `x`? If we're not careful here, we could end up corrupting our output files! The simple solution to this would be to set a very large value for `x`, however the more elegant (and future-proof) solution would be to dynamically allocate any amount of memory, depending on how long the line is. This is discussed in the companion article on C.

### Compressing output data
As mentioned above, data files are often GZ compressed in bioinformatics due to their sheer size otherwise. This means that our script needs to be able to read and write `fastq.gz` files. This is simple enough via C's `zlib` library. At first, our Fastq filtering script did this, however it turned out to run at about the same speed as Sickle, despite not doing any of Sickle's sliding windows or quality filtering! Clearly, there was something else that was slowing us down.

We next tried removing data compression/decompression from the script (reading and writing uncompressed Fastqs), which we found to improve performance significantly. It turned out that the rate-limiting step in our C script was the GZ compression of output data. Decompression of input data was not found to significantly affect performance. Fortunately, it is possible to compress data in parallel with a multi-threaded version of `gz`: [pigz](https://github.com/madler/pigz) by Mark Adler, co-developer of `zlib`. If we could output uncompressed data from our script and pipe that into `pigz`, we might be able to get some good results.

One problem, though: we couldn't use simple Unix pipes, because we have two output Fastqs (we could have used `stdout` and `stderr`, of course, but what if something went wrong and we had to print an error message?). Fortunately, Bash has a solution for this: process substitution. This allows you to create expressions that pretend to be files - they can be opened and written to, but underneath, they are Linux pipelines. It's probably easiest to give an example:


```Bash
set -o pipefail; ./filter_fastq_files --i1 ./R1.fastq.gz --i2 ./R2.fastq.gz --o1 >(pigz -c > ./R1_filtered.fastq.gz) --o2 >(pigz -c > ./R2_filtered.fastq.gz) --threshold 36
```

The C script we developed is fairly simple to use: the two input Fastqs (R1 and R2) are passed as `i1` and `i2`, the corresponding (filtered) output Fastqs are specified as `o1` and `o2`, and a length threshold at which to trim out a read pair is passed as `threshold`. The `>()` process substitutions may look unfamiliar, but they're not too complicated to understand. As far as the C script is concerned, the process substitution passed as `o1` is a file path, so it opens it using C's `stdio` library and writes to it. The substitution accepts the output data and, instead of writing it to disk, passes it as `stdin` to pigz, which compresses it and then writes it to disk. In this case, compressed data is read in, decompressed and processed by our script, and then passed to pigz via process substitution for recompression.

Using this approach, we were able to get the processing time for a pair of 4 Gb fastqs from and hour and a half to about 5 minutes.

This improvement in performance was less dramatic when working with full-size test files, however we still found the script to run in about 1/5 or 1/6 of the time Sickle was taking. As of this writing, we are now working to integrate this new C script into our pipeline, allowing us to filter Fastq reads much more quickly than before.

### Conclusions
- Firstly, C is fast - really fast! Remember that an equivalent Fastq filtering script in Python took about 3 times as long to run as Sickle or our initial C script.
- Secondly, while things like GZ compression can slow down programs, C is so fast that sliding windows, quality filtering and GZ decompression can be done without such significant slow-downs.
- Thirdly, multithreading of GZ compression is not only possible, but it's also very effective! Pigz probably deserves to be better known in scientific programming.
