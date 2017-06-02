---
layout: post
title: Building a web app part 4 - Running multiple Flask apps on Apache
category: Programming
tags: ['web development', 'reporting app']
external_links:
    - title: Apache
      link: 'http://httpd.apache.org'
      description: Official documentation for Apache
    - title: mod_wsgi
      link: 'http://www.modwsgi.org'
      description: Website for the mod_wsgi Apache plugin, developed by Graham Dumpleton
    - title: Deploy Django on Apache with virtualenv and mod_wsgi
      link: 'http://thecodeship.com/deployment/deploy-django-apache-virtualenv-and-mod_wsgi'
      description: This post on Ayman Farhat's site 'The Code Ship' was a useful starting point for setting up a WSGI-compatible web app on Apache.
    - title: Running a Flask app under Apache WSGI
      link: 'http://www.jakowicz.com/flask-apache-wsgi'
      description: Check this post for another useful, more Flask-centric look at mod_wsgi.
---

After running our reporting app for some time on a home-spun setup combining Tornado with Linux forking and Bash scripts, we decided to move to an Apache-based stack as a more professional alternative. This would allow us to run the web app as a service rather than attached to a user. We would also be able to use Apache to multiplex our Rest API and web app onto one port. Furthermore if we run this on port 80, we wouldn't have to worry about adding the port number to the url when visiting it, because port 80 is the default port that web browsers visit.

Apache is one of the most popular HTTP servers on the internet. It's very full-featured and flexible to configure, however it can be a bit tricky to set up. Running multiple apps on a single port especially would require some tricky configuration, because Apache would have to be able to decide whether to serve the web app or the Rest API, depending on which url is visited.

Here, then, are my experiences in setting up multiple Flask apps side by side via mod_wsgi on an Apache server. All of this was done on the CentOS 7 distribution of Linux. Depending on your OS, files may have different names and/or locations, but the gist of it remains the same.

## Installing Apache and mod_wsgi
Apache is listed in most major Linux package managers as various names, including 'Apache2' and 'httpd'. On CentOS:

    # yum install httpd

This should both install and start the `httpd` service. Navigate to your hosting server (you don't need a port number because by default Apache runs on port 80, the port that web browsers visit for HTTP) and you should see a placeholder web page. This is an HTML page served according to an Apache configuration called `welcome.conf`, which can be found in `/etc/httpd/conf.d`. `conf.d` is the main folder you'll be working in for configuring Apache. `/etc/httpd/conf` contains the main `httpd.conf` config file, but you shouldn't normally need to edit it.

Firstly, however, we need mod_wsgi. This is the module that allows WSGI-compatible web apps to run on Apache. At first, I tried installing it centrally:

    # yum install httpd-mod_wsgi

This creates a `mod_wsgi.so` file in `/etc/httpd/modules` and a config file in `/etc/httpd/conf.modules.d`, which looks something like:

    LoadModule wsgi_module /etc/httpd/modules/mod_wsgi.so


## Configuring Apache
When Apache starts up, it scans its `/etc` folder for config files and loads them. To add a configuration for our two apps, then, we just need to add a new `.conf` file.

/etc/httpd/conf.d/app.conf:

    <VirtualHost *:80>
        ServerName <url of hosting server>
        WSGIDaemonProcess reporting_app

        WSGIScriptAlias /api/v1 /path/to/rest_api.wsgi
        WSGIScriptAlias / /path/to/web_app.wsgi

        <Directory /path/to/code/for/web_app>
            AllowOverride None
            WSGIProcessGroup web_app
            WSGIApplicationGroup %{GLOBAL}
            WSGIScriptReloading On
            Order allow,deny
            Allow from all
        </Directory>
    </VirtualHost>

Here, we create a VirtualHost on port 80. A VirtualHost allows you to manage the running of multiple websites on one Linux server - probably not needed in this case, but it's generally good practice to use them. ServerName should be the url or IP address of the machine running the webserver, unless you have a DNS system that routes the url to your server. We now define the locations of two WSGI scripts (one for the Rest API and one for the web app) and register them to the paths `/api/v1` and `/` respectively. The order in which these are declared is important - `/api/v1` should be registered first or it will be seen as part of the web app registered to the root (`/`), and visiting `/api/v1` will not result in a request for the Rest API. Finally, we define a Directory allowing the server to find the `static` and `template` files required by the web app (the Rest API doesn't need any of this since it serves plain text).


## WSGI scripts
We've pointed mod_wsgi to the locations of some `.wsgi` scripts, but we haven't written them yet! In the same way we had scripts to set up our Flask apps for Tornado, these `.wsgi` scripts will do something similar for mod_wsgi. It's common practice to put web content in `/var/www`, so let's set up a directory structure there:

{% highlight bash %}
/var/www/Web-App/
    config.yaml
    wsgi/
        rest_api.wsgi
        web_app.wsgi
    app/
        <versions>  # tagged versions of the reporting app codebase
        production/  # symlink to current production codebase

{% endhighlight %}

Now let's write a WSGI script:

{% highlight python %}
top_level = '/var/www/Web-App'
config_file = os.path.join(top_level, 'config.yaml')
production = os.path.join(top_level, 'app', 'production')
os.environ['CONFIGFILE'] = config_file  # the Rest API and web app use an environment variable to find their config file
sys.path.append(production)  # make sure Python can import the app

from config import cfg
from rest_api import app as application

# set up some logging...
{% endhighlight %}

