---
extends: post.html
title: Building a web app part 2 - Flask front end
date: 2016-07-15 16:00:00 
category: programming
tags: ['python', 'web development', 'reporting app']
external_links:
    - title: Official Flask documentation.
      link: 'http://flask.pocoo.org'
---

The main focus of the stack in Edinburgh Genomics' reporting app is the Rest API/database back end (both in terms of
both rate of development and lines of code) rather than the front end. This post, then, could turn out to be quite a
short one. There's little point in rehashing Flask's documentation, so instead I will briefly describe how we set up a
simple web app with template rendering .

The basics of the app's setup is almost exactly the same as the introductory section of Flask's documentation:

```python
import flask
app = flask.Flask(__name__)

@app.route('/')
def main_page():
    return 'This is a main page'
```

Visit the top level of this app in a web browser and you will see the text 'This is a main page'. Of course, this is not
very useful. We could have the function return html content, but it would be a hugely horrible function if we tried to
render a full html-formatted web page. If we want to do that, it's best to use templates.

## HTML templates

Flask's template rendering is done via Jinja2, and allows you to render dynamic html pages:

```python
@app.route('/')
def main_page():
    return flask.render_template('main_page.html')
```


```html
<!-- templates/main_page.html: -->
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>Main page</title>
    </head>

    <body>
        <p>This is a main page</p>
    </body>
</html>
```

Of course we don't want to have to write all of this boilerplate for every page we want. Ideally we would structure it
like classes - a base template, with sub-templates that inherit from it. Let's add another page, and add some Jinja2
syntax to the templates.


```jinja
<!-- templates/base.html: -->
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
```

```jinja
<!-- templates/main_page.html: -->
{% extends 'base.html' %}
{% block title %}Main page{% endblock %}
{% block content %}
    {{ super() }}
    <p>This is a main page</p>
{% endblock %}
```

```jinja
<!-- templates/second_page.html: -->
{% extends 'base.html' %}
{% block title %}Another page{% endblock %}
{% block content %}
    <p>This is a main page</p>
    {{ text }}
{% endblock %}
```

Here, base.html contains a default <head>, allowing css styling to be included in all sub-templates. Each sub-template
implements a `title` block, which will be inserted into the `<title>` element of the base template. `main_page` will
have the base's `content` block plus some of its own stuff, and `second_page` completely overrides `content`. Text can
be added into `second_page` by Flask with `flask.render_template('second_page.html', text='Some text to add')`.

## User input
While visiting websites, you may have noticed that the URL of the page sometimes has a `?` followed by a series of `x=y`
key-value pairs. This is the query string, and is accessible to Flask via `flask.request`:

```python
@app.route('/second_page')
def second_page():
    return flask.render_template('second_page.html', text=flask.request.args.get('input_text'))
```

If this endpoint is visited, e.g. via the url `localhost:5000/second_page?input_text=thisthatother`, the string
`thisthatother` will be passed via flask.request.args to the template and rendered on the web page. Not too useful in
this case, but very useful for instances where data needs to be passed from the client to the server.
