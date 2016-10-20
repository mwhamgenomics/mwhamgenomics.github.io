---
layout: post
title: Building a web app part 2 - Flask front end
category: programming
tags: ['python', 'web_development', 'reporting_app']
external_links:
    - title: Official Flask documentation.
      link: 'http://flask.pocoo.org'
---

Since the main focus of the Edinburgh Genomics Reporting App is its back end (in terms of both rate of development and lines of code), this could turn out to be quite a short post. There's little point in rehashing Flask's documentation, so I will attempt to focus on the problems that we specifically have faced and solved.

The basics of the app's setup is almost exactly the same as the introductory section of Flask's documentation:

{% highlight python %}
import flask
app = flask.Flask(__name__)

@app.route('/')
def main_page():
    return 'This is a main page'
{% endhighlight %}

Visit the top level of this app in a web browser and you will see the text 'This is a main page'. Of course, this is not very useful. We could have the function return html content, but it would be a hugely horrible function if we want to render a full formatted web page. If we want to do that, it's best to use templates.

### HTML templates

Flask's template rendering is done via Jinja2, and allows you to render dynamic html pages:

{% highlight python %}
@app.route('/')
def main_page():
    return flask.render_template('main_page.html')
{% endhighlight %}

templates/main_page.html:
{% highlight html %}{% raw %}
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>Main page</title>
    </head>

    <body>
        <p>This is a main page</p>
    </body>
</html>
{% endraw %}{% endhighlight %}

Of course we don't want to have to do this for every page we want. Ideally we would structure it like classes - a base template, with sub-templates that inherit from it. Let's add another page, and add some Jinja2 syntax to the templates.

templates/base.html:
{% highlight html %}{% raw %}
<!DOCTYPE html>
<html lang="en">
    <head>
        {% block head %}
        <!-- some css styling -->
        <title>{% block title %}{% endblock %} - web page</title>
        {% endblock %}
    </head>

    <body>
        {% block content %}
        <p>This is a website</p>
        {% endblock %}
    </body>
</html>
{% endraw %}{% endhighlight %}

templates/main_page.html:
{% highlight html %}{% raw %}
{% extends 'base.html' %}
{% block title %}Main page{% endblock %}
{% block content %}
    {{ super() }}
    <p>This is a main page</p>
{% endblock %}
{% endraw %}{% endhighlight %}

templates/second_page.html:
{% highlight html %}{% raw %}
{% extends 'base.html' %}
{% block title %}Another page{% endblock %}
{% block content %}
    <p>This is a main page</p>
    {{ text }}
{% endblock %}
{% endraw %}{% endhighlight %}

Here, base.html contains a default <head>, allowing css styling to be included in all sub-templates. Each sub-template implements a `title` block, which will be inserted into the `<title>` element of the base template. `main_page` will have the base's `content` block plus some of its own stuff, and `second_page` completely overrides `content`. Text can be added into `second_page` by Flask with `flask.render_template('second_page.html', text='Some text to add')`.

### User input
While visiting websites, you may have noticed that the URL of the page sometimes has a `?` followed by a series of `x=y` key-value pairs. This is the query string, and is a useful way passing data from the client to the server. This is accessible to Flask via `flask.request`:

{% highlight python %}
# localhost:5000/second_page?input_text=thisthatother
@app.route('/second_page')
def second_page():
    return flask.render_template('second_page.html', text=flask.request.args.get('input_text'))
{% endhighlight %}
