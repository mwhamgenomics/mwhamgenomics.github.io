---
layout: post
title: Deploying and versioning Python projects
category: Programming
tags: ['python', 'environment']
external_links:
    - title: Pip documentation
      link: 'https://pip.pypa.io'
      description: Guides on pip's many capabilities in installing, uninstalling, updating and listing Python modules.
    - title: Python packaging guide
      link: 'https://packaging.python.org'
      description: More general docs on deploying Python projects in general.
---

For all of Python's simplicity and intuitiveness at the point of development, management and versioning of 3rd-party Python modules can get rather complicated. Given the extensive community-driven ecosystem of the [Python Package Index](https://pypi.python.org/pypi) (or PyPi), it's pretty much inevitable that keeping track of dependencies is going to get a bit tricky.

(Note: this post will not discuss publishing projects to PyPi. It will, however, discuss deploying modules from version control)

## Installing a module to a Python interpreter
During the development of one of the open-source projects for Edinburgh Genomics, I used a fairly clean way of running a command-line app through an executable entry point. You would have a structure like...

    A_Project/
        bin/
            run_app.py
        a_project/
            # some kind of Main method will be here in a client.py or __init__.py file
            a_submodule
        tests/
        requirements.txt

...and in `bin/run_app.py`, you would have something like:

{% highlight python %}
import sys
from os.path import dirname, abspath

sys.path.append(dirname(dirname(abspath(__file__))))
from a_project import main

sys.exit(main())
{% endhighlight %}

