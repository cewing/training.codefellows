****************************************
Behavior Driven Development with Lettuce
****************************************

You've created a github repository for a project called FizzBuzz, as part of
a previous assignment.

We're going to use that project as an example of how we can use the Python
package `lettuce`_ to incorporate `behavior driven development`_.

.. _lettuce: http://lettuce.it
.. _behavior driven development: http://en.wikipedia.org/wiki/Behavior-driven_development

Practice Safe Development
=========================

Because ``lettuce`` is not in the standard library, we're going to follow best
practices and use `virtualenv`_ to create a sandbox for this project.

.. _virtualenv: http://virtualenv.org

.. note:: If you have already set up a virtualenv for your fizzbuzz project,
          you can skip this up until the point where we install lettuce below.


First, make sure you're in your fizzbuzz repository:

.. code-block:: bash

    $ cd /path/to/fizzbuzz/repository
    [master=]
    heffalump:fizzbuzz cewing$ pwd
    /Users/cewing/projects/fizzbuzz

Next, we create a virtualenv using ``mkvirtualenv`` provided by `virtualenvwrapper`_:

.. _virtualenvwrapper: http://virtualenvwrapper.readthedocs.org

.. code-block:: bash

    [master=]
    heffalump:fizzbuzz cewing$ mkvirtualenv fizzbuzz
    New python executable in fizzbuzz/bin/python
    Installing setuptools, pip...done.
    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$


Setting Our Project
===================

Now, our virtualenv is set up and active. Since we've already got a "project"
directory--our existing repository--we are going to use the
``setvirtualenvproject`` command to associate this new virtualenv with our
existing repository directory. 

The command takes two arguments, the path to the virtualenv, and the path to
the project directory. Both these arguments must be **full, absolute path
names**.

It's a pain to type out those pathnames, luckily, ``bash`` (or whatever shell
you are using) can come to the rescue.

First, when a virtualenv is active, it sets an environmental variable
``VIRTUAL_ENV`` which holds the path to the active virtualenv:

.. code-block:: bash

    heffalump:fizzbuzz cewing$ echo $VIRTUAL_ENV
    /Users/cewing/virtualenvs/fizzbuzz

Second, the ``pwd`` command will print our current working directory, which is
the project directory we want. If we surround this bash command with backticks,
the output will be substituted into a shell expression.

.. code-block:: bash

    source

    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$ pwd
    /Users/cewing/projects/fizzbuzz
    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$ setvirtualenvproject $VIRTUAL_ENV `pwd`
    Setting project for fizzbuzz to /Users/cewing/projects/fizzbuzz
    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$

Excellent.  Now, when we ``workon`` fizzbuzz, we'll *both* activate the
environment and automatically move to this directory.

Time Saved!!!

Install Lettuce
===============

Now, we're ready to go ahead and install lettuce:

.. code-block:: bash

    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$ pip install lettuce
    Downloading/unpacking lettuce
      ...

    Successfully installed lettuce sure fuzzywuzzy nose rednose python-termstyle
    Cleaning up...
    [fizzbuzz]
    [master=]

Once that's finished, we should find that we have a new command available in
our shell: ``lettuce``:

.. code-block:: bash

    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$ which lettuce
    /Users/cewing/virtualenvs/fizzbuzz/bin/lettuce


Lettuce Setup
=============

There's a nice walkthrough for lettuce `on the website`_. If you're starting
totally from scratch, try it out.

.. _on the website: http://lettuce.it/tutorial/simple.html#tutorial-simple

We aren't.

The basic gist of the walkthrough tells us that ``lettuce`` works by
convention. To have lettuce tests, you have to set up an appropriate directory
structure in your Python project.

We'll create a directory called ``features``.  And in it we'll create two
files: ``fizzbuzz.feature`` and ``steps.py``.

.. code-block:: bash

    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$ mkdir features
    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$ touch features/fizzbuzz.feature
    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$ touch features/steps.py

The ``<name>.feature`` file is where we'll create our BDD tests.  The name of
the file is arbitrary, the extension is not.

The ``<name>.py`` file is where we will create the Python code that implements
steps in our BDD tests.  Again, the name is arbitrary, and ``lettuce`` will
scan for any ``.py`` files that contain step definitions.

