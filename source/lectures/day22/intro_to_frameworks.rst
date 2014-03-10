******************************
Introduction to Web Frameworks
******************************

We've been at this for a couple of weeks now.  We've learned a great deal:

* Sockets, the TCP/IP Stack and Basic Networking Mechanics
* Web Protocols and the Importance of Clear Communication
* APIs and Consuming Data from The Web
* CGI and WSGI and Getting Information to Your Dynamic Applications

Everything we do from here out will be based on tools built using these
*foundational technologies*.

Think of everything we do from here on out as sitting on top of WSGI

This may not *actually* be true, but we will always be working at that level of
abstraction.

Frameworks
==========

From Wikipedia::

    A web application framework (WAF) is a software framework that is designed
    to support the development of dynamic websites, web applications and web
    services. The framework aims to alleviate the overhead associated with
    common activities performed in Web development. For example, many
    frameworks provide libraries for database access, templating frameworks and
    session management, and they often promote code reuse


Great, but what does that really *mean*?

Well, it means that a *framework* is something you use to build an *application*.

A framework allows you to build different kinds of applications.

A framework abstracts what needs to be abstracted, and allows control of the
rest.

Think back over the last weeks. In particular, think about last week and your
first large-scale project.

What were your pain points? Which bits do you wish you didn't have to think
about?

Appropriate Abstractions
------------------------

This last question is important when it comes to choosing a framework

* abstraction ∝ 1/freedom
* The more they choose, the less you can
* *Every* framework makes choices in what to abstract
* *Every* framework makes *different* choices

One important lesson to keep in mind: **Don't Fight the Framework**

There are scores of Python web frameworks (this is a partial list).

========= ======== ======== ========== ==============
Django    Grok     Pylons   TurboGears web2py
Zope      CubicWeb Enamel   Gizmo(QP)  Glashammer
Karrigell Nagare   notmm    Porcupine  QP
SkunkWeb  Spyce    Tipfy    Tornado    WebCore
web.py    Webware  Werkzeug WHIFF      XPRESS
AppWsgi   Bobo     Bo7le    CherryPy   circuits.web
Paste     PyWebLib WebStack Albatross  Aquarium
Divmod    Nevow    Flask    JOTWeb2    Python Servlet
Engine    Pyramid  Quixote  Spiked     weblayer
========= ======== ======== ========== ==============

Each of them has made choices about the appropriate level of abstraction. Each
has made slightly different choices. Picking the right one is an important
choice.

Choosing a Framework
--------------------

Many folks will tell you "<XYZ> is the **best** framework".

In most cases, what they really mean is "I know how to use <XYZ>"

In some cases, what they really mean is "<XYZ> fits my brain the best"

What they usually forget is that everyone's brain (and everyone's use-case) is
different.


Cris' First Law of Frameworks: **Pick the Right Tool for the Job**

First Corollary:

    The right tool is the tool that allows you to finish the job quickly and
    correctly.

But how do you know which that one is?


Cris' Second Law of Frameworks: **You can't know unless you try**

So that's what we're going to do.  Over the next three weeks we'll be trying
out three of the top players in the Python web framework world.

We'll begin with the current king of the micro-framework class, **Flask**.

Flask
=====

Last night you walked through a quick introduction to the *Flask* web
framework. You wrote a file that looked like this (at least at some point):

.. code-block:: python

    from flask import Flask
    
    app = Flask(__name__)

    @app.route('/')
    def hello_world():
        return 'Hello World!'

    if __name__ == '__main__':
        app.run()


When you ran this file with your virtualenv Python executable, you should have
seen something like this in your browser:

.. image:: /_static/flask_hello.png
    :align: center
    :width: 60%


What's Happening Here?
----------------------

Flask the framework provides a Python class called `Flask`. This class
functions as a single *application* in the WSGI sense.

We know a WSGI application must be a *callable* that takes the arguments
*environ* and *start_response*.

It has to call the *start_response* method, providing status and headers.

