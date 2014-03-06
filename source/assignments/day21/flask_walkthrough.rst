*************************
A Quick Flask Walkthrough
*************************

This walkthrough is designed to give you a quick introduction to the basics of
the Flask microframework.

In it, you'll create a virtualenv and install Flask and its dependencies.  Then
you'll interact a bit with the framework by creating a very simple Flask
application and seeing what it can do.

Setting Up
==========

Start by creating a ``flask_intro`` project and virtualenv:

.. code-block:: bash

    heffalump:cewing cewing$ mkproject flask_intro
    New python executable in flask_intro/bin/python
    Installing setuptools, pip...done.
    Creating /Users/cewing/projects/flask_intro
    Setting project for flask_intro to /Users/cewing/projects/flask_intro
    [flask_intro]
    heffalump:flask_intro cewing$


Finally, install Flask using ``pip``:

.. code-block:: bash

    flask_intro]
    heffalump:flask_intro cewing$ pip install flask
    Downloading/unpacking flask
      Downloading Flask-0.10.1.tar.gz (544kB): 544kB downloaded
      ...
    Successfully installed flask Werkzeug Jinja2 itsdangerous markupsafe
    Cleaning up...
    [flask_intro]
    heffalump:flask_intro cewing$


Kicking the Tires
=================

We've installed the Flask microframework and all of its dependencies.

Now, let's see what it can do.

With your flaskenv active, create a file called ``flask_intro.py`` and open it
in your text editor.


A Flask App
-----------

Getting started with Flask is pretty straightforward. Here's a complete,
simple app.  Type it into `flask_intro.py`:

.. code-block:: python

    from flask import Flask
    app = Flask(__name__)

    @app.route('/')
    def hello_world():
        return 'Hello World!'

    if __name__ == '__main__':
        app.run()


As you might expect by now, the last block in our ``flask_intro.py`` file
allows us to run this as a python program. Save your file, and in your
terminal try this:

.. code-block:: bash

    [flask_intro]
    heffalump:flask_intro cewing$ python flask_intro.py
     * Running on http://127.0.0.1:5000/

Load ``http://localhost:5000`` in your browser to see it in action.


Debugging Flask
---------------

You've seen how ``pdb`` can come in handy in debugging Python code. And when we
discussed the CGI specification, you used ``cgitb`` to enable tracebacks for
errors raised in otherwise impenetrable CGI code.

Flask provides a combination of these features.

To explore them a bit, add a divide-by-0 error to your app, and also update the
call to ``app.run()``:

.. code-block:: python

    def hello_world():
        bar = 1 / 0         #<- Add an error so you can see the fireworks.
        return 'Hello World!'

    if __name__ == '__main__':
        app.run(debug=True) #<- Update this here


Restart your app and then reload your browser to see what happens.

Hover over a single line in the stack trace and keep your eye on the right end
of the window. You should see a little terminal icon and a page icon. Click on
the terminal.

Flask provides (by way of the development server it uses) an in-broswer
debugger that allows you to interact with the code frames in your stack trace
whenever an error happens. This is pretty nice.

Play with this for a while. Try introducing some slightly more complex code
with a few bound symbols and see what you can find out about them.

