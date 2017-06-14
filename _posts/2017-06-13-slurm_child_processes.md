---
layout: post
title: 'Slurm and disappearing jobs: parallel processes in non-terminal environments'
category: Programming
tags: ['bash', 'linux']
---

Lately, I've been puzzling over a problem with [our fastq filtering process](/biology/2016/10/14/filtering_fastq_reads.html) at work. We recently added a new feature to our [fastq filterer](http://github.com/EdinburghGenomics/Fastq-Filterer) to write a text file with statistics of the data processing performed. When we ran this new version in production, though, we found that sometimes this stats file was not being written, even though the Slurm job running it was returning a 0 exit status. This prompted a long investigation that has been my primary source of annoyance over the past few weeks, but taught me something useful in the end.

## Debugging
My first idea was to check the fastq filterer itself for bugs - I'm not that experienced with C, after all. After a lengthy investigation with [Valgrind](/programming/2017/05/22/valgrind.html), I had much tidier memory management and reassurance that my C programming abilities aren't as shabby as I initially feared (in hindsight, how complicated and error-prone can writing to a file be?). I still had no answer, though, and I had to be sure that the randomly-disappearing stats files were not due to a problem in the script.

## Reimplementation
I tried implementing the filterer in Python, and the problem remained. This acquitted the filterer of any wrongdoing. I then tried adding some debugging `print` statements into the script, and found something interesting: whenever a stats file failed to write, I was also getting print statements failing to work. It seemed that either the IO was getting cut off, or the job was getting killed.

I tried running this re-implemented script with `python -v`, which prints extra information about what the interpreter is doing. Surprisingly, whenever I got a failing stats file and print statements, I also got none of the cleanup messages I expected from the interpreter at the end of execution. Not only was the script seemingly getting stopped, the entire Python interpreter was crashing silently (Slurm was still returning an exit status of 0)! Clearly something funny was going on here, and it wasn't bugs in my code.

## Narrowing it down
Since stats files were only failing to write occasionally, my boss suggested that it was a race condition. Race conditions occur when concurrent threads or processes don't complete in the correct order - two threads 'race' to completion and if the wrong one finishes first, weird things happen. I tried putting some `sleep` statements around the code that wrote my stats file, which made the problem occur every single time. This may not sound like an improvement, but we are now one step closer to identifying the problem.

The next step was to eliminate more factors from the environment in which we were running the fastq filtering. The problem did not occur in a plain Bash session, so it was something to do with Slurm. The script was being called inside a complex bit of Bash syntax including the use of `set -e`, the removal of which did not affect anything. Our filtering process made use of [Unix named pipes](/programming/2016/11/28/bash_io.html) for GZ compression of output data on the fly, and if we removed this, the problem did not occur. When we tried moving all `fclose()` calls to the very end of the filter script, the problem also did not occur. At this point, it looked like the problem was something to do with the way Slurm handles the closing of named pipes (although it turned out not to be).

## A minimal test
The race condition and the parallel processes seemingly getting killed indicated that it could be a problem with named pipes, or a problem with concurrent child processes in general. Just to be sure that the problem wasn't caused by us before getting angry at our sysadmins, we tried a very minimal test case, consisting of a Python script run under Slurm:

test_concurrency.py:
{% highlight python %}
from time import sleep

print('Writing some data to f1')
f1 = open('f1.txt', 'w')
f1.write('some data\n')
f1.close()
print('f1 done')
sleep(5)

print('Writing some more data to f2')
f2 = open('f2.txt', 'w')
f2.write('some more data\n')
f2.close()
print('f2 done')
{% endhighlight %}

test_concurrency.slurm:
{% highlight bash %}
#!/bin/bash
#SBATCH --job-name="test_concurrency"
#SBATCH --cpus-per-task=1
#SBATCH --mem=2g
#SBATCH --output=path/to/test_concurrency.slurm.log

function test_concurrency {
    echo "[slurm script] Starting"
    python test_concurrency.py > concurrency_test.log
    echo "[slurm script] Done"
}