`rest_api.app`, in this case, is the Eve/Flask Rest API application. It's important, however, that we import it as `application` because mod_wsgi loads up this script, looks for something called `application` and loads it as the main bit of Python code to port to Apache.

## Troubleshooting
It would have been nice if everything had worked as is, but unfortunately I encountered some problems.


### Which Python?
I needed to set up a Python 3 virtualenv, but nowhere in the documentation for mod_wsgi could I find a way of pointing it at the right Python interpreter. When I tried to run the app (as below), I got a series of cryptic error messages suggesting it was still trying to run the app with Python 2. After looking at the stacktraces, it appeared that something was amiss regarding the original mod_wsgi install.

Enter the Python Packaging Index. Here we found an entry for mod_wsgi - a bit strange, as mod_wsgi is C, not Python. The reason it's there, though, is made clear on mod_wsgi's [PyPi page](https://pypi.python.org/pypi/mod_wsgi):

<blockquote>
    Installation of mod_wsgi can now be performed in one of two ways. The first way of installing mod_wsgi is the traditional way that has been used in the past, where it is installed as a module directly into your Apache installation. The second and newest way of installing mod_wsgi is to install it as a Python package into your Python installation.
</blockquote>

When mod_wsgi is installed, it is compiled against a given Python interpreter. When installing it centrally via Linux's package manager, this will be the central Python install. Install it as a Python package, however, and it will be compiled against whichever interpreter you're using at the time. This, then is how to point mod_wsgi at the correct version of Python - it's determined at compile-time.

Running `pip install mod_wsgi` will create a mod_wsgi `.so` file in the Python interpreter's `site-packages`, and will also create a new executable, `mod_wsgi-express`. When run, this actually sets up a complete, isolated and automatically configured instance of Apache, and runs it from the terminal - very useful for testing and debugging!

Now that we can install the mod_wsgi `.so` to a virtualenv, we now just need to point Apache to it. To do this, we need to alter Apache's mod_wsgi config file:

/etc/httpd/conf.modules.d/10-mod_wsgi.conf
{% highlight bash %}
# main mod_wsgi installed via Yum - points to central Python install, so comment this out
# LoadModule wsgi_module '/etc/httpd/modules/mod_wsgi.so'

# virtualenv's mod_wsgi - load this one instead
LoadModule wsgi_module '/path/to/python/virtualenv/site-packages/mod_wsgi/server/mod_wsgi-py34.cpython-34m.so'  
{% endhighlight %}

Apache should now read this config file and load up the correct `mod_wsgi` and Python interpreter.

### Static libraries
When setting up the central Python interpreter to create virtualenvs from, you may need to enable shared libraries, which you must then point to whenever you run it or any virtualenvs created from it. Setting LD_LIBRARY_PATH in your BashRC is simple enough, but you may also need to set it in the Apache config. At the top of `app.conf`:

    SetEnv LD_LIBRARY_PATH /path/to/Python/lib
    WSGIPythonHome /path/to/python/virtual_env

As you can see, you may need to set a WSGIPythonHome as well so that mod_wsgi can call the Python interpreter and find its libraries.


### Authentication
If your apps implement server-side authentication, you may find some strange things happening on Apache. Initially, I was sending HTTP requests to the web app with some Auth headers, but in the logs I was seeing the app complaining that I supplied no credentials! It turns out that this is a result of a mod_wsgi config: by default, it doesn't pass on HTTP Auth headers to the Python app underneath. To allow this, simply add `WSGIPassAuthorization On` to your Apache config.

Note, however, that mod_wsgi has this default behaviour for a reason - otherwise, it allows all your WSGI apps to see all Auth headers. Not such a concern here, but it could be a problem if you have many unconnected WSGI apps running on the same server - in this case, it would be better to handle authentication through Apache.

### Script reloading
This was very simple in Tornado, but unfortunately isn't in Apache. Apache can be configured to automatically reload the WSGI app whenever the `.wsgi` script is updated (or `touch`-ed). When this happens, the `wsgi` script is re-run and import statements for the web app, submodules, etc are repeated as you'd expect. However, since the modules have already been imported into the Python interpreter, nothing happens, even though the code has changed! In this case, we need to explicitly tell the Python interpreter not only to import a module, but also to reload it. This is perfectly possible through `importlib`:

{% highlight python %}
from importlib import reload

import a_module
reload(a_module)
{% endhighlight %}

However, this reload is not recursive - any submodules within are not reloaded! In this case, the following would be necessary:

{% highlight python %}
import a_module
reload(a_module.submodule_1)
reload(a_module.submodule_2)
reload(a_module)
{% endhighlight %}

If you are wondering why this is the case, my answer is that I don't know, and if you think this is ridiculous, I agree. In any case, our `rest_api.wsgi` now looks something like:

{% highlight python %}
top_level = '/var/www/Web-App'
config_file = os.path.join(top_level, 'config.yaml')
production = os.path.join(top_level, 'app', 'production')
os.environ['CONFIGFILE'] = config_file  # the Rest API and web app use an environment variable to find their config file
sys.path.append(production)  # make sure Python can import the app

import config
import rest_api
reload(config)
reload(rest_api.aggregation)  # reload a submodule
reload(rest_api)  # reload the top-level module

application = rest_api.app
{% endhighlight %}

If your app starts getting complex with multiple modules and endpoints, authentication, etc, you might want to think about moving to a more elaborate platform like Django. For now, though, our app runs well on Flask through Apache and mod_wsgi.
