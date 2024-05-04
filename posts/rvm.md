---
extends: post.html
title: Setting up virtual Ruby environments with RVM
date: 2016-03-24
category: programming
tags: ['environment']
external_links:
    - title: RVM documentation
      link: https://rvm.io
      description: Official site and full documentation for RVM
    - title: virtualenv post
      link: '{{ site.baseurl }}/programming/2016/03/24/virtualenv.html'
      description: See here for a post on virtualenv, Python's equivalent of RVM
---

When I was first building this website, I needed to set up Jekyll so I could build a development instance of this site.
For this, I had to set up a Ruby development environment. If you've seen my post on using
[virtualenv](/programming/2016/03/24/virtualenv.html), you might be getting a bit of déjà vu. Fortunately, as with
Python, there is a proper way of setting up isolated Ruby instances, namely RVM.

RVM works in a similar way to virtualenv, with a few minor differences. Instead of a module that you install to an
interpreter and use to set up environments in a given location, RVM is a single program that lives in your home -
`~/.rvm/`. All Ruby environments created will also go here, and you then call RVM to select one. In the end, however, it
does the same job as virtualenv and solves the same problems.

## Installation
RVM is installed via a [script](https://raw.githubusercontent.com/rvm/rvm/master/binscripts/rvm-installer) found in the
RVM GitHub project (you can also use [get.rvm.io](http://get.rvm.io), which is a redirect). Simply download this script
and run it:

    [mwhamgenomics]$ curl -o rvm_installer.sh https://raw.githubusercontent.com/rvm/rvm/master/binscripts/rvm-installer
    [mwhamgenomics]$ sh rvm_installer.sh stable  # or 'master' for the latest dev version of RVM

At this point, your `.bash_profile` shell config is modified to include the RVM executable and some helper functions. To
get these, either reload your session or source the relevant files:

    [mwhamgenomics]$ source ~/.profile
    [mwhamgenomics]$ source ~/.rvm/scripts/rvm

Now RVM is ready to use. Calling it with no arguments prints usage information, including what actions are available:
- use - set RVM to use an installed Ruby environment
- install - install a version of Ruby
- uninstall - uninstall a version of Ruby
- list known - search for available Ruby versions

Installation of a new Ruby version would go something like:

    [mwhamgenomics]$ rvm list
    (shows installed Ruby versions)
    [mwhamgenomics]$ rvm list known
    (searches for available Ruby versions)
    [mwhamgenomics]$ rvm install 2.3.0
    [mwhamgenomics]$ which ruby
    /home/mwhamgenomics/.rvm/rubies/ruby-2.3.0/bin/ruby
    [mwhamgenomics]$ which gem
    /home/mwhamgenomics/.rvm/rubies/ruby-2.3.0/bin/gem

(Note: RVM on OSX requires Homebrew. When using RVM for the first time, you may be asked to install it. As far as Jekyll
is concerned, you can specify an isolated non-root location for Homebrew, but this is not recommended for general
Homebrew use)


## Gemsets
All RVM has done up to this point is manage Ruby versions. What about managing all your Ruby Gems per project? RVM does
this via gemsets.

    [mwhamgenomics]$ rvm gemset list
    (lists all gemsets for the current Ruby environment)
    [mwhamgenomics]$ rvm gemset create <gemset_name>
    [mwhamgenomics]$ rvm gemset use <gemset_name>

This way, you can keep your Gems tied to a project as virtualenv does.
