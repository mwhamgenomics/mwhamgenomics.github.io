---
layout: post
title: Multiprocessing and the dark side of concurrency
category: Programming
tags: []
---

Some of our automated processing at Edinburgh Genomics uses Python's `multiprocessing` standard library module to
parallelise tasks. This opens some opportunities for things to get complicated at times, and we found this out a
few months ago when we noticed SSL errors occurring seemingly at random from our software's interactions with web
services.

We were using `multiprocessing` as part of [Luigi](https://github.com/spotify/luigi), a pipelining framework
developed by Spotify. Whereas the `threading` module spawns new threads and is therefore affected by Python's
[Global Interpreter Lock](https://docs.python.org/3/glossary.html#term-global-interpreter-lock), `multiprocessing`
parallelises tasks by splitting them off into different processes, bypassing the GIL and allowing true parallelism
and use of multiple CPUs.

At the time, we'd already been seeing randomly-occurring SSL errors in a process connecting to an internal web
interface, and had [implemented a fix](https://github.com/EdinburghGenomics/EGCG-Core/pull/82) for this. The fix
used a Retry object from urllib3. This had to be added to a `requests.HTTPAdaptor` object, which in turn had to
be mounted to a `requests.Session` object. This meant that instead of using `requests`' default methods, we had
to build a custom Session object. At the time, we decided we might as well cache this Session object so we could
use the same one when we made many requests in quick succession - unfortunately, this turned out to be the cause
of our problems.

When we put this change into production and still saw SSL errors, we had more searching to do. Eventually we found
a couple of discussions [here](https://github.com/kennethreitz/requests/issues/4323) and
[here](https://stackoverflow.com/questions/3724900/python-ssl-problem-with-multiprocessing/3724938),
implicating multiprocessing in the weirdness that we were seeing.

When a new process is started with multiprocessing, the Python session is duplicated into the new process, along
with all its existing objects. This can be proven by writing a short script:

{% highlight python %}
import multiprocessing

class Thing:
    def __init__(self):
        self.data = None

    def __str__(self):
        return '<%s at %s: data=%s>' % (self.__class__.__name__, hex(id(self)), self.data)

t = Thing()

def run(proc_number):
    t.data = proc_number
    print('In subprocess %i: t is now %s' % (proc_number, t))

procs = set()
for i in range(1, 5):
    procs.add(multiprocessing.Process(target=run, args=(i,)))

print('In main thread before multiprocesses: t is %s' % t)
for p in procs:
    p.start()

for p in procs:
    p.join()

print('In main thread after multiprocesses: t is %s' % t)

run(0)
print('In main thread after control experiment: t is %s' % t)
{% endhighlight %}

Here, we have some mutable data which is modified by various sub-processes, printing the results to stdout in the
process. The result of running this is:

```
In main thread before multiprocesses: t is <Thing at 0x102999c88: data=None>
In subprocess 4: t is now <Thing at 0x102999c88: data=4>
In subprocess 2: t is now <Thing at 0x102999c88: data=2>
In subprocess 1: t is now <Thing at 0x102999c88: data=1>
In subprocess 3: t is now <Thing at 0x102999c88: data=3>
In main thread after multiprocesses: t is <Thing at 0x102999c88: data=None>
Modified t in process 0 to <Thing at 0x102999c88: data=0>
In main thread after control experiment: t is <Thing at 0x102999c88: data=0>
```

As can be seen, the subprocesses modify `t` and report that they have successfully done so. After the
multiprocessing has exited, though, it is reported that `t` has not changedin the main process. Modifying it then
in the main process works as expected. The explanation for this behaviour is that each sub-process has its own
copy of `t`. This approximately mirrored our setup - we had a global shared Session object, which was being called
throughout our pipeline through Luigi tasks, which due to their use of `multiprocessing` ended up with their own copy
of this Session object.

The kicker here is that when you copy a Session object, the copy will be using the same SSL connection. If the
original Session and the copy then both make a request at the same time, the data of both requests gets scrambled
together on the same I/O stream.

As discussed in the requests issue linked above, there is no generic way of solving this inside of `requests` or
`multiprocessing`, so the solution was to handle the forking and Sessions ourselves. Fortunately, each process
will have its own pid, so we were able to use `os.getpid()` to ensure that we were always using a different
Session from the main process.

I usually find that when there is a seemingly incomprehensible problem causing hours of headscratching, the
solution is usually extremely simple. This rule is true here, except there is some complexity to the technical
underpinnings of the problem, and it highlighted the complexities of concurrency and how, as with Python's
Global Interpreter Lock, it sometimes helps to make demarcations of when and when not to use it.
