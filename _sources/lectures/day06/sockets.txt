******************
TCP/IP and Sockets
******************

Computer Communications
=======================


.. image:: /_static/network_topology.png
    :align: center
    :width: 50%

image: http://en.wikipedia.org/wiki/Internet_Protocol_Suite

* processes can communicate

* inside one machine

* between two machines

* among many machines


.. image:: /_static/data_in_tcpip_stack.png
    :align: center
    :width: 50%

image: http://en.wikipedia.org/wiki/Internet_Protocol_Suite

* This communications is divided into 'layers'

* 'Layers' are mostly arbitrary

* Different descriptions have different layers

* Most common is the 'TCP/IP Stack'


The TCP/IP Stack - Link
-----------------------

The bottom layer is the 'Link Layer'

* Deals with the physical connections between machines, 'the wire'

* Packages data for physical transport

* Executes transmission over a physical medium

  * what that medium is is arbitrary

* Implemented in the Network Interface Card(s) (NIC) in your computer


The TCP/IP Stack - Internet
---------------------------

Moving up, we have the 'Internet Layer'

* Deals with addressing and routing

  * Where are we going and how do we get there?

* Agnostic as to physical medium (IP over Avian Carrier - IPoAC)

* Makes no promises of reliability

* Two addressing systems


  * IPv4 (current, limited '192.168.1.100')

  * IPv6 (future, 3.4 x 10^38 addresses, '2001:0db8:85a3:0042:0000:8a2e:0370:7334')


The TCP/IP Stack - Transport
----------------------------

Next up is the 'Transport Layer'

* Deals with transmission and reception of data

  * error correction, flow control, congestion management

* Common protocols include TCP & UDP

  * TCP: Tranmission Control Protocol

  * UDP: User Datagram Protocol

* Not all Transport Protocols are 'reliable'

  * TCP ensures that dropped packets are resent

  * UDP makes no such assurance

  * Reliability is slow and expensive


The 'Transport Layer' also establishes the concept of a **port**

* IP Addresses designate a specific *machine* on the network

* A **port** provides addressing for individual *applications* in a single host

* 192.168.1.100:80  (the *:80* part is the **port**)

* [2001:db8:85a3:8d3:1319:8a2e:370:7348]:443 (*:443* is the **port**)

This means that you don't have to worry about information intended for your
web browser being accidentally read by your email client.

There are certain **ports** which are commonly understood to belong to given
applications or protocols:

* 80/443 - HTTP/HTTPS
* 20 - FTP
* 22 - SSH
* 23 - Telnet
* 25 - SMTP
* ...

These ports are often referred to as **well-known ports**