# test case 1: single command only
test_concurrency

# test case 2: running the script as a child process
test_concurrency & echo "Doing some other stuff"
{% endhighlight %}

When we ran this with test cases 1 and 2 alternately commented in/out, we found that test case 1 worked fine, but test case 2 failed to write files, and there was no `'[slurm script] Done'` echoed into the Slurm log file. We then found that adding a `wait` statement in the Slurm script after starting the two concurrent jobs solved it. At last we had found the problem!

## The race condition explained
In this case, the problem was a result of using Bash child processes incorrectly in a non-terminal environment (in this case, Slurm). In test case 2, we kicked off a `test_concurrency` function in the background and then echoed a string to stdout in the foreground. Once the `echo` has run, as far as Slurm is concerned, the script has completed, even though the Python process is still running. Slurm then stops executing and closes the parent Bash process, cutting off any child processes. This did not happen with my tests in an interactive Bash session because there is no cut-off point for completion of child processes in a long-running interactive terminal.

Let's apply this to the situation we had with the fastq filterer:

fastq_filterer.slurm
{% highlight bash %}
mkfifo r1_filtered.fastq; mkfifo r2_filtered.fastq

./fastq_filterer --threshold 36 --i1 r1.fastq.gz --i2 r2.fastq.gz --o1 r1_filtered.fastq --o2 r2_filtered.fastq &
pigz -c r1_filtered.fastq > r1_filtered.fastq.gz &
pigz -c r2_filtered.fastq > r2_filtered.fastq.gz
{% endhighlight %}

Here, we kick off the filterer and a pigz in the background, and another pigz in the foreground. The filterer will process its inputs, writing to the `fifo` named pipes and closing them when it's finished. At this point, the two pigz processes will complete. Now because the foreground pigz has finished, Slurm exits, cutting off the filterer, even if it's still busy writing its stats file. This explains both the missing stats files and why moving the `fclose()` calls to the end of the script fixed it (they essentially delayed the exiting of the foreground pigz).

## The fix
All we need to do in this case is explicitly wait for all child processes to finish. This is possible with a Bash command aptly called `wait`:

{% highlight bash %}
./fastq_filterer --threshold 36 --i1 r1.fastq.gz --i2 r2.fastq.gz --o1 r1_filtered.fastq --o2 r2_filtered.fastq &
pigz -c r1_filtered.fastq > r1_filtered.fastq.gz &
pigz -c r2_filtered.fastq > r2_filtered.fastq.gz &  # we can now run everything as a child process
wait  # with no arguments, this will wait for all child processes
{% endhighlight %}

## A more elaborate fix
The addition of a simple four-character command at the end of the script does solve the problem, but we wanted to be able to look at the exit status of each child process. To do this, we would need to capture the pid of each process with `$!` and wait on them explicitly:

{% highlight bash %}
./fastq_filterer --threshold 36 --i1 r1.fastq.gz --i2 r2.fastq.gz --o1 r1_filtered.fastq --o2 r2_filtered.fastq &
fq_filt_pid=$!
pigz -c r1_filtered.fastq > r1_filtered.fastq.gz &
pigz_r1_pid=$!
pigz -c r2_filtered.fastq > r2_filtered.fastq.gz &  # we can now run everything as a child process
pigz_r2_pid=$!

wait $fq_filt_pid  # wait returns the exit status of the pid specified
fq_filt_exit_status=$?
wait $pigz_r1_pid
pigz_r1_exit_status=$?
wait $pigz_r2_pid
pigz_r2_exit_status=$?
{% endhighlight %}

It doesn't matter which order you wait for the pids, because you can still call `wait` on a child process after it's finished, and even call it multiple times. I couldn't find any discussion of this subject online, so hopefully this will be an aid to anyone encountering problems with parallel processes within distributed compute jobs. To conclude:

- To find the cause of a problem, try to eliminate as many external factors as possible from your testing
- When something randomly fails, it may be a race condition, in which case `sleep` statements can be useful in finding the cause
- _Always_ wait on your child processes.
