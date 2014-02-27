************************************
A Simple Multi-Page WSGI Application
************************************

To explore the concept of a WSGI application that serves more than one resource
more in-depth, we'll build a simple application.

Our application will serve a small database of python books.

Set Up
======

Begin by creating a new project to work in:

.. code-block:: bash

    heffalump:tests cewing$ mkproject bookapp
    New python executable in bookapp/bin/python
    Installing setuptools, pip...done.
    Creating /Users/cewing/projects/bookapp
    Setting project for bookapp to /Users/cewing/projects/bookapp
    [bookapp]
    heffalump:bookapp cewing$

Our application will need some data.  I've got a simple database that we can
use all set up.

The database (with a very simple api) `can be found here`_. Copy that code and
paste it into a file in your bookapp project folder.  Keep the same name
(``bookdb.py``).

.. _can be found here: https://github.com/cewing/training.python_web/blob/master/resources/session04/wsgi/bookdb.py

Once you have that in place, you're ready to get started on the application.

What It Does
============

A good place to start when thinking about a new app is to consider what it does.

For our application we will want it to provide the following:

* a listing page that shows the titles of all the books
* each title will link to a details page for that book
* a details page for each book that will display all the information and have a
  link back to the list

Our First Problem
-----------------

When viewing our first wsgi app, do we see the name of the wsgi application
script anywhere in the URL?

In our wsgi application script, how many applications did we actually have?

How are we going to serve different types of information out of a single
application?

Dispatch
--------

We have to write an app that will map our incoming request path to some code
that can handle that request.

This process is called ``dispatch``. There are many possible approaches

Let's begin by designing this piece of it.

Create a new file called ``bookapp.py`` in your bookapp project folder.  We'll
do our work here.

The wsgi environment gives us access to *PATH_INFO*, which maps to the URI the
user requested when they loaded the page.

We can design the URLs that our app will use to assist us in routing.

Let's declare that any request for ``/`` will map to the list page

We can also say that the URL for a book will look like this::

    http://localhost:8080/book/<identifier>

We need to design a function that will take the incoming path information from
the wsgi environment and map it to something that will actually build our
response for us.

Conceptually, this is much like the ``map_uri`` function you built for your
HTTP server.

The difference is that the thing it finds will be a Python function, not a
filesystem file or folder.

Add a new function called ``resolve_path`` in our application file.

.. class:: incremental

* It should take the *PATH_INFO* value from environ as an argument.
* It should return the function that will be called.
* It should also return any arguments needed to call that function.
* This implies of course that the arguments should be part of the PATH


.. code-block:: python

    import re
    from bookdb import BookDB

    DB = BookDB()


    def resolve_path(path):
        urls = [(r'^$', books),
                (r'^book/(id[\d]+)$', book)]
        matchpath = path.lstrip('/')
        for regexp, func in urls:
            match = re.match(regexp, matchpath)
            if match is None:
                continue
            args = match.groups([])
            return func, args
        # we get here if no url matches
        raise NameError

Because this code references symbols (``book`` and ``books``) that do not
exist, we need to make some dummy functions to stand in for them:

.. code-block:: python

    def book(book_id):
        return "<h1>a book with id %s</h1>" % book_id

    def books():
        return "<h1>a list of books</h1>"


Application Code
----------------

These function are not a WSGI application. They are pieces that the application
we write will use to make things happen.

Let's add our actual application next:

* The path should be extracted from ``environ``.
* The dispatch function should be used to get a function and arguments
* The body to return should come from calling that function with those
  arguments
* If an error is raised by calling the function, an appropriate response
  should be returned
* If the router raises a NameError, the application should return a 404
  response

.. code-block:: python

    def application(environ, start_response):
        headers = [("Content-type", "text/html")]
        try:
            path = environ.get('PATH_INFO', None)
            if path is None:
                raise NameError
            func, args = resolve_path(path)
            body = func(*args)
            status = "200 OK"
        except NameError:
            status = "404 Not Found"
            body = "<h1>Not Found</h1>"
        except Exception:
            status = "500 Internal Server Error"
            body = "<h1>Internal Server Error</h1>"
        finally:
            headers.append(('Content-length', str(len(body))))
            start_response(status, headers)
            return [body]

Finally, you'll need to add a ``__main__`` block to run your application:

.. code-block:: python

        if __name__ == '__main__':
            from wsgiref.simple_server import make_server
            srv = make_server('localhost', 8080, application)
            srv.serve_forever()

Once you've got your script settled, run it::

    $ python bookapp.py

Then point your browser at ``http://localhost:8080/``

* ``http://localhost/book/id3``
* ``http://localhost/book/id73/``
* ``http://localhost/sponge/damp``

Did that all work as you would have expected?


Handling Requests
-----------------

The basics of our app are already in place.  Let's move on next to build the
functions that will generate our individual pages.

The function ``books`` should return an html list of book titles where each
title is a link to the detail page for that book

* You'll need all the ids and titles from the book database.
* You'll need to build a list in HTML using this information
* Each list item should have the book title as a link
* The href for the link should be of the form ``/book/<id>``

Look at the ``bookdb.py`` file and ad the api for the books

.. code-block:: python

    def books():
        all_books = DB.titles()
        body = ['<h1>My Bookshelf</h1>', '<ul>']
        item_template = '<li><a href="/book/{id}">{title}</a></li>'
        for book in all_books:
            body.append(item_template.format(**book))
        body.append('</ul>')
        return '\n'.join(body)

To see the effect of this function, quit your application and restart it::

    $ python bookapp.py

Then reload the root of your application::

    http://localhost:8080/

You should see a nice list of the books in the database. Do you?

Click on a link to view the detail page. Does it load without error?

The next step of course is to polish up those detail pages.

* You'll need to retrieve a single book from the database
* You'll need to format the details about that book and return them as HTML
* You'll need to guard against ids that do not map to books

In this last case, what's the right HTTP response code to send?

.. code-block:: python

    def book(book_id):
        page = """
    <h1>{title}</h1>
    <table>
        <tr><th>Author</th><td>{author}</td></tr>
        <tr><th>Publisher</th><td>{publisher}</td></tr>
        <tr><th>ISBN</th><td>{isbn}</td></tr>
    </table>
    <a href="/">Back to the list</a>
    """
        book = DB.title_info(book_id)
        if book is None:
            raise NameError
        return page.format(**book)

Quit and restart your script one more time

Then poke around at your application and see the good you've made

And your application is portable and sharable

It should run equally well under any
`wsgi server <http://www.wsgi.org/en/latest/applications.html>`_


A Few Steps Further
-------------------

Next steps for an app like this might be:

* Create a shared full page template and incorporate it into your app
* Improve the error handling by emitting error codes other than 404 and 500
* Swap out the basic backend here with a different one, maybe a Web Service?
* Think about ways to make the application less tightly coupled to the pages
  it serves
* Write tests to cover your functions (and the database too).
