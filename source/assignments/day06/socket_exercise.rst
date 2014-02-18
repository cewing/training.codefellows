*********************
Socket Communications
*********************

This exercise will walk you through your first client-server interaction using
basic Python sockets

Client Communications
=====================

First, connect and send a message, then tell the server you're done sending:

.. code-block:: pycon

    >>> import socket
    >>> streams = [info
    ...     for info in socket.getaddrinfo('crisewing.com', 'http')
    ...     if info[1] == socket.SOCK_STREAM]
    >>> info = streams[0]
    >>> cewing_socket = socket.socket(*info[:3])
    >>> cewing_socket.connect(info[-1])
    >>> msg = "GET / HTTP/1.1\r\n"
    >>> msg += "Host: crisewing.com\r\n\r\n"
    >>> cewing_socket.sendall(msg)
    >>> cewing_socket.shutdown(socket.SHUT_WR)

Then, receive a reply, iterating until it is complete:

.. code-block:: pycon

    >>> buffsize = 4096
    >>> response = ''
    >>> done = False
    >>> while not done:
    ...     msg_part = cewing_socket.recv(buffsize)
    ...     if len(msg_part) < buffsize:
    ...         done = True
    ...         cewing_socket.close()
    ...     response += msg_part
    ... 
    >>> len(response)
    19427
    >>> cewing_socket.close()

When you are finished with a connection, you should **always close it**!


Server Side
===========

.. class:: big-centered

What about the other half of the equation?

Construct a Socket
------------------

**For the time being, don't bother typing this.  You'll do it again shortly**

Again, we begin by constructing a socket. Since we are actually the server
this time, we get to choose family, type and protocol:


.. code-block:: pycon

    >>> server_socket = socket.socket(
    ...     socket.AF_INET,
    ...     socket.SOCK_STREAM,
    ...     socket.IPPROTO_TCP)
    ... 
    >>> server_socket
    <socket._socketobject object at 0x100563c90>

Bind and Listen
---------------

Our server socket needs to be bound to an address. This is the IP Address and
Port to which clients must connect:

.. code-block:: pycon

    >>> address = ('127.0.0.1', 50000)
    >>> server_socket.bind(address)
    >>> server_socket.listen(1)

**Terminology Note**: In a server/client relationship, the server *binds* to
an address and port. The client *connects*

* ``listen`` prepares our socket to receive connections from clients.

* The argument to ``listen`` is the *backlog*

* The *backlog* is the **maximum** number of connection requests that the
  socket will queue

* Once the limit is reached, the socket refuses new connections.


Accept Incoming Messages
------------------------

When a socket is listening, it can receive incoming connection requests:

.. code-block:: pycon

    >>> connection, client_address = server_socket.accept() # this blocks until a client connects
    >>> connection.recv(16)

* The ``connection`` returned by a call to ``accept`` is a **new socket**.
  This new socket is used to communicate with the client

* The ``client_address`` is a two-tuple of IP Address and Port for the client
  socket

* When a connection request is 'accepted', it is removed from the backlog
  queue.


Send a Reply
------------

The same socket that received a message from the client may be used to return
a reply:

.. code-block:: pycon

    >>> connection.sendall("message received")
    >>> connection.shutdown(socket.SHUT_WR)

Clean Up
--------

Once a transaction between the client and server is complete, the
``connection`` socket should be closed:

.. code-block:: pycon

    >>> connection.close()

Note that the ``server_socket`` is *never* closed as long as the server
continues to run.

Getting the Flow
================

The flow of this interaction can be a bit confusing.  Let's see it in action
step-by-step.

Open a second python interpreter and place it next to your first so you can
see both of them at the same time.


Create a Server
---------------

**Start Typing Now**

In your first python interpreter, create a server socket and prepare it for
connections:

.. code-block:: pycon

    >>> server_socket = socket.socket(
    ...     socket.AF_INET,
    ...     socket.SOCK_STREAM,
    ...     socket.IPPROTO_IP)
    >>> server_socket.bind(('127.0.0.1', 50000))
    >>> server_socket.listen(1)
    >>> conn, addr = server_socket.accept()

At this point, you should **not** get back a prompt. The server socket is
waiting for a connection to be made.


Create a Client
---------------

In your second interpreter, create a client socket and prepare to send a
message:

.. code-block:: pycon

    >>> import socket
    >>> client_socket = socket.socket(
    ...     socket.AF_INET,
    ...     socket.SOCK_STREAM,
    ...     socket.IPPROTO_IP)

Before connecting, keep your eye on the server interpreter:

.. code-block:: pycon

    >>> client_socket.connect(('127.0.0.1', 50000))

Send a Message Client->Server
-----------------------------

As soon as you made the connection above, you should have seen the prompt
return in your server interpreter. The ``accept`` method finally returned a
new connection socket.

When you're ready, type the following in the *client* interpreter:

.. code-block:: pycon

    >>> client_socket.sendall("Hey, can you hear me?")
    >>> client_socket.shutdown(socket.SHUT_WR)


Receive and Respond
-------------------

Back in your server interpreter, go ahead and receive the message from your
client:

.. code-block:: pycon

    >>> conn.recv(32)
    'Hey, can you hear me?'

Send a message back, and then close up your connection:

.. code-block:: pycon

    >>> conn.sendall("Yes, I hear you.")
    >>> conn.close()


Finish Up
---------

Back in your client interpreter, take a look at the response to your message,
then be sure to close your client socket too:

.. code-block:: pycon

    >>> client_socket.recv(32)
    'Yes, I hear you.'
    >>> client_socket.close()

And now that we're done, we can close up the server too (back in the server
interpreter):

.. code-block:: pycon

    >>> server_socket.close()


Congratulations!
----------------

.. class:: big-centered

You've run your first client-server interaction

Now, you're ready to write a basic echo server and client