This works well if your app isn't going to be imported by anything else. However, there are a couple of problems. Firstly, it involves fiddling around with sys.path, which ideally you wouldn't need to do. Secondly, you need to use the correct Python virtualenv for the app (you are [using virtualenv](/programming/2016/03/24/virtualenv.html), aren't you?). Thirdly, there's no clean way of making a function from this project available to another project.

Edinburgh Genomics has two projects: Analysis-Driver and Reporting-App, both of which declare classes for reading config files, building web URLs and setting up logging. Ideally we'd only have to write shared code like this once and put this into a separate 'core' project, which is imported by our various apps. We decided to do this and call the new module EGCG-Core. The challenge now is making it possible for Analysis-Driver and Reporting-App to import it. You could just deploy the source code thus:

    projects/
        EGCG-Core/
            config/
            app_logging/
            requirements.txt
        Analysis-Driver
        Reporting-App

 Analysis-Driver and Reporting-App could access EGCG-Core if it's on `sys.path` or `PYTHONPATH`. That's one more thing to keep track of, though, and the problem of using the correct Python interpreter remains.

To solve this problem, it is necessary to delve into Python's packaging and deployment features, and write a `setup.py` build/deploy script. You may have come across the command `python setup.py install` while installing modules. While the prospect of writing a build script may sound terrifying, it really isn't. To write a setup.py, it's easiest to look at examples. Let's look at how the setup.py for EGCG-Core looked upon its initial commit:

{% highlight python %}
from distutils.core import setup

setup(
    name='EGCG-Core',
    version='0.1',
    packages=['egcg_core', 'egcg_core.executor', 'egcg_core.executor.script_writer']
    url='< project website >',
    license='MIT',
    description='Shared functionality across EGCG projects',
    requires=['PyYAML', 'pytest', 'requests', 'genologics']
)
{% endhighlight %}

As you can see, all that setup.py does is declare a `setup` function providing metadata about the project. This gets picked up by distutils when you run `python setup.py install`, which deploys the project's code into the Python interpreter's `site-packages`, which is where installed modules live. That's all you really need to get started. However, it's pretty easy to go a little further. After a bit of development, the setup.py for EGCG-Core now looks something like:

{% highlight python %}
from setuptools import setup, find_packages
from os.path import join, abspath, dirname
from egcg_core import __version__
requirements_txt = join(abspath(dirname(__file__)), 'requirements.txt')
requirements = [l.strip() for l in open(requirements_txt) if l and not l.startswith('#')]

def translate_req(req):
    # this>=0.3.2 -> this(>=0.3.2)
    ops = ('<=', '>=', '==', '<', '>', '!=')
    version = None
    for op in ops:
        if op in req:
            req, version = req.split(op)
            version = op + version

    if version:
        req += '(%s)' % version
    return req

setup(
    name='EGCG-Core',
    version=__version__,
    packages=find_packages(exclude=('tests',)),
    url='< project website >',
    license='MIT',
    description='Shared functionality across EGCG projects',
    long_description='< a full description >',
    requires=[translate_req(r) for r in requirements],
    install_requires=requirements
)
{% endhighlight %}

The `setup` function is still there, but with a few differences. Firstly instead of distutils, it uses setuptools, a more full-featured extension of distutils which is also in the standard library. This allows you to specify some additional arguments such as `install_requires`, which can pass information to the installation process and automatically install dependencies. Finally, some logic has been added to reduce the amount of hard-coding in the script. Requirements are obtained from parsing `requirements.txt` (with some formatting changes), `packages=` now goes through `setuptools.find_packages`, and version information is imported from the project itself.

A project with a setup.py like this can be installed via `python setup.py install` or through pip. Upon doing so, your module will be installed to the Python interpreter's `site-packages`, at last giving a strong link between the module and the interpreter.

## Installing via version control
Installing your own module via pip sounds great, but you may have noticed a stumbling point. Pip goes through the Python Package Index! There are two ways around this. The first is to submit your project to PyPi, which is beyond the scope of this post. The second is to use pip's version control support. As of this writing, the latest version of pip supports Git, Mercurial, SVN and Bazaar. All of these have support for installing particular tagged versions, as documented [here](https://pip.pypa.io/en/latest/reference/pip_install/#vcs-support). If a dependency is available on Github, you can point to it thus:

    git+https://github.com/<organisation>/<project_name>.git@<tag_version>

The requirements.txt file for Analysis-Driver, for example, points to a dependency on EGCG-Core via Git, so EGCG-Core should be installed when Analysis-Driver is deployed. You can, of course, install a project via a direct pip command:

    pip install git+https://github.com/<organisation>/<project_name>.git@<tag_version>

## Beware top-level files - the project vs the module
It's important to bear in mind that what you have in version control is not the same as what will be deployed into `site-packages`. In the above example, the versioned project is called EGCG-Core but the module given to `setup` in `packages=` is `egcg_core`. This means that your module cannot depend on any top-level files in the project, because they won't actually be installed. For example, EGCG-Core used to have the following structure:

    EGCG-Core/
        requirements.txt
        version.txt
        egcg_core/
            __init__.py

\_\_init\_\_.py used to set a variable called `__version__` by parsing version.txt. This worked fine when the module was run from source, but not when run as a setup.py installation, as `version.txt` was not installed with `egcg_core`. There are ways of specifying data files to be installed along with the module, but it's far easier and cleaner in this case to just set `__version__` directly in \_\_init\_\_.py. The reason we used a version.txt originally was so we could parse the file for version information both at runtime and at the point of deploying a new Git tag via a versioning script.

    # version.txt
    0.1

{% highlight bash %}
# tag_project.sh
version=$(cat version.txt)
git tag $version
git push --tags
{% endhighlight %}

Setting the version in \_\_init\_\_.py doesn't actually change that much - we just need to parse out the expression that sets `__version__`.

{% highlight python %}
# __init__.py
__version__ = 0.1
{% endhighlight %}

{% highlight bash %}
# tag_project.sh
# note: this won't work if __version__ is set more than once in the project's code
version="$(grep -rh '__version__ = ' $(pwd) | sed 's/\(__version__\) = \(.*\)/\2/' | sed "s/\'//g")"
git tag $version
git push --tags
{% endhighlight %}

## A quick note on version naming
GitHub's cofounder has some notes on semantic versioning [here](http://semver.org). I pretty much use the same naming system for versions.

- dot-separated list of two or three numbers: `<major>.<minor>.<patch>`
- the major version changes very rarely, only when there is a huge update that breaks a lot of old stuff
- the minor version changes when you add a new feature that doesn't break old stuff
- the patch version changes when you add minor updates and fixes
- GitHub recommends prepending 'v' to the version, e.g. 'v0.1.2'
- Most of my projects have their major version as 0. Version 1.0 should be _really_ good, and likely to be useful to other people.
