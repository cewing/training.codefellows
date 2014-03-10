****************************
Jinja2 Template Introduction
****************************

When you installed ``flask`` into your virtualenv, along with it came a
Python-based templating engine called ``Jinja2``.

In this walkthrough, you'll see some basics about how templates work, and get
to know what sorts of options they provide you for creating HTML from a Python
process.

::

    "I enjoy writing HTML in Python"

      -- nobody, ever


A good framework will provide some way of generating HTML with a templating
system.

There are nearly as many templating systems as there are frameworks. Each comes
with advantages and disadvantages. advantages and disadvantages.

Flask includes the *Jinja2* templating system (perhaps because it's built by
the same folks)


Jinja2 Template Basics
======================

Let's start with the absolute basics.

Fire up a Python interpreter, using your flask virtualenv:
    
.. code-block:: bash

    (flaskenv)$ python

.. code-block:: pycon

    >>> from jinja2 import Template

A template starts life as a simple string:

.. code-block:: pycon

    >>> t1 = Template("Hello {{ name }}, how are you?")
    >>> 

But it has a bit more to it than that. You can call the ``render`` method of a
template object, providing some *context*:

.. code-block:: pycon

    >>> t1.render(name="Freddy")
    u'Hello Freddy, how are you?'
    >>> t1.render({'name': "Roberto"})
    u'Hello Roberto, how are you?'
    >>> 

*Context* can either be keyword arguments, or a dictionary. 

Simple Python values passed in as context will be resolved in the template by
the *key* they are assigned to in the *context*.  These keys are arbitrary.
*Placeholders* like ``{{ name }}`` in this example will be replaced by the
corresponding values from the *context*.

Item and Attribute Access in Templates
--------------------------------------

Dictionaries passed in as part of the *context* can be addressed with *either*
subscript or dotted notation:

.. code-block:: pycon

    >>> person = {'first_name': 'Frank',
    ...           'last_name': 'Herbert'}
    >>> t2 = Template("{{ person.last_name }}, {{ person['first_name'] }}")
    >>> t2.render(person=person)
    u'Herbert, Frank'

* Jinja2 will try the *correct* way first (attr for dotted, item for
  subscript).
* If nothing is found, it will try the opposite.
* If nothing is found, it will return an *undefined* object.

The exact same is true of objects passed in as part of *context*:

.. code-block:: pycon

    >>> t3 = Template("{{ obj.x }} + {{ obj['y'] }} = Fun!")
    >>> class Game(object):
    ...   x = 'babies'
    ...   y = 'bubbles'
    ...
    >>> bathtime = Game()
    >>> t3.render(obj=bathtime)
    u'babies + bubbles = Fun!'

This means your templates can be a bit agnostic as to the nature of the things
passed in via *context*

`Read more about variables in Jinja2 templates`_.

.. _Read more about variables in Jinja2 templates: http://jinja.pocoo.org/docs/templates/#variables


Filtering values in Templates
-----------------------------

You can apply *filters* to the data passed in *context* with the pipe ('|')
operator:

.. code-block:: pycon

    t4 = Template("shouted: {{ phrase|upper }}")
    >>> t4.render(phrase="this is very important")
    u'shouted: THIS IS VERY IMPORTANT'

You can also chain filters together:

.. code-block:: python

    t5 = Template("confusing: {{ phrase|upper|reverse }}")
    >>> t5.render(phrase="howdy doody")
    u'confusing: YDOOD YDWOH'

There are `a large number of filters`_ available to use in ``jinja2``.

.. _a large number of filters: http://jinja.pocoo.org/docs/templates/#builtin-filters



Control Flow
------------

``Jinja2`` provides all the expected control structures of a featureful
programming language:

.. code-block:: pycon

    tmpl = """
    ... {% for item in list %}{{ item }}, {% endfor %}
    ... """
    >>> t6 = Template(tmpl)
    >>> t6.render(list=[1,2,3,4,5,6])
    u'\n1, 2, 3, 4, 5, 6, '

Any control structure introduced in a template **must** be paired with an 
explicit closing tag ({% for %}...{% endfor %})

You can `learn more about control structures`_ by reading the documentation.

.. _learn more about control structures: http://jinja.pocoo.org/docs/templates/#list-of-control-structures


Conditionals in Templates
-------------------------

There are a number of specialized *tests* available for use with the
``if...elif...else`` control structure:

.. code-block:: pycon

    >>> tmpl = """
    ... {% if phrase is upper %}
    ...   {{ phrase|lower }}
    ... {% elif phrase is lower %}
    ...   {{ phrase|upper }}
    ... {% else %}{{ phrase }}{% endif %}"""
    >>> t7 = Template(tmpl)
    >>> t7.render(phrase="FOO")
    u'\n\n  foo\n'
    >>> t7.render(phrase="bar")
    u'\n\n  BAR\n'
    >>> t7.render(phrase="This should print as-is")
    u'\nThis should print as-is'

`Here's a list`_ of all the built-in tests in the ``jinja2`` template language.

.. _Here's a list: http://jinja.pocoo.org/docs/templates/#builtin-tests

Python Expressions in Templates
-------------------------------

You can also use basic Python-like expressions in ``jinja2`` templates. There
are some syntactic differences, though.