Write a Scenario
================

Our example here is a bit artificial, in that we are adding BDD tests *after*
we've already written the Python.  To make things a bit more realistic, comment
out the function bodies for the two functions in your fizzbuzz.py file. Each
function should look like this (where ``function name`` is of course replaced
by the actual names in your fizzbuzz Python file).

.. code-block:: python

    def function_name(n):
        pass

Now we can assume we have yet to implement the fizzbuzz function.

If we wanted to express the fizzbuzz game as a user story, it might go
something like this::

    As a user, when I call fizzbuzz with the number 5, I see 'Buzz'.

Expressed in BDD form, this translates to something like this:

    Given the number 5, when I call fizzbuzz, then I see the output 'Buzz'

Let's add this 'Scenario' to our BDD tests:

.. code-block:: cucumber

    Feature: Simple FizzBuzz
        Implement a simple version of the FizzBuzz game

        Scenario: FizzBuzz of 5
            Given the number 5
            When I call FizzBuzz
            Then I see the output Buzz

Implement The Steps
===================

We have three steps.  Lettuce will match our steps to Python functions using
the text we set in the scenario, so we need to create three steps.

Steps are defined as Python functions decorated with the ``@step`` decorator
factory from lettuce. The decorator factory takes a single argument which is
the text used for the step in scenarios. Variable values are matched by regular
expression.

Let's implement the first step first.  In ``steps.py`` add the following:

.. code-block:: python

    from lettuce import step
    from lettuce import world

    from fizzbuzz import fizzbuzz


    @step('the number (\d+)')
    def the_number(step, num):
        world.number = int(num)

Here we can see that the ``@step`` decorator is passed the argument "the number
(\d+)".  How does that match? The secret is that there are a few 'magic' words
in ``lettuce``. When the test parser sees ``Given``, ``When`` or ``Then`` it
uses these to determine that the following text on the line is a step. That
text is what is considered for the match.

The regular expression ``(\d+)`` will match a sequence of one or more digits,
and the value it finds will be passed to the function decorated by ``@step``.

Also notice that the Python function decorated by ``@step`` takes *two*
arguments. The first will always be the step itself. The second comes from the
matched regular expression.

How do you expect you might handle a step like this:

.. code-block:: cucumber

    Given the numbers 5 and 7

What would the argument passed to the ``@step`` decorator look like? How many
arguments would the Python function need to expect?

Let's go ahead and add two more steps for the remaining parts of our scenario:

.. code-block:: python

    @step('when I call fizzbuzz')
    def call_fizzbuzz(step):
        world.fb = fizzbuzz(world.number)
    
    @step('I see the output (\w+)')
    def compare(step, expected):
        assert world.fb == expected, "Got %s" % expected


Once you've done this, you should be able to run your BDD tests from the
command line:

.. code-block:: bash

    [fizzbuzz]
    [master *=]
    heffalump:fizzbuzz cewing$ lettuce

    Feature: Simple FizzBuzz                          # features/fizzbuzz.feature:1
      Implement a simple version of the FizzBuzz game # features/fizzbuzz.feature:2

      Scenario: FizzBuzz of 5                         # features/fizzbuzz.feature:4
        Given the number 5                            # features/steps.py:8
        When I call FizzBuzz                          # features/steps.py:13
        Then I see the output Buzz                    # features/steps.py:18
        Traceback (most recent call last):
          File "/Users/cewing/virtualenvs/fizzbuzz/lib/python2.7/site-packages/lettuce/core.py", line 144, in __call__
            ret = self.function(self.step, *args, **kw)
          File "/Users/cewing/projects/fizzbuzz/features/steps.py", line 19, in compare
            assert world.fb == expected, "Got %s" % world.fb
        AssertionError: Got None

    1 feature (0 passed)
    1 scenario (0 passed)
    3 steps (1 failed, 2 passed)

You can see, the tests report that we've run 1 feature, 1 scenario and 3 steps.
They also tell us that one of our steps failed. And they kindly provide a
traceback that shows us where in our steps things went awry.

Excellent.  Now we can uncomment our code for the fizzbuzz function, and re-run
the test:

.. code-block:: bash

    [fizzbuzz]
    [master *=]
    heffalump:fizzbuzz cewing$ lettuce

    Feature: Simple FizzBuzz                          # features/fizzbuzz.feature:1
      Implement a simple version of the FizzBuzz game # features/fizzbuzz.feature:2

      Scenario: FizzBuzz of 5                         # features/fizzbuzz.feature:4
        Given the number 5                            # features/steps.py:8
        When I call FizzBuzz                          # features/steps.py:13
        Then I see the output Buzz                    # features/steps.py:18

    1 feature (1 passed)
    1 scenario (1 passed)
    3 steps (3 passed)

Wonderful!  Our first passing BDD test!

Iterating Over Multiple Calls
=============================

But it really isn't enough to just test with one value. FizzBuzz is a dynamic
process that returns different values depending on what you pass in. We should
cover at least a reasonable range of alternative possibilities.

We could write a new scenario for each different value we want to pass in. But
that would mean re-writing the same lines over and over with only two
variations. Not very programmerish.

Luckily, there's a solution. With the syntax supported by ``lettuce`` we can
replace the *specific* numbers in our current scenario with *placeholders*.
Then we can provide a set of data for the scenario to use, and ``lettuce`` will
automatically plug in our values and run the same scenario over and over for
us. Update ``fizzbuzz.feature`` like so:

.. code-block:: cucumber

        Scenario Outline: FizzBuzz [just enough]
            Given the number <input>
            When I call FizzBuzz
            Then I see the output <output>

        Examples:
        | input | output   |
        | 0     | 0        |
        | 1     | 1        |
        | 3     | Fizz     |
        | 5     | Buzz     |
        | 6     | Fizz     |
        | 10    | Buzz     |
        | 15    | FizzBuzz |

We've changed our *scenario* into a **Scenario Outline**. This lets ``lettuce``
know that we expect this scenario to be run multiple times.

We also replaced the specific input ``5`` and output ``Buzz`` with
placeholders, marked by angle brackets.

Finally, we provided a table of example data for our outline to use. The first
row of the table contains the names of our placeholders, and then each row
represents a set of values to be used for one iteration.

Now when we run the tests again, we can see that we get more passes through the
scenario automatically:

.. code-block:: cucumber

    [fizzbuzz]
    [master=]
    heffalump:fizzbuzz cewing$ lettuce

    Feature: Simple FizzBuzz                          # features/fizzbuzz.feature:1
      Implement a simple version of the FizzBuzz game # features/fizzbuzz.feature:2

      Scenario Outline: FizzBuzz [just enough]        # features/fizzbuzz.feature:4
        Given the number <input>                      # features/steps.py:8
        When I call FizzBuzz                          # features/steps.py:13
        Then I see the output <output>                # features/steps.py:18

      Examples:
        | input | output   |
        | 0     | 0        |
        | 1     | 1        |
        | 3     | Fizz     |
        | 5     | Buzz     |
        | 6     | Fizz     |
        | 10    | Buzz     |
        | 15    | FizzBuzz |

    1 feature (1 passed)
    7 scenarios (7 passed)
    21 steps (21 passed)

Next Steps
==========

We've written one scenario that covers our simple implementation of FizzBuzz.
But the assignment had a second implementation. That version was supposed to be
extensible with additional numbers and the values that should be printed in
their place.  For example, if the number 7 was supplied, the value 'Sizz'
should be returned.

You may also wish to read more of `the documentation`_ for ``lettuce`` and see
if you can't figure out how to add a new scenario and new steps that will cover
that version of the game.

.. _the documentation: http://lettuce.it

To complete this task you'll need to learn a bit more about the syntax of the
language you are using to write these tests.  It's called ``Gherkin`` (the
first implementation was called ``cucumber``). There is a very nice outline of
`the features of Gherkin syntax`_ you can read to learn more. 

You should be aware that not **all** the features of the ``Gherkin`` syntax are
supported by ``lettuce.`` Moreover, some features are supported in alternate
forms, for example, ``Gherkin`` backgrounds are implemented as hooks in
``lettuce``.

.. _the features of Gherkin syntax: http://docs.behat.org/guides/1.gherkin.html

Happy testing!