(clean up your error(s) when you're done playing).


A Quick Recap
-------------

Time for a quick review of what you've accomplished so far:

* You instantiated a `Flask` app with a name that represents the package or
  module containing the app

  * Because our app is a single Python module, this should be ``__name__``
  * This is used to help the `Flask` app figure out where to look for
    *resources*

* You defined a function that returned a "response body"
* You told the app which requests should be handled by that function with a
  *route*

Let's take a look at how that last bit works for a moment...


URL Routing
===========

Remember our bookdb WSGI exercise? How did you end up solving the problem of
mapping an HTTP request to the right function?

Flask solves this problem by using the `route` decorator from your app.

A 'route' takes a URL rule (more on that in a minute) and maps it to an
*endpoint* and a *function*.

When a request arrives at a URL that matches a known rule, the function is
called.


URL Rules
---------

URL Rules are strings.  They represent what environ['PATH_INFO'] will look like
when it arrives at your application code.

These rules are added to a *mapping* on the Flask object called the *url_map*.

You can call ``app.add_url_rule()`` to add a new one

Or you can use what we've used, the ``app.route()`` decorator.

This *imperative* code example:

.. code-block:: python

    def index():
        """some function that returns something"""
        return "this is the homepage"
    
    app.add_url_rule('/foo', 'homepage', index)

is identical to this *declarative* example.

    .. code-block:: python
    
        @app.route('/foo', endpoint='homepage')
        def index():
            """some function that returns something"""
            return "this is the homepage"

Pick one of the above, and add a the new function to your ``flask_intro.py``
app file. (Make sure it comes above the ``__main__``)

Notice that the the route for the ``hello_world`` function we defined in
``flask_intro.py`` did not have an ``endpoint`` argument. This argument allows
us to establish a different 'name' for our route than the name of the function
that will handle the route. If this argument is ommitted, then the 'name' for
the route will be the same as the name of the function.

Reversing URLs
--------------

The 'name' of a route is useful to us because it allows us to build a url for a
part of an application without needing to know it ahead of time.

This means *you don't have to hard-code your URLs when building links*

That means *you can change the URLs for your app without changing code or
templates*

This separation is called **decoupling** and it is a good thing.

Let's try it out to make the concept clear.

Fire up a python interpreter in the same directory where you've created
``flask_intro.py``. When it's ready, type the following:

.. code-block:: python

    >>> from flask_intro import app
    >>> from flask import url_for
    >>> with app.test_request_context():
    ...     print url_for('hello_world');
    ...     print url_for('homepage');
    ...
    /
    /foo
    >>> 


Notice that we didn't need to know what the url for a give endpoint was, we
just needed to know its name.

There are a few important points to note about what we just did:

* ``url_for`` requires an *HTTP request context* to work
* You can fake an HTTP request when working in a terminal (or testing)
* The ``test_request_context`` method of your app object provides one.
* That method is a *context manager*, so you can use it with the ``with``
  statement. When execution leaves the block defined by ``with``, the context
  is safely destroyed.

Dynamic URLs
------------

In our WSGI bookapp, we created a dynamic URL for each book detail page using
Python regular expressions.

Flask allows for dynamic urls as well, but uses *placeholders* instead.

A *placeholder* in a URL rule becomes a named arg to your function.

And *converters* ensure the incoming argument is of the correct type.

Add these two new routes to ``flask_intro.py``:

.. code-block:: python

    @app.route('/profile/<username>')
    def show_profile(username):
        return "My username is %s" % username

    @app.route('/div/<float:val>/')
    def divide(val):
        return "%0.2f divided by 2 is %0.2f" % (val, val / 2)

There are four available *converters* for urls:

* **string** accepts any text without a slash (the default)
* **int** accepts integers
* **float** like int but for floating point values
* **path** like the default but also accepts slashes

Filtered URLS
-------------

You can also determine which HTTP *methods* a given route will accept:

.. code-block:: python

    @app.route('/blog/entry/<int:id>/', methods=['GET',])
    def read_entry(id):
        return "reading entry %d" % id

    @app.route('/blog/entry/<int:id>/', methods=['POST', ])
    def write_entry(id):
        return 'writing entry %d' % id

After adding that to ``flask_intro.py`` and saving, start up your app and try
loading ``http://localhost:5000/blog/entry/23/`` into your browser. Which of
the two was called?

Before finishing the tutorial, try reversing one more time now that you have a
bit more variety to work with:

Quit your Flask app with ``^C``.  Then start a python interpreter in that same
terminal and import your ``flask_intro.py`` module:

.. code-block:: python

    >>> from flask_intro import app
    >>> from flask import url_for
    >>> with app.test_request_context():
    ...     print url_for('show_profile', username="cris")
    ...     print url_for('divide', val=23.7)
    ... 
    '/profile/cris/'
    '/div/23.7/'
    >>> 

That will give you plenty to think about before class.