.. code-block:: pycon

    tmpl = """
    ... {% set sum = 0 %}
    ... {% for val in values %}
    ... {{ val }}: {{ sum + val }}
    ...   {% set sum = sum + val %}
    ... {% endfor %}
    ... """
    >>> t8 = Template(tmpl)
    >>> t8.render(values=range(1,11))
    u'\n\n\n1: 1\n  \n\n2: 3\n  \n\n3: 6\n  \n\n4: 10\n
      \n\n5: 15\n  \n\n6: 21\n  \n\n7: 28\n  \n\n8: 36\n
      \n\n9: 45\n  \n\n10: 55\n  \n'

`Learn all about expressions`_, including `assignments`_  in the documentation.

.. _Learn all about expressions: http://jinja.pocoo.org/docs/templates/#expressions
.. _assignments: http://jinja.pocoo.org/docs/templates/#assignments


Jinja2 Templates in Flask
=========================

The Jinja2 template engine has a concept it calls an *Environment*. The
environment for the template engine is used to:

* Figure out where to look for templates
* Set configuration for the templating system
* Add some commonly used functionality to the template *context*

In Flask, this environment is set up automatically when you instantiate your
``app``.

Flask uses the value you pass to the ``app`` constructor to calculate the root
of your application on the filesystem. This means that Flask apps have a sense
of their own filesystem location.

From that root, a Flask app expects to find templates in a directory name
``templates``.

This allows you to use the ``render_template`` command from ``flask`` like
so:

.. code-block:: pycon
    
    from flask import render_template
    page_html = render_template('hello_world.html', name="Cris")

In this case, Flask would expect to find a file called ``hello_world.html`` in
a directory called ``templates`` in the app where this call appeared.

Let's look at what a template file like that might look like:

.. code-block:: jinja

    {% extends "layout.html" %}
    {% block body %}
      <h1>Hello World!</h1>
    {% endblock %}

That's not much to look at.  Where's the rest of the HTML that makes up a page?

Template Inheritance
--------------------

``Jinja2`` templates allow for *inheritance*.  This means that you can create
shared structure in base templates, and then override or fill in named parts of
that structure in *sub-templates*.

In the above case, the ``hello_world.html`` sub-template *extends* the
``layout.html`` template. What does that file look like?

.. code-block:: jinja

    <!DOCTYPE html>
    <html>
      <head>
        <title>Hello World!</title>
      </head>
      <body>
        <h1>A simple page.</h1>
        <div class="content">
        {% block body %}{% endblock %}
        </div>
      </body>
    </html>

You can see here that the ``body`` block is defined in ``layout.html`` and then
that block is filled by the templating in ``hello_world.html``.

Inheritance can work the other way, as well. In addition to filling blocks in a
larger structure, you can pull in smaller blocks using the ``include`` template
tag.  For example, all the pages on your site might include a common footer
which is defined in ``footer.html``:

.. code-block:: jinja

    <div id="footer">
      I am the footer, seen on all pages.
    </div>

Then, we can include this structure in our ``layout.html`` file:

.. code-block:: jinja

    <!DOCTYPE html>
    <html>
      <head>
        <title>Hello World!</title>
      </head>
      <body>
        <h1>A simple page.</h1>
        <div class="content">
        {% block body %}{% endblock %}
        </div>
        {% include "footer.html" %}
      </body>
    </html>

Finally, you can also *import* template macros from templates where you define
them. This can be a convenient way to create libraries of shareable template
structures for repetetive elements like form inputs:

.. code-block:: jinja2

    {% macro input(name, value='', type='text') -%}
        <input type="{{ type }}" value="{{ value|e }}" name="{{ name }}">
    {%- endmacro %}

    {%- macro textarea(name, value='', rows=10, cols=40) -%}
        <textarea name="{{ name }}" rows="{{ rows }}" cols="{{ cols
            }}">{{ value|e }}</textarea>
    {%- endmacro %}

Once such a library is established, say in a file called ``forms.html``, the
macros it contains can be used in other templates:

.. code-block:: jinja

    {% import 'forms.html' as forms %}
    <dl>
        <dt>Username</dt>
        <dd>{{ forms.input('username') }}</dd>
        <dt>Password</dt>
        <dd>{{ forms.input('password', type='password') }}</dd>
    </dl>
    <p>{{ forms.textarea('comment') }}</p>

There's more to learn about `inheritance`_ and `importing`_ than we can cover
here, so read up.

.. _inheritance: http://jinja.pocoo.org/docs/templates/#template-inheritance
.. _importing: http://jinja.pocoo.org/docs/templates/#import


Common Flask Context
--------------------

Keyword arguments you pass to ``render_template`` become the *context* passed
to the template for rendering.

Flask will add a few things to this context, thanks to the ``jinja2``
environment it creates.

* **config**: contains the current configuration object
* **request**: contains the current request object
* **session**: any session data that might be available
* **g**: the request-local object to which global variables are bound
* **url_for**: so you can easily *reverse* urls from within your templates
* **get_flashed_messages**: a function that returns messages you flash to your
  users (more on this later).


Much, Much More
===============

Make sure that you bookmark the Jinja2 documentation for later use::

    http://jinja.pocoo.org/docs/templates/
