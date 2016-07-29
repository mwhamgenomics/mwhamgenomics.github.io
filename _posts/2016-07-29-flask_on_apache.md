---
layout: post
title: Building a web app part 4 - Running multiple Flask apps on Apache
category: programming
tags: ['web_development', 'reporting_app']
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

After running our Reporting App for some time on a home-spun setup combining Tornado with Linux forking and Bash scripts, we decided to move to an Apache-based stack as a more professional alternative. This would allow us to have a constantly-running, autoreloading web app without having to cobble together our own infrastructure, and would also allow us to run on a default web browser port, meaning we wouldn't have to worry about adding the port number to the url when visiting it.

Apache is one of the most popular HTTP servers on the internet, so is very full-featured and flexible to configure. However, its features and flexibility mean that it can be a bit tricky to set up. Another interesting problem also comes up at this point. Our Reporting App consists of a Flask web app mounted at the root level of the web server (e.g. localhost:5000/), and a Rest API mounted on a different port with a url prefix (e.g. localhost:5001/api/v1/). Here, however, we want both apps to run on port 80 and visit, in this example, `localhost/` for the web app and `localhost/api/v1/` for the Rest API. This means we need some way of multiplexing the two apps onto the same port, with the Apache server deciding whether to serve the web app or the Rest API, depending on which url is visited.

Here, then are my experiences in setting up multiple Flask apps side by side via mod_wsgi on an Apache server.

### Installing Apache and mod_wsgi
Apache is listed in major Linux package managers, known by various names, including 'Apache2' and 'httpd'. Installation on CentOS goes something like:

    # yum install httpd

This should both download and start the `httpd` service. Navigate to your hosting server (you don't need a port number because by default Apache runs on port 80, the default port that web browsers visit). and you should see a placeholder web page. This is an HTML page being served (on CentOS 7, at least) by an Apache configuration called `welcome.conf`, which can be found in `/etc/httpd/conf.d`. `conf.d` is the main folder you'll be working in for configuring Apache. `/etc/httpd/conf` contains the main `httpd.conf` config file, but you shouldn't normally need to edit it.

Firstly, however, we need mod_wsgi. This is the module that allows WSGI-compatible web apps to run on Apache. At first, I tried installing it centrally:

    # yum install httpd-mod_wsgi

This creates a `mod_wsgi.so` file in `/etc/httpd/modules` and a config file in `/etc/httpd/conf.modules.d`:

10-mod_wsgi.conf:

    LoadModule wsgi_module /etc/httpd/modules/mod_wsgi.so


### Configuring Apache
When Apache starts up, it scans its `/etc` folder for config files and loads them. To add a configuration for our two apps, then, we just need to add a new `.conf` file.

/etc/httpd/conf.d/app.conf:

    <VirtualHost \*:80>
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

Here, we create a VirtualHost on port 80. A VirtualHost allows you to manage the running of multiple websites on one Linux server - probably not needed in this case, but from what I've seen, it's generally good practice to use them. ServerName should be the url or IP address of the machine running the webserver, unless you have a DNS system that routes the url to your server. We now define the location of two WSGI scripts, one for the Rest API and one for the web app, and register them to the paths `/api/v1` and `/` respectively. The order in which these are declared is important - it's important to register `/api/v1` as the Rest API first, or `/api/v1` will be seen as part of the web app registered to the root (`/`). Finally, we define a Directory allowing the server to find the `static` and `template` files required by the web app (the Rest API doesn't need any of this since it serves plain text).


### WSGI scripts
We've pointed mod_wsgi to the locations of some `.wsgi` scripts, but we haven't written them yet! In the same way we had scripts to set up our Flask apps for Tornado, these `.wsgi` scripts will do something similar for mod_wsgi. It's common practice to put web content in `/var/www`, so let's set up a directory structure there:

{% highlight bash %}
/var/www/Web-App/
    config.yaml
    wsgi/
        rest_api.wsgi
        web_app.wsgi
    app/
        <versions>  # tagged versions of the Reporting App codebase
        production/  # symlink to current production codebase

{% endhighlight %}

Now let's write a WSGI script:

