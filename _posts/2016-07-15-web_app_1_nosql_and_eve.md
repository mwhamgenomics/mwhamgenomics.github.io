---
layout: post
title: Building a web app part 1 - NoSQL and Eve
category: Programming
tags: ['python', 'web development', 'reporting app']
external_links:
    - title: MongoDB documentation
      link: 'https://docs.mongodb.com'
      description: Official MongoDB website.
    - title: Eve documentation
      link: 'http://python-eve.org'
      description: Documentation for the many features of Eve.
---

Behind the scenes, the reporting app we developed at Edinburgh Genomics is powered by MongoDB and Eve. MongoDB, being a NoSQL database, stores JSON instead of tables, as in SQL databases. This makes it much more free-form than SQL, and allows for nesting of data structures, although the way we have been using it, there's not actually that much difference from SQL.

We could have had the front end website query the database directly, however this would have made things complicated for other parts of our stack. Our pipelines and other scripts all push data to this database, so all of our programs would have to load a MongoDB client, set up a connection and push data, hopefully in the correct format. Because we had so many things using this database, and because we still wanted some kind of schema, we decided we wanted to interact with the database through a Rest API.

Eve is a Flask-based Rest API library. This gave us an HTTP-based interface to the database and a very flexible schema for data flowing in. This allowed us to validate the data being stored while also being able to add new types of data over time without migrating the database.

## Eve
Being an extension of Flask, Eve is fairly familiar to install and configure:

{% highlight python %}

import eve

settings = {
    'DOMAIN': {
        'an_endpoint': {
            'url': 'an_endpoint',
            'item_title': 'thing',
            'schema': {
                'a_field': {'type': 'string', 'required': True, 'unique': True},
                'another_field': {'type': 'integer', 'required': False, 'min': 1, 'max': 8 }
                'yet_another_field': {'type': 'list', 'required': False, 'schema': {'type': 'string'}}
            }
        },
        'MONGO_HOST': 'mongodb://localhost',
        'MONGO_PORT': 4888,
        'MONGO_DBNAME': 'reporting_app',
        'URL_PREFIX': 'api',
        'API_VERSION': '0.1',
        'XML': False
    }
}

app = eve.Eve(settings=settings)

if __name__ == '__main__':
    app.run('localhost', 4889)
{% endhighlight %}

Here, eve is configured through a dict, mainly through the `DOMAIN` field. We tell the app how to connect to a running instance of MongoDB, define a schema and run it locally on a given port. All we need now is a running database. With MongoDB, this is fairly easy. After downloading the executable binaries:

    path/to/mongod --port 4888 --dbpath path/to/reporting_app_db

