*********************
Concurrency in Python
*********************

We've been working for a few days now on writing servers and clients in Python.

To do so, we've made extensive use of the ``socket`` library and the interface
it provides to low-level network I/O primitives.

There's a problem with the code we've been writing, however. It is
``blocking``.

Consider the following code, part of our basic echo server:

.. code-block:: python
    :linenos:
    :emphasize-lines: 10

    def server():
        address = ('127.0.0.1', 10000)
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        sock.bind(address)
        sock.listen(5)
        buffsize = 16

        try:
            while True:
                conn, addr = sock.accept() # blocking
                try:
                    while True:
                        data = conn.recv(buffsize)
                        if data:
                            conn.sendall(data)
                        else:
                            conn.shutdown(socket.SHUT_RDWR)
                            break
                finally:
                    conn.close()
        except KeyboardInterrupt:
            sock.close()

Line 10 of this block contains a ``blocking`` call to ``socket.accept``. This
means that although in principle our server can handle more than one connection
(it has a backlog of 5, right?), in fact it is only able to process one request
at a time.

Even for a trivial program like this, this is a problem. What if the message
the client is sending us to be echoed back is the collected works of
Shakespeare? Any other clients looking to have their simple messages echoed
would be fored to wait while we process Shakespeare 16 bytes at a time.

Operations that ``block`` like this are called *synchronous*. Getting around
them is one of the tasks that comes with scaling a program.

Simple Concurrency w/ ``select``
================================

The Python standard library provides the ``select`` module to help us alleviate
some of the issues with blocking I/O operations.

The module provides ``select``, an interface to the underlying Unix ``select``
system call. The purpose of select is to take a list of possible I/O channels
(file handles, Unix sockets, network sockets) and return lists of those that
are ready to be read, written or in an error state.

Consider the following update to our echo server code:

.. code-block:: python
    :linenos:
    :emphasize-lines: 9, 10, 14, 31

    def server(log_buffer=sys.stderr):
        address = ('127.0.0.1', 10000)
        server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server_socket.bind(address)
        server_socket.listen(5)
        buffsize = 16

        input = [server_socket, sys.stdin]
        running = True
    
        while running:
            
            read_ready, write_ready, except_ready = select.select(input, [], [], 0)
            
            for readable in read_ready:

                if readable is server_socket:
                    # spin up new handler sockets as clients connect
                    handler_socket, address = readable.accept() # won't block now
                    input.append(handler_socket)
                
                elif readable is sys.stdin:
                    # handle any stdin by terminating the server
                    sys.stdin.readline()
                    running = False
                
                else:
                    # handle each client connection 1 buffer at a time
                    data = readable.recv(buffsize) # also won't block
                    if data:
                        # return one buffer's worth of message to the client
                        readable.sendall(data)
                    else:
                        readable.close()
                        input.remove(readable)

        server_socket.close()

What's Changed
--------------

This code has incorporated ``select.select`` in order to allow the outer loop
to continue running regardless of the readiness of any given I/O operation.

On line 9-10, we initialize a list of possible I/O sources by providing the server
socket we've created and ``sys.stdin``, which we will use to gracefully
terminate the server.

On line 14, we pass this initialized list of I/O sources into a call to
``select.select`` as the potential readables. The list of potential writables
and potential exceptions can be ignored for this application. By additionally
providing a ``timeout`` value of ``0``, we ensure that this call to
``select.select`` will not block the loop. If no sources are ready for reading
when the call is executed, the lists will be returned empty and we'll simply
move on to the next iteration.

When the list of I/O sources bound to ``read_ready`` is non-empty, we use a for
loop construct to handle each read_ready source individually.

Now, when we call ``accept`` on the original server socket on line 18, we can
be assured that in fact a client *has* made a connection and the call will
return immediately. When it does, we add the new socket it has given us, which
represents the connection to the client, to the list of inputs so that it can
be processed by ``select`` along with our server and stdin sources.

Once a handler for a client has been added to the list of inputs, it might also
be one of the items in the ``read_ready`` list returned by the call to
``select.`` This brings us to line 31, where a single buffer of data is
received from the client. The data is immediately returned to the client, if
present.

In this way, a series of longer messages from clients may be handled
concurrently, with each being dealt with one buffer at a time. All requests
might take a bit longer than they would have if they came in one at a time, but
the aggregate time spent for a message received when the load is heavy will be
lower. Shakespeare will no longer prevent a smaller message from being
processed in a timely fashion.


What Hasn't Changed
-------------------

For a simple process like this echo server, an update like this might be enough
to solve the problem. But what if the job the server needs to accomplish for
any given connection is more complex? Even if we are handling jobs "at the same
time", we still need to deal with the length of time it takes each job to run
through a given cycle.