rest_api.wsgi
{% highlight python %}
top_level = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
config_file = os.path.join(top_level, 'config.yaml')
production = os.path.join(top_level, 'app', 'production')
os.environ['CONFIGFILE'] = config_file  # the Rest API and web app use an environment variable to find their config file
sys.path.append(production)  # make sure Python can import the app

from config import cfg
from rest_api import app as application

# set up some logging...
{% endhighlight %}

`rest_api.app`, in this case, is the Eve/Flask Rest API application. It's important, however, that we import it as `application` because mod_wsgi loads up this script, looks for something called `application` and loads it as the main bit of Python code to port to Apache.

### Troubleshooting
It would have been nice if everything had worked as is, but unfortunately I encountered some problems.


#### Which Python?
I wanted to set up the web app with a virtualenv (in any case, the app was written for Python 3, and CentOS' default is Python 2), but nowhere in the documentation for mod_wsgi could I find a way of pointing it at the right Python interpreter. When I tried to run the app (as below), I got a series of cryptic error messages suggesting it was still trying to run the app with Python 2. After looking at the stacktraces, it appeared that something was a bit amiss regarding the original mod_wsgi install.

Enter the Python Packaging Index. Here we found an entry for mod_wsgi - a bit strange, as mod_wsgi is C, not Python. The reason it's there, though, is made clear on mod_wsgi's [PyPi page](https://pypi.python.org/pypi/mod_wsgi):

    Installation of mod_wsgi can now be performed in one of two ways. The first way of installing mod_wsgi is the traditional way that has been used in the past, where it is installed as a module directly into your Apache installation. The second and newest way of installing mod_wsgi is to install it as a Python package into your Python installation.

It turns out that, at the point of installing mod_wsgi, it is compiled against a given Python interpreter. When installing it centrally via Linux's package manager, this will be the central Python install. Install it as a Python package, however, and it will be compiled against whichever interpreter you're using. This, then is how to point mod_wsgi at the correct version of Python - it's determined at compile-time.

Running `yum install mod_wsgi` will create a mod_wsgi `.so` file in the Python interpreter's `site-packages`, and will also create a new executable, `mod_wsgi-express`. When run, this sets up a complete isolated, automatically configured instance of Apache and runs it from the terminal - very useful for testing and debugging!

Now, however, we need to point to the correct mod_wsgi in the central Apache install. To do this, we need to alter mod_wsgi's config file:

/etc/httpd/conf.modules.d/10-mod_wsgi.conf
{% highlight bash %}
# LoadModule wsgi_module '/etc/httpd/modules/mod_wsgi.so'  # main mod_wsgi installed via Yum - points to central Python install, so comment this out
LoadModule wsgi_module '/path/to/python/virtualenv/site-packages/mod_wsgi/server/mod_wsgi-py34.cpython-34m.so'  # virtualenv's mod_wsgi - load this one instead
{% endhighlight %}

`mod_wsgi` should now be running the Flask apps with the correct Python interpreter.

#### Static libraries
When setting up the central Python interpreter to create virtualenvs from, you may need to enable shared libraries, which you must then point to whenever you run it or any virtualenvs created from it. Setting LD_LIBRARY_PATH in your BashRC is simple enough, but you may also need to set it in the Apache config. At the top of `app.conf`:

    SetEnv LD_LIBRARY_PATH /path/to/Python/lib
    WSGIPythonHome /path/to/python/virtual_env

As you can see, you may need to set a WSGIPythonHome as well so that mod_wsgi can call the Python interpreter and find its libraries.


#### Other
- If one or more or your apps implements authentication server-side, you may find some strange things happening on Apache. At first, for example, I was sending HTTP requests to the web app with some Auth headers, but in the logs I was seeing the app complaining that I supplied no credentials! It turns out that this is a result of a mod_wsgi config: by default, it doesn't pass HTTP Auth headers to the Python app. To allow this, simply add `WSGIPassAuthorization On` to your Apache config. Note, however, that mod_wsgi's b for a reason, namely that it allows all your WSGI apps to see the Auth headers

- All of this was done on the CentOS 7 distribution of Linux. Depending on your distribution and version, files may have different names and/or locations. The gist of it, though, remains the same.