(see http://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers)

Ports are grouped into a few different classes

* Ports numbered 0 - 1023 are *reserved*

* Ports numbered 1024 - 65535 are *open*

* Ports numbered 1024 - 49151 may be *registered*

* Ports numbered 49152 - 65535 are called *ephemeral*


The TCP/IP Stack - Application
------------------------------

The topmost layer is the 'Application Layer'

* Deals directly with data produced or consumed by an application

* Reads or writes data using a set of understood, well-defined **protocols**

  * HTTP, SMTP, FTP etc.

* Does not know (or need to know) about lower layer functionality

  * The exception to this rule is **endpoint** data (or IP:Port)

this is where we live and work


Sockets
=======

Think back for a second to what we just finished discussing, the TCP/IP stack.

* The *Internet* layer gives us an **IP Address**

* The *Transport* layer establishes the idea of a **port**.

* The *Application* layer doesn't care about what happens below...

* *Except for* **endpoint data** (IP:Port)

A **Socket** is the software representation of that endpoint.

Opening a **socket** creates a kind of transceiver that can send and/or
receive *bytes* at a given IP address and Port.


Sockets in Python
-----------------

Python provides a standard library module which provides socket functionality.
It is called **socket**.  

The library is really just a very thin wrapper around the system
implementation of *BSD Sockets*

Let's spend a few minutes getting to know this module.

We're going to do this next part together, so open up a terminal and start a
python interpreter

The Python sockets library allows us to find out what port a *service* uses:

.. code-block:: pycon

    >>> import socket
    >>> socket.getservbyname('ssh')
    22

You can also do a *reverse lookup*, finding what service uses a given *port*:
small

.. code-block:: pycon

    >>> socket.getservbyport(80)
    'http'

The sockets library also provides tools for finding out information about
*hosts*. For example, you can find out about the hostname and IP address of
the machine you are currently using:

.. code-block:: pycon

    >>> socket.gethostname()
    'heffalump.local'
    >>> socket.gethostbyname(socket.gethostname())
    '10.211.55.2'

You can also find out about machines that are located elsewhere, assuming you
know their hostname. For example:

.. code-block:: pycon

    >>> socket.gethostbyname('google.com')
    '173.194.33.4'
    >>> socket.gethostbyname('uw.edu')
    '128.95.155.135'
    >>> socket.gethostbyname('crisewing.com')
    '108.59.11.99'

The ``gethostbyname_ex`` method of the ``socket`` library provides more
information about the machines we are exploring:

.. code-block:: pycon

    >>> socket.gethostbyname_ex('google.com')
    ('google.com', [], ['173.194.33.9', '173.194.33.14',
                        ...
                        '173.194.33.6', '173.194.33.7',
                        '173.194.33.8'])
    >>> socket.gethostbyname_ex('crisewing.com')
    ('crisewing.com', [], ['108.59.11.99'])
    >>> socket.gethostbyname_ex('www.rad.washington.edu')
    ('elladan.rad.washington.edu', # <- canonical hostname
     ['www.rad.washington.edu'], # <- any machine aliases
     ['128.95.247.84']) # <- all active IP addresses

To create a socket, you use the **socket** method of the ``socket`` library.
It takes up to three optional positional arguments (here we use none to get
the default behavior):

.. code-block:: pycon

    >>> foo = socket.socket()
    >>> foo
    <socket._socketobject object at 0x10046cec0>

A socket has some properties that are immediately important to us. These
include the *family*, *type* and *protocol* of the socket::

    >>> foo.family
    2
    >>> foo.type
    1
    >>> foo.proto
    0

You might notice that the values for these properties are integers.  In fact, 
these integers are **constants** defined in the socket library.


A quick utility method
----------------------

Let's define a method in place to help us see these constants. It will take a
single argument, the shared prefix for a defined set of constants:

.. code-block:: pycon

    >>> def get_constants(prefix):
    ...     """mapping of socket module constants to their names."""
    ...     return dict(
    ...         (getattr(socket, n), n)
    ...         for n in dir(socket)
    ...         if n.startswith(prefix)
    ...     )
    ...
    >>>

Socket Families
===============

Think back a moment to our discussion of the *Internet* layer of the TCP/IP
stack.  There were a couple of different types of IP addresses:

* IPv4 ('192.168.1.100')

* IPv6 ('2001:0db8:85a3:0042:0000:8a2e:0370:7334')

The **family** of a socket corresponds to the *addressing system* it uses for
connecting.

Families defined in the ``socket`` library are prefixed by ``AF_``:

.. code-block:: pycon

    >>> families = get_constants('AF_')
    >>> families
    {0: 'AF_UNSPEC', 1: 'AF_UNIX', 2: 'AF_INET',
     11: 'AF_SNA', 12: 'AF_DECnet', 16: 'AF_APPLETALK',
     17: 'AF_ROUTE', 23: 'AF_IPX', 30: 'AF_INET6'}

*Your results may vary*

Of all of these, the ones we care most about are ``2`` (IPv4) and ``30`` (IPv6).


Unix Domain Sockets
-------------------

When you are on a machine with an operating system that is Unix-like, you will
find another generally useful socket family: ``AF_UNIX``, or Unix Domain
Sockets. Sockets in this family:

* connect processes **on the same machine**

* are generally a bit slower than IPC connnections

* have the benefit of allowing the same API for programs that might run on one
  machine __or__ across the network

* use an 'address' that looks like a pathname ('/tmp/foo.sock')


Test your skills
----------------

What is the *default* family for the socket we created just a moment ago?

(remember we bound the socket to the symbol ``foo``) center

How did you figure this out?


Socket Types
============

The socket *type* determines the semantics of socket communications.

Look up socket type constants with the ``SOCK_`` prefix:

.. code-block:: pycon

    >>> types = get_constants('SOCK_')
    >>> types
    {1: 'SOCK_STREAM', 2: 'SOCK_DGRAM',
     ...}

The most common are ``1`` (Stream communication (TCP)) and ``2`` (Datagram
communication (UDP)).


Test your skills
----------------

What is the *default* type for our generic socket, ``foo``?


Socket Protocols
================

A socket also has a designated *protocol*. The constants for these are
prefixed by ``IPPROTO_``:

.. code-block:: pycon

    >>> protocols = get_constants('IPPROTO_')
    >>> protocols
    {0: 'IPPROTO_IP', 1: 'IPPROTO_ICMP',
     ...,
     255: 'IPPROTO_RAW'}

The choice of which protocol to use for a socket is determined by the
*internet layer* protocol you intend to use. ``TCP``? ``UDP``? ``ICMP``?
``IGMP``?


Test your skills
----------------

What is the *default* protocol used by our generic socket, ``foo``?


Custom Sockets
--------------

These three properties of a socket correspond to the three positional
arguments you may pass to the socket constructor.

Using them allows you to create sockets with specific communications
profiles:

.. code-block:: pycon

    >>> bar = socket.socket(socket.AF_INET,
    ...                     socket.SOCK_DGRAM, 
    ...                     socket.IPPROTO_UDP)
    ...
    >>> bar
    <socket._socketobject object at 0x1005b8b40>

Address Information
===================

When you are creating a socket to communicate with a remote service, the
remote socket will have a specific communications profile.

The local socket you create must match that communications profile.

How can you determine the *correct* values to use? center

You ask.

The function ``socket.getaddrinfo`` provides information about available
connections on a given host.

.. code-block:: python

    socket.getaddrinfo('127.0.0.1', 80)

This provides all you need to make a proper connection to a socket on a remote
host. The value returned is a tuple of:

* socket family
* socket type
* socket protocol
* canonical name (usually empty, unless requested by flag)
* socket address (tuple of IP and Port)


On Your Own Machine
-------------------

Now, ask your own machine what possible connections are available for 'http':

.. code-block:: pycon

    >>> socket.getaddrinfo(socket.gethostname(), 'http')
    [(2, 2, 17, '', ('10.29.144.178', 80)), 
     ...
     (30, 2, 17, '', ('fe80::e2f8:47ff:fe21:af92%en1', 80, 0, 5)), 
     ...
    ]
    ...
    >>>

What answers do you get?


On the Internet
---------------

.. code-block:: pycon

    >>> get_address_info('crisewing.com', 'http')
    [(2, 2, 17, '', ('108.168.213.86', 80)), (2, 1, 6, '', ('108.168.213.86', 80))]
    >>>

Try a few other servers you know about.

Communicating
=============

Sockets communicate by sending a receiving messages.

Let's test this by building a client socket and communicating with a server.

Client Side Communications
--------------------------

First, connect and send a message:

.. code-block:: pycon

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
    >>> cewing_socket.shutdown(socket.SHUT_RD)
    >>> cewing_socket.close()

Sending Messages
----------------

There are two basic methods on a socket for sending messages, ``send`` and
``sendall``. We're using the latter here.

* the transmission continues until all data is sent or an error occurs

* success returns ``None``

* failure to send raises an error

* you can use the type of error to figure out why the transmission failed

* if an error occurs you **cannot** know how much, if any, of your data was
  sent

With ``send``, you send the message one chunk at a time.  You are responsible
for checking if a particular chunk succeeded or not, and you are also
responsible for determining when the full transmission is done.

Receiving Messages
------------------

The ``recv`` method handles incoming messages in buffers.

* The sole required argument is ``buffer_size`` (an integer). It should be a
  power of 2 and smallish (~4096)
* It returns a byte string of ``buffer_size`` (or smaller if less data was
  received)
* If the response is longer than ``buffer size``, you can call the method
  repeatedly. The last bunch will be less than ``buffer size``.


Accumulators
------------

Hotice that receiving a message is not a one-and-done kind of thing

We don't know how big the incoming message is before we start receiving it.

As a result, we have to use the ``Accumulator`` pattern to gather incoming
buffers of the message until there is no more to get.

The ``recv`` method will return a string less than ``buffsize`` if there isn't
any more to come.

The EOT Problem
---------------

Sockets do not have a concept of the "End Of Transmission".

So what happens if the message coming in is an *exact multiple of the
buffsize*?

There are a couple of strategies for dealing with this. One is to punt to the
*application level protocol* and allow it to predetermine the size of the
message to come. HTTP works this way

The other is to use the ``shutdown`` method of a socket to close that socket
for reading, writing or both.

When you do so, a 0-byte message is sent to the partner socket, allowing it to
know that you are finished.

For more information, read the `Python Socket Programming How-To`_.

.. _Python Socket Programming How-To: http://docs.python.org/2/howto/sockets.html

Exercises
=========

Tonight you'll put this to work, first by walking through a basic client server
interaction, then by building a basic echo server and client.
