---
extends: post.html
title: Simple file processing in C
date: 2016-10-16
category: programming
tags: ['c']
external_links:
    - title: EGCG Fastq-Filterer
      link: 'https://github.com/EdinburghGenomics/Fastq-Filterer'
      description: Source code for the fastq filtering program we developed.
    - title: tcezard
      link: 'https://github.com/tcezard'
      description: My boss at Edinburgh Genomics and the source of this post's opening quote.
---

> "With high level languages you stand on the shoulders of giants, whereas with C you stand on the shoulders of lots of
  little mice that run really fast."

Recently, I wrote an article about some of Edinburgh Genomics' exploits in fastq read length filtering. At first, we
were using a tool called [Sickle](https://github.com/najoshi/sickle), however it was causing a significant slow-down in
our pipeline. Upon discovering that Sickle implemented a costly (we thought) sliding window, we decided to roll our own
solution to this problem in C.

## Implementation
Firstly, despite knowing it would be slow, we decided to 'sketch' an implementation in pure Python:

```python
def filter(r1i, r2i, r1o, r2o, threshold):
    while True:
        r1_header = r1i.readline()
        r1_seq = r1i.readline()
        r1_strand = r1i.readline()
        r1_qual = r1i.readline()

        r2_seq = r2i.readline()
        r2_header = r2i.readline()
        r2_strand = r2i.readline()
        r2_qual = r2i.readline()

        if len(r1_seq) >= len_threshold or len(r2_seq) >= len_threshold:
            r1o.writelines((r1_header, r1_seq, r1_strand, r1_qual))
            r2o.writelines((r2_header, r2_seq, r2_strand, r2_qual))
```

Here, we see a good example of the trade-off between processing speed and development speed. We had a full Python script
implementing the above functionality in about 10-15 minutes, however it took about 3 times the time that Sickle took to
process a pair of 4 Gb fastqs. This may look like an unsuccessful experiment, but it's perfectly appropriate to sketch a
solution in a high-level language before optimising in something like C, which has a much faster running speed, but a
much slower development speed.


## C
Implementing the above functionality in C would present a whole new set of challenges: not only was C a new language to
me at the time, but it was also quite different from anything else I was used to at the time. It lacks things like
dynamic typing and automatic memory management, and is a product of an older era and programming style, where it's not
always clear what the best implementation is for a given solution. My boss remarked that with high level languages you
stand on the shoulders of giants, whereas with C you stand on the shoulders of lots of little mice that run really fast.

Our program would have a series of tasks: read in a pair of corresponding entries from two fastq files, check both
sequences and, if both were long enough, add them to a pair of corresponding output files. Let's get to it.


## Reading in data
C's standard library may not have the awesome size and diversity of Python's standard library, but it does have
everything we need. File IO is provided in the `stdio` library:

```c
char* readln(FILE* f) {
    char* line = malloc(sizeof (char) * 1024);
    fgets(line, 1024, f);
    return line;
}
```

Here, we implement a function that takes a `FILE` handle and reads a line from it. To a Python programmer, most of this
function seems fairly easy to follow. The function `fgets` is used to read in text from a file until a '\n' newline or
1024 characters have been read, and insert that string into a variable called `line`, which is then returned. The first
line of the function, however, appears completely unfamiliar. Another thing that I at first didn't understand from
browsing C code was the seemingly random scattering of asterisks (`*`) around variable names and function declarations.

What the asterisk (`*`) does is declare a pointer - a variable that, rather than storing some data itself, stores the
location of some data in memory. The first line, then, declares a `char` pointer. `malloc` is part of C's memory
management - here, we allocate the `line` char pointer enough memory to store 1024 characters (a char is one byte
anyway, but other data types can be more). Now that the `line` pointer has access to some defined memory, we can store
some data and return the pointer. Using pointers and memory allocation in this way, we can pass strings and other data
structures around between functions.

There are two caveats with this implementation. Firstly, we need to ensure that we free up any memory we allocate once
we've finished using it, using the `free` function - C's counterpoint to `malloc`. In the example above, we can't really
free the `line` pointer after returning it (1, any code after a `return` statement won't be executed; and 2, we haven't
finished with the data yet!), so we free it after the function has been called and we've finished with the data:

```c
FILE* file = fopen("a_file.txt", "r");    
char* line = readln(file);
do_something_with(line);
free(line);  // readln does an malloc, so free the memory afterwards
```

If we don't free the memory after using it, the script will keep allocating more and more new memory as it runs, causing
a memory leak - a buildup of memory that's allocated but not used. Besides robbing the rest of the computer of RAM, your
program can eventually run out of free memory and crash.

Secondly, it's pretty hard to miss the hard-coded limits on the memory allocated and data read in.
It's very easy to use `fgets` to read in a line of up to `x` characters, however what if we find a line line longer than
`x`? Consider a file with the following content:

    a line\n
    a much longer line that is longer than 1024 characters [...]\n
    another line\n

I've added the '\n' line terminators for the sake of the example. Calling the above `readln` function once will return
the string 'a line\n', which we may then write to an output file. Calling it again will read in the next line. If it's
longer than 1024 characters, not only will the returned string be truncated, it will also be missing the '\n'
terminator! Of course, if we try to write this to an output file and then run the function a third time for 'another
line', the string 'another line' will be appended to the second line of the output file, because there was no '\n' on
the end of the second line. In the context used here, we would end up with corrupted fastq files!

## Accommodating long lines
Of course we could just set the block size to something bigger than 1024, but if we want to be able to tolerate lines of
any length, we'll need to do some logic, such as the following:

```c
#define block_size 1024

static char* readln(FILE* f) {
    int b = block_size;
    char* line;
    line = malloc(sizeof (char) * b);  // initial memory allocation
    line[0] = '\0';  // ensure array is empty from any previous function calls

    while (line[strlen(line) - 1] != '\n') {
        b += block_size;
        line = realloc(line, b + 1);  // increase line's memory

        // temporary array for fgets
        char* line_part = malloc(sizeof (char) * block_size);
        line_part[0] = '\0';

        if (fgets(line_part, block_size, f)) {
            // fgets returned something, so store it and free line_part
            strcat(line, line_part);
            free(line_part);
        } else {
            // fgets returned nothing, so have reached the end of the file
            free(line_part);
            return line;
        }
    }
    return line;
}
```

Here, we define an array pointer called `line` and allocate some initial memory to it. We then enter a `while` loop
where, as long as `line` doesn't end with '\n', keep reading in from the file. We now increment the block size,
reallocate memory for `line`, and create a temporary array, `line_part`, to use with `fgets` to read in data from the
file. If `fgets` read some data, it is concatenated onto `line` via `line_part`. If it didn't read any data, then the
end of the file has been reached, so return `line`.

Due to the extra memory management done in this function, it is slightly slower than just hardcoding a maximum block
size. However, this turned out not to be a problem, as it was not the rate-limiting step. The rate-limiting step turned
out to be recompression of output data, as discussed [here](/biology/2016/10/14/filtering_fastq_reads.html).

As it stands, our C script reads in GZ compressed data, checks read lengths, and writes uncompressed data to output
files. Compressed output files can be obtained as discussed in the original article that this accompanies.

Now that we know what the rate-limiting step is, we have some lee-way regarding how much logic we can do in this script.
We can now, for example, compare the R1 and R2 read IDs for similarity or do some simple quality checking. This should
enable us to filter fastq files more effectively without slowing down our demultiplexing and fastq production pipeline.