Although we've woven a number of tasks together, completing any one of them can
still have a significant impact on how quickly any others might be completed.


Batteries Included
==================

Python provides us with a couple of possible solutions within the standard
library: `threading`_ and `multiprocessing`_. Each takes a different approach
to solving the problem of sharing resources among concurrent tasks.

.. _threading: http://docs.python.org/2/library/threading.html
.. _multiprocessing: http://docs.python.org/2/library/multiprocessing.html

Threading
---------

The ``threading`` module extends and provides a more useable API for the older,
more basic ``thread`` module.  You should never need to address the ``thread``
module directly.

Threads are lightweight, independent processes that share the memory space used
by the process that spawns them. This allows for great convenience in
processing shared data. But that convenience comes at the price of complexity,
as using threads can lead to race conditions, deadlocks and other hard-to-debug
problems unless done with the greatest of care.

To use threading in our echo server, we could spin off a new thread for every
client connection and allow the thread to run itself. A reasonable example of
this approach, implemented as a set of Python classes, `can be seen here`_.

.. _can be seen here: http://ilab.cs.byu.edu/python/threadingmodule.html

But threading doesn't really solve our problems.  Although it *does* ostensibly
create a separate process in which to run the handler code for a client
connection, in reality, that process shares the GIL and so even though it is
running 'separately', it's bound by the same resource limitations as simply
using ``select``.

Multiprocessing
---------------

The ``multiprocessing`` module basically offers that same API as the threading
module. So using it would look much like the example linked above, except that
we would create the ``Client`` class as a subclass of
``multiprocessing.Process`` rather than ``threading.Thread``.

Multiprocessing differs from threading in that it creates completely separate
processes rather than lightweight threads. The processes created **do not
share** memory space with each-other, and so if you have a need to mutate some
shared state with multiprocessing, you'll need to pass messages or data from
one process to another.

On the other hand, because they do not share memory, much of the challenge of
threaded programming is avoided. There is no worry about deadlocks and race
conditions because no two processes are working with the same resources.

It appears that there is general agreement that the future of concurrent
programming in Python lies more along the path of multiprocessing than
threading. It's easier to comprehend, fits better with the mental model most
programmers have of processes running in isolation, and the drawbacks of
isolation can be reduced with not too much effort.


Asynchronous Concurrency
========================

Another approach to the issue of concurrency is to treat the problem
asynchronously. This approach uses the idea of *events* to handle I/O outside
the flow of the main process.

If you think of the fact of a client connecting to a server socket as an event,
then when such an event happens, you can hand off the problem of dealing with
that event to a *handler*. The handler can go on it's merry way, doing what it
needs to do, without concern for what happens back in the server process.

There are two models for this type of *event-driven* operation: Callbacks and
Coroutines. These two methods are nicely modeled by two well-known packages in
the Python ecosystem: twisted and gevent

With either of these packages, it's possible to write our echo server to handle
incoming requests asynchronously. This can enable enourmous concurrency
allowing a server to deal with incoming requests on the order of tens of
thousands at once.

For today, we'll be using gevent. The advantage of gevent over twisted is that
it allows you to write code as if it were synchronous, without needing to worry
about when it might be interrupted.

The code for this example is remarkably light, simply because the majority of
the work is done by a ``StreamServer`` class provided by gevent. All we have to
do is write the portion that actually handles a single client connection.

.. code-block:: python
    :linenos:

    def echo(socket, address):
        """a simple echoing handler for incoming client connections

        It will be used for each incoming connection on a separate greenlet.
        """
        buffsize = 16
        while True:
            data = socket.recv(buffsize)
            if data:
                socket.sendall(data)
            else:
                socket.close()
                break

This code looks remarkably like the connection-handling code from our original
server, and from the ``select`` version above. In fact, it is exactly the same,
minus the work needed to manage readable sockets for select.

Running this server is accomplished like so:

.. code-block:: python
    :linenos:
    :emphasize-lines: 4

    if __name__ == '__main__':
        from gevent.server import StreamServer
        from gevent.monkey import patch_all
        patch_all()
        server = StreamServer(('127.0.0.1', 10000), echo)
        print('Starting echo server on port 10000')
        server.serve_forever()

Note on line 4 the call to ``gevent.monkey.patch_all``. Gevent works by
leveraging a low-level library called ``libevent``. To accomodate this library,
it makes some changes to how blocking is handled by code in a number of
standard Python libraries such as socket. In order to fully take advantage of
these optimizations, calling one of the patch functions from ``gevent.monkey``
(or simply calling ``patch_all``) will apply these changes so that if you are
using a library built on one of these bases (such as ``requests``) you can take
advantage of the asynchronous I/O provided by gevent.

