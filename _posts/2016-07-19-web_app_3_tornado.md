---
layout: post
title: Building a web app part 3 - Production environment on Tornado
category: Programming
tags: ['web development', 'reporting app']
external_links:
    - title: Tornado
      link: 'http://www.tornadoweb.org'
      description: 'Official documentation for the Tornado webserver.'
    - title: PEP 333 - Python Webserver Gateway Interface v1.0.1
      link: 'https://www.python.org/dev/peps/pep-0333'
      description: 'PEP 333 created the WSGI standard, allowing the development of interfaces between HTTP and Python.'
    - title: PEP 3333
      link: 'https://www.python.org/dev/peps/pep-3333'
      description: 'An update of PEP 333 to accommodate changes between Python 2 and 3.'
---

Having developed a database/API/front-end stack for the Edinburgh Genomics reporting app, we had to set it up on a webserver so that people could access it. To do this, we settled on Tornado, an all-Python webserver that allowed us to easily connect up our Flask/Eve apps and set them up with logging and seamless reloading.

## Webserver script
Firstly, we needed a script to start up Tornado with our apps loaded:

{% highlight python %}
# run_app.py:
p = argparse.ArgumentParser()
p.add_argument('app', choices=['reporting_app', 'rest_api'])
p.add_argument('--port', type=int)
args = p.parse_args()
if args.app == 'reporting_app':
    from reporting_app import app, cfg
else:
    from rest_api import app, cfg

# set up the Flask app logger and some of Tornado's internal loggers
loggers = (tornado.log.access_log, tornado.log.app_log, tornado.log.gen_log, app.logger)
# set up the loggers with handlers, formatters, etc...

watched_files = []  # some extra non-code files to watch for reloading, e.g. configs
for f in watched_files:
    tornado.autoreload.watch(f)
tornado.autoreload.start()  # tell Tornado to reload the app when any files are changed

# set up the app inside a WSGIContainer, inside an HTTPServer
container = tornado.wsgi.WSGIContainer(app)
http_server = tornado.httpserver.HTTPServer(container)
http_server.listen(args.port)
tornado.ioloop.IOLoop.instance().start()

{% endhighlight %}

The important part is the last four lines. Tornado works in a similar way to the Apache web server in terms of abstraction. Firstly there is the app, containing the code that we wrote - endpoints, server logic, etc. Around that, there is a WSGI container, which acts as an interface between the Python app and the HTTP server (which deals with HTTP traffic, not Python code - the fact that Tornado is written in Python is a bit of a red herring, which is why I haven't put a 'python' tag on this post). WSGI stands for Web Server Gateway Interface, and came from endeavours to get Python web apps to run on Apache, which will be discussed in the next post.

We can now use this script to run each of our Flask apps, then navigate to the appropriate port on the hosting machine and see the running web app. However, this is not an ideal scenario: by now, you'll have three SSH terminals open, which must stay running for as long as you want the app to be available. It's possible to solve this problem by running the terminals in a Screen session, but a more reliable way is to detach the processes from the terminal. We did this simply by rolling our own. For `rest_api`:

{% highlight bash %}
# rest_api.sh:
case $1 in
    start)
        if [ -n pids/rest_api.* ]
        then
            echo "rest_api already running"
            exit 1
        fi
        nohup path/to/python run_app.py rest_api <port> > /dev/null 2>&1 &
        pid=$!
        echo $pid > pid_files/rest_api.$pid
        echo "Started rest_api with pid $pid"
        ;;
    stop)
        pid=$(echo pid_files/rest_api.* | sed -r 's/.+\.([0-9]+)/\1/')
        if ! [ $pid ]
        then
            echo "rest_api is not running"
            exit 0
        fi
        kill -2 $pid
        rm pid_files/rest_api.*
        ;;
    *)
        echo "Usage: $0 (start|stop)"
        ;;
esac

{% endhighlight %}

When `rest_api.sh start` is run, a command to run the app is executed inside a `nohup &` statement, which detaches the process from external control, allowing it to keep running after logging out. Immediately, it uses `$!` to capture the process' pid and stores it as a lock file (a file which stores information in its filename, e.g. 'rest_api.123456'). `stop` can then use this lock file to find the right process to stop when `rest_api.sh stop` is run.

If we do this for the Rest API, Flask app and NoSQL database, we now have a way of controlling each part of the stack. The use of `tornado.autoreload` also allows us to automatically reload the app whenever we update it. There are probably more refined ways of doing this, but this gave us an easy solution to running our Reporting App stack in production. [Read on](/programming/2016/07/29/flask_on_apache.html) for my adventures in Apache when we decided to move to a full production environment.