Should you wish more control over things like logging and the storage engine, the documentation for MongoDB has an [extensive section](https://docs.mongodb.com/manual/reference/configuration-options) on configuring the server from a Yaml config file. However the database is configured, once it and the Eve app are running, you should now be able to go to 'http://localhost:4889/api/0.1' in a web browser and see a summary of all the endpoints in the database:

{% highlight json %}
{
    "_links": {
        "child": [
            {"title": "an_endpoint", "href": "an_endpoint"}
        ]
    }
}
{% endhighlight %}

And upon visiting 'http://localhost:4889/api/0.1/an_endpoint', you should see some content:

{% highlight json %}
{
    "data": [],
    "_links": {
        "self": {
            "href": "an_endpoint",
            "title": "an_endpoint"
        },
        "parent": {
            "href": "/", "title": "home"
        }
    },
    "_meta": {
        "page": 1,
        "max_results": 25,
        "total": 0
    }
}
{% endhighlight %}

Of course there's nothing in the database yet, so there won't actually be anything in the `data` field. Let's rectify that.

{% highlight python %}
>>> import requests
>>> new_data = {'a_field': 'some text', 'another_field': 6, 'yet_another_field': ['this', 'that', 'other']}
>>> requests.post('localhost:4889/api/0.1/an_endpoint', new_data)
{'_status': 'OK', '_etag': '<random_string>', '_id': '<random_string>', '_updated': '<a_date_time>', '_created': '<a_date_time>', '_links': {'self': {'title': 'run', 'href': 'runs/<_id>'}}}
{% endhighlight %}

The schema defined above controls what can enter the database. `a_field` has to be a string, `another_field` has to be an integer between 1 and 8, and `yet_another_field` has to be a list of strings. If you try and push some dodgy data, you should see an error message:

{% highlight python %}
>>> requests.post('localhost:4889/api/0.1/an_endpoint', {'some': 'dodgy data'})
{'_status': 'ERR', '_error': {'message': 'Insertion failure: 1 document(s) contain(s) error(s)', 'code': 422}, '_issues': {'some': 'unknown field', 'a_field': 'required field'}}
{% endhighlight %}

Having pushed some data, we should be able to visit the endpoint and see something like:
{% highlight json %}
{
    "data": [
        {"a_field": "some text", "another_field": 6, "yet_another_field": ["this", "that", "other"]},
    ],
    "_links": {
        "self": {
            "href": "an_endpoint",
            "title": "an_endpoint"
        },
        "next": {
            "href": "an_endpoint?max_results=25&page=2",
            "title": "next page"
        },
        "parent": {
            "href": "/", "title": "home"
        },
        "last": {
            "href": "an_endpoint?max_results=25&page=3",
            "title": "last page"
        }
    },
    "_meta": {
        "page": 1,
        "max_results": 25,
        "total": 75
    }
}
{% endhighlight %}

Now things make a little more sense. The information you have pushed to the database is now stored in `data` as a list of dicts. Eve adds some metadata to the content, such as how much data is in the endpoint, as well as some information on pagination. If there's too much data for one page, you can visit the next page by visiting `http://localhost:4889/api/0.1/an_endpoint?max_results=25&page=2`. Eve's [documentation](http://python-eve.org) contains complete information on Eve's capabilities.

## Simple aggregation
Eve is certainly not lacking for features. It's possible to filter, sort, limit which JSON fields are returned, control pagination, and embed data from other endpoints, all of which is useful to the web app front end. However, the one thing that Eve doesn't really do (as of version 0.6) is aggregation.

At first, we were able to use Eve's event hook system. For a given endpoint, it is possible to register a function to the app to modify the data being returned on the fly. In our case, we could register a function that would add calculated fields to each JSON element being returned:

{% highlight python %}
app = eve.Eve(settings)

def add_calculated_field(request, response):
    for element in response.data:
        # modify each element of response.data
        element['an_aggregated_field'] = len(element['yet_another_field']) + element['another_field']

app.on_post_GET_an_endpoint += add_calculated_field  # register the function
{% endhighlight %}

This allows us to aggregate on the fly rather than store aggregated data, which was something that we didn't want to do. However, because these fields were being added after filtering, it was not possible to filter data based on calculated fields. To do that, we need to use MongoDB's own aggregation, which has an extensive range of logic/aggregation functions, allowing you to pass returned data through a pipeline.

## Pipeline aggregation
It is possible to aggregate in PyMongo or the MongoDB shell with a `list[dict]` pipeline. Here's a relatively simple example:

{% highlight python %}
import pymongo
cli = pymongo.MongoClient('localhost', 4888)
db = cli['reporting_app_db']
endpoint = db['an_endpoint']

pipeline = [
    {
        '$project': {
            'a_field': '$a_field',
            'another_field': '$another_field'
            'yet_another_field': '$yet_another_field',
            'a_calculated_field': {'$add': ['$another_field', 2]}
        }
    }
]

endpoint.aggregate(pipeline)
{% endhighlight %}

Of course, it's not as simple as that. This is outside of Eve's filtering and pagination functionality. In this case, we need to this functionality back in ourselves. To filter depending on `a_field`:

{% highlight python %}
pipeline = [
    {
        '$project': {
            'a_field': '$a_field',
            'another_field': '$another_field'
            'yet_another_field': '$yet_another_field',
            'a_calculated_field': {'$add': ['$another_field', 2]}
        }
    },
    {
        '$match': {
            'a_calculated_field': 4
        }
    }
]

endpoint.aggregate(pipeline)
{% endhighlight %}

This will allow us to filter statically on a calculated field. The challenge now is to pass it through Eve. You can register a Flask endpoint (since Eve is Flask) linked to this pipeline, and if the pipeline listens to `flask.request`, you could visit, e.g, `http://localhost:4889/api/0.1/pipeline_aggregation?a_calculated_field=4`. The final challenge, in this case, is making the pipeline fully dynamic. If you are filtering on something that's already in the schema, you may as well do the `$match` at the start of the pipeline to minimise the calculations required, and if you're filtering on a calculated field, you'll want to calculate it first and have the `$match` after. Current versions of the Reporting App use mostly-dynamic aggregation pipelines, but this remains an interesting problem to solve and something that we're still working on.

Apart from aggregation, for which we've rolled our own solution, Eve has allowed us, with relatively little effort, to set up a database and Rest API back-end for our Reporting App. Being Flask, we were able to run it in an identical way to our front end. Eve is fairly easy to configure, full-featured, and has a very flexible and lenient schema system, allowing us to work with a rapidly-developing app and database, even in production.
