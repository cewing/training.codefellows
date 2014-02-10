*************
Which Python?
*************

There are two versions of Python we might possibly use for this course: 2.7.x
or 3.x.

For a number of reasons, we have chosen to teach this course in Python 2.7. For
the time being, Python 2 is the version you are most likely to see in a
professiona situation. Every company we spoke with here in Seattle uses it, and
the large pre-existing code base built in Python 2 means that it will be a
while before Python 3 is the default Python to learn.

That being said, some of the exercises you will be doing this week during our
intensive Python review are written in Python 3. This should not be a problem,
though.

For the purposes of the exercises this week, please learn these two facts:

1. In Python there are two division operators: ``/`` and ``//``. In Python 3
   these are ``division`` and ``integer division`` (also known as ``floored
   division``):

    .. code-block:: pycon

        >>> 1 / 2
        0.5
        >>> 1 // 2
        0
        >>> 3 / 2
        1.5
        >>> 3 // 2
        1

   But in Python 2, things are a bit different. The ``//`` operator works
   similarly (it still gives you the ``floored`` result). The ``/`` operator is
   a completely different story. The return values for both depend on what you
   give them, and the ``/`` operator may work as floored division if you give 
   use integers:

    .. code-block:: pycon
    
        >>> 1 // 2
        0
        >>> 1 / 2
        0
        >>> 1.0 / 2
        0.5
        >>> 1.0 // 2
        0.0

   If you want your division in 2.7 to work like what you see in your
   assignments this week, you can fix it using an import from the special
   ``__future__`` module:

    .. code-block:: pycon
    
        >>> from __future__ import division
        >>> 1 / 2
        0.5
        >>> 1 // 2
        0
        >>> 3 / 2
        1.5
        >>> 3 // 2
        1
        >>> 4.0 // 2.0
        2.0
        >>> 4 / 2
        2.0
        >>> 4 // 2
        2

   You can use this in Python 2.7, but remember, much existing Python code does
   not do so. Be aware of the difference when reviewing code written for Python
   2.x

2. In Python 3, ``print`` has been changed from a *statement* to a *function*.
   Here's how print looks in Python 3:

    .. code-block:: language
    
        >>> print('this is a much better syntax')
        this is a much better syntax
        >>> from StringIO import StringIO
        >>> from io import StringIO
        >>> buff = StringIO()
        >>> print('this also makes more sense', file=buff)
        >>> buff.tell()
        27
        >>> buff.seek(0)
        0
        >>> print(buff.read())
        this also makes more sense

   But in Python 2.x it looks like this:

    .. code-block:: pycon
    
        >>> print `this is a funny way of doing things'
        this is a funny way of doing things
        >>> from StringIO import StringIO
        >>> buff = StringIO()
        >>> buff.tell()
        0
        >>> print >>buff, 'this is even more odd'
        >>> buff.tell()
        22
        >>> buff.seek(0)
        >>> print buff.read()
        this is even more odd

   Again, a quick import from ``__future__`` will bring this to you, but beware
   that most Python 2.x code you see **will not** look like this:

    .. code-block:: pycon
    
        >>> from __future__ import print_function
        >>> print('ahh that is much nicer')
        ahh that is much nicer
        >>> newbuff = StringIO()
        >>> print('and this works, too', file=newbuff)
        >>> newbuff.tell()
        20
        >>> newbuff.seek(0)
        >>> print(newbuff.read())
        and this works, too

Be warned. The code you see in your exercises will be Python 3 in these two
respects, but in our class, we will be using the Python 2.x way of doing
things. Make sure that you build a mental map of both syntaxes, so you can
recognize the differences *in situ*.

