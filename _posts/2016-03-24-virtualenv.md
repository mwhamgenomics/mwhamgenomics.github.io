---
layout: post
title: Setting up virtual Python environments with virtualenv
category: programming
tags: ['python']
external_links:
    - title: Virtualenv documentation
      link: https://virtualenv.pypa.io
      description: Full documentation for virtualenv
    - title: RVM post
      link: /programming/2016/03/24/rvm.html
      description: See here for a post on RVM, Ruby's equivalent of virtualenv
---

When I first started doing proper software development in Python in my first job, I quickly ran into a problem that many people new to Python may ask, namely how to keep a consistent programming environment for a given project. How can I be sure which version of Python I'm on, and how can I keep track of the third-party modules I'm importing?

Firstly to distinguish between Python 2 and 3, it is perfectly possible to have `python2` and `python3` executables side by side in `/usr/bin`. However, what if you want to use Python 3.4 for one project and 3.5 for another? What if the central Python install is used by something else? If you overwrite it, you can end up breaking things (the [Python docs](https://docs.python.org/3.4/using/mac.html) specifically warn against modifying the central Python install on Mac OSX).

Keeping track of modules per project also has its challenges. You might keep a folder of compiled/built modules for each project you have, and then point to the right one using `PYTHONPATH`. This, however, is quite a lot of work. Rather than going through Python's `pip` package manager, you're building each module manually. There is also nothing linking the modules to the Python version - you could end up using Python 3 and importing modules compiled for Python 2.

As an alternative, you might even decide to make a Python developer really cringe and install everything on the machine's central Python instance. This is not a good idea for various reasons. Firstly, you might not have permission to do so on the machine. Secondly, what if your software is stable and you just want to let it sit there? If it's stable with Flask 0.10, then you don't particularly want someone to come along and update Flask to 0.11 without you knowing.

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

This will create an isolated instance of Python in the specified location. To use it, simply call the interpreter or any related executable (pip, etc.) in its `bin` dir. Any modules you install using the virtualenv's pip will be put in its `lib`, and any new executables created (e.g, py.test) will be put in its `bin`.

If you want to bind this virtualenv to your shell session, you can source the `activate` script in `bin`:

{% highlight bash %}
    mwhamgenomics$ which python
    /usr/bin/python  # or wherever the main instance is
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

In production, you would have the virtualenv sitting alongside your project, or in a separate `virtualenvs` folder, and either call `<virtualenv>/bin/python` directly or use `<virtualenv>/bin/activate` if you use `PYTHONPATH`, etc. By using isolated virtual Python instances, you can link a distinct Python instance to each of your projects, keeping your environment predictable, the machine clean, and the project insulated against system changes.
