---
layout: post
title: Setting up virtual Python environments with virtualenv
category: Programming
tags: ['Python']
external_links:
    - title: Virtualenv documentation
      link: https://virtualenv.pypa.io
      description: Full documentation for virtualenv
    - title: RVM post
      link: /programming/2016/03/24/rvm.html
      description: See here for a post on RVM, Ruby's equivalent of virtualenv
---

When I first started developing software in Python in my first job, I quickly ran into a problem that many people new to Python may ask, namely how to keep a consistent programming environment for a given project. How can I be sure which version of Python I'm on, and how can I keep track of the third-party modules I'm importing?

Firstly, let's rule out installing everything on the machine's central Python instance. This is a bad idea for various reasons. Firstly, you might not have the right user permissions that machine. Secondly, what if your software is stable and you just want to let it sit there doing its job? If it's stable with Flask 0.10, then you don't particularly want someone to come along and update Flask to 0.11 because they need it for something else. Thirdly, the central Python install could be used by something important, and if you fiddle with it you can end up breaking things. The [Python docs](https://docs.python.org/3.4/using/mac.html) specifically warn against modifying the central Python install on Mac OSX.

It's perfectly possible to use `make altinstall` during setup, and have `python2` and `python3` executables side by side in `/usr/bin`, however what if you want to distinguish between Python 3.4 and 3.5?

Keeping track of modules per project also has its challenges. You might keep a folder of modules for each project you have, and then point to the right one using `PYTHONPATH`. This, however, is quite a lot of work. Rather than going through Python's `pip` package manager, you're building each module manually. There is also nothing linking the modules to the Python version - you could end up trying to import Python 3 modules into Python 2.

Fortunately, there's a proper way of doing it all: virtualenv.

### virtualenv
Virtualenv is a Python module that allows you to set up isolated instances of Python, each with their own executables and libraries. Simply install it on one Python instance, and use that as a 'master' for setting up virtualenvs:

{% highlight bash %}
mwhamgenomics$ pip install virtualenv
{% endhighlight %}

You can now call virtualenv in one of two equivalent ways:
{% highlight bash %}
mwhamgenomics$ virtualenv path/to/new_virtualenv
mwhamgenomics$ python -m virtualenv path/to/new_virtualenv
{% endhighlight %}

This will create an isolated instance of Python in the specified location. To use the new virtualenv, simply call the interpreter or any related executable (pip, etc.) in its `bin` dir. Any modules you install using the virtualenv's pip will be put in its `lib`, and any new executables created (e.g, py.test) will be put in its `bin`.

You can also bind this virtualenv to your shell session by sourcing the `activate` script in `bin`:

{% highlight bash %}
mwhamgenomics$ which python
/usr/bin/python
mwhamgenomics$ source path/to/new_virtualenv/bin/activate
(new_virtualenv) mwhamgenomics$ which python
path/to/new_virtualenv/bin/python
(new_virtualenv) mwhamgenomics$ which pip
path/to/new_virtualenv/bin/pip
{% endhighlight %}

You will notice that it modifies your command line with the virtualenv's name in brackets. You can also deactivate the virtualenv like so:

{% highlight bash %}
mwhamgenomics$ deactivate
{% endhighlight %}

In production, you would deploy your program with a virtualenv sitting somewhere alongside, with any necessary modules for running the program. By using isolated virtual Python instances, you can link a distinct Python instance to each of your projects, keeping your environment predictable, the machine clean, and the project insulated against system changes.