And it has to return an *iterable* that represents the HTTP response body.


In Python, an object is a *callable* if it has a ``__call__`` method.

Take a moment to start up your ``flask_intro`` virtualenv and fire up a Python
interpreter:

.. code-block:: bash

    heffalump:~ cewing$ workon flask_intro
    [flask_intro]
    heffalump:flask_intro cewing$ python
    Python 2.7.5 (default, Aug 25 2013, 00:04:04)
    [GCC 4.2.1 Compatible Apple LLVM 5.0 (clang-500.0.68)] on darwin
    Type "help", "copyright", "credits" or "license" for more information.
    >>>

Once there, import the ``flask`` package. Our ``app`` is an instance of the
``Flask`` class from this package.  Let's go look that up and see what it does:

.. code-block:: pycon

    >>> import flask
    >>> flask.__file__
    '/Users/cewing/virtualenvs/flask_intro/lib/python2.7/site-packages/flask/__init__.pyc'
    >>> 

Open that ``flask`` directory in your editor and open ``__init__.py``:

.. code-block:: python
    :linenos:

    # -*- coding: utf-8 -*-
    """
        flask
        ~~~~~

        A microframework based on Werkzeug.  It's extensively documented
        and follows best practice patterns.

        :copyright: (c) 2011 by Armin Ronacher.
        :license: BSD, see LICENSE for more details.
    """

    __version__ = '0.10.1'

    # utilities we import from Werkzeug and Jinja2 that are unused
    # in the module but are exported as public interface.
    from werkzeug.exceptions import abort
    from werkzeug.utils import redirect
    from jinja2 import Markup, escape

    from .app import Flask, Request, Response
    from .config import Config

On line 21 you should see that ``Flask`` is imported into the global ``flask``
namespace from ``.app``.  Open the ``app.py`` file to dig a bit further.

Here's the ``__call__`` method of the ``Flask`` class (lines 1834-36 in my
version):

.. code-block:: python

    def __call__(self, environ, start_response):
        """Shortcut for :attr:`wsgi_app`."""
        return self.wsgi_app(environ, start_response)

As you can see, it calls another method, called ``wsgi_app``.  Let's follow
this down...

.. code-block:: python

    def wsgi_app(self, environ, start_response):
        """The actual WSGI application.  
        ...
        """
        ctx = self.request_context(environ)
        ctx.push()
        error = None
        try:
            try:
                response = self.full_dispatch_request()
            except Exception as e:
                error = e
                response = self.make_response(self.handle_exception(e))
            return response(environ, start_response)
        #...

``response`` is another WSGI app.  ``Flask`` is actually *WSGI middleware*!

Following this all the way down leads to a ``Response`` class from a package
called *werkzeug*. Here's the ``__call__`` method provided by that class:

.. code-block:: python

    def __call__(self, environ, start_response):
        """Process this response as WSGI application.

        :param environ: the WSGI environment.
        :param start_response: the response callable provided by the WSGI
                               server.
        :return: an application iterator
        """
        app_iter, status, headers = self.get_wsgi_response(environ)
        start_response(status, headers)
        return app_iter

Given the amount of time you've spent over the last week working on WSGI apps,
this should look pretty familiar to you.

All Python web frameworks that operate under the WSGI spec will do this same
sort of thing.

They have to do it.

And these layers of abstraction allow you, the developer to focus only on the
thing that really matters to you.

Getting input from a request, and returning a response.

In the case of ``Flask`` both the Request and the Response are actually
instances of Python classes defined in the ``werkzeug`` package. These classes
smooth over some of the complications of interacting with the raw WSGI
``environ``.

So Without Further Ado
======================

In addition to walking through a Flask intro last night you should also have
completed a walkthrough of interacting with a PostgreSQL database using the
``psycopg2`` DBAPI wrapper.

Today we are going to put those two items together and create the base for a
tumblr-like microblog application.

We'll spend the next bit whiteboarding what is needed for that, and figuring
out what we'll need to know to get it going.

