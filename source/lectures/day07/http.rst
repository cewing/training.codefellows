*******************************
Understanding the HTTP Protocol
*******************************

We've seen a number of network protocols so far.

Each of them consisted of a set of allowed commands and possible responses,
*messages* that are passed from client to server and then from server to
client.

In each protocol we've seen so far, these commands and responses have been
*delimited*

And for each of the protocols we've seen so far, that delimiter has been
consistent: <CRLF>

A further consistency is shared between these protocols. In each case we've
seen so far, the *client* is responsible for initiating the interaction. The
*server* is passive until it receives some sort of request.

The HTTP Protocol
=================

HTTP is no different

HTTP is also message-centered, with two-way communications we call *requests*
and *responses*.

* Requests (Asking for information)
* Responses (Providing answers)


HTTP Request (Ask for information)::

    GET /index.html HTTP/1.1
    Host: www.example.com
    <CRLF>

HTTP Response (Provide answers)::

    HTTP/1.1 200 OK
    Date: Mon, 23 May 2005 22:38:34 GMT
    Server: Apache/1.3.3.7 (Unix) (Red-Hat/Linux)
    Last-Modified: Wed, 08 Jan 2003 23:11:55 GMT
    Etag: "3f80f-1b6-3e1cb03b"
    Accept-Ranges:  none
    Content-Length: 438
    Connection: close
    Content-Type: text/html; charset=UTF-8
    <CRLF>
    <438 bytes of content>


HTTP Req/Resp Format
--------------------

Both share a common basic format:

* Line separators are <CRLF> (familiar, no?)
* A *required* initial line (a command or a response code)
* A *(mostly) optional* set of headers, one per line
* A blank line
* An *optional* body

Let's look at each a bit more closely.


HTTP Requests
=============

In HTTP 1.0, the only required line in an HTTP request looks like this::

    GET /path/to/index.html HTTP/1.0
    <CRLF>


As virtual hosting grew more common, that was not enough, so HTTP 1.1 adds a
single required *header*, **Host**:

    GET /path/to/index.html HTTP/1.1
    Host: www.mysite1.com:80
    <CRLF>

Every HTTP request **must** begin with a single line, broken by whitespace into
three parts::

    GET /path/to/index.html HTTP/1.1

The three parts are the *method*, the *URI*, and the *protocol*

Let's look at each in turn.


HTTP Methods
------------

**GET** ``/path/to/index.html HTTP/1.1``

* Every HTTP request must start with a *method*
* There are four main HTTP methods:

    * GET
    * POST
    * PUT
    * DELETE

* There are others, notably HEAD, but you won't see them too much

These four methods can be mapped to the four basic steps (*CRUD*) of persistent
storage:

* POST = Create
* GET = Read
* PUT = Update
* DELETE = Delete


Safe <--> Unsafe
----------------

HTTP methods can be categorized as **safe** or **unsafe**, based on whether
they might *change something* on the server:

* Safe HTTP Methods
    * GET
* Unsafe HTTP Methods
    * POST
    * PUT
    * DELETE

This is a *normative* distinction, which is to say **be careful**


Idempotent <--> Non-Idempotent
------------------------------

HTTP methods can be categorized as **idempotent**, based on whether a given
request will *always* have the same result:

* Idempotent HTTP Methods
    * GET
    * PUT
    * DELETE
* Non-Idempotent HTTP Methods
    * POST

Again, *normative*. The developer is responsible for ensuring that it is true.


HTTP Requests: URI
------------------

``GET`` **/path/to/index.html** ``HTTP/1.1``

* Every HTTP request must include a **URI** used to determine the **resource** to
  be returned

* URI??
  http://stackoverflow.com/questions/176264/whats-the-difference-between-a-uri-and-a-url/1984225#1984225

* In static systems, the URI maps directly to a filesystem location on the
  server.

* In dynamic systems, it may still do so (PHP, CGI)

  * It may also be used to determine what code object should be used to build a
    response

* Static or dynamic, we call whatever that end point might be a *resource*

* Resource?  Files (html, img, .js, .css), but also:

  * Dynamic scripts
  * Raw data
  * API endpoints


In any server application, this job of connecting the URI requested to the
appropriate end point is very important.


HTTP Responses
==============

In both HTTP 1.0 and 1.1, a proper response consists of an intial line,
followed by optional headers, a single blank line, and then optionally a
response body::

    HTTP/1.1 200 OK
    Content-Type: text/plain
    <CRLF>
    this is a pretty minimal response

As with requests, the initial line of the response is strictly formatted,
divided by whitespace into a *response code* and an *explanation*


HTTP Response Codes
-------------------

``HTTP/1.1`` **200 OK**

All HTTP responses must include a **response code** indicating the outcome of
the request.

* 1xx (HTTP 1.1 only) - Informational message
* 2xx - Success of some kind
* 3xx - Redirection of some kind
* 4xx - Client Error of some kind
* 5xx - Server Error of some kind

The response code is a machine-readable number. The explanation that follows is
meant as a way to make the responses more human-friendly.


Common Response Codes
---------------------

There are certain HTTP response codes you are likely to see (and use) most
often:

* ``200 OK`` - Everything is good
* ``301 Moved Permanently`` - You should update your link
* ``304 Not Modified`` - You should load this from cache
* ``404 Not Found`` - You've asked for something that doesn't exist
* ``500 Internal Server Error`` - Something bad happened

Do not be afraid to use other, less common codes in building good apps. There
are a lot of them for a reason. See
http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html


HTTP Headers
------------

After the required initial line of a response, the HTTP protocol allows the
server to send additional information to the client in the form of **headers**

In fact, both requests and responses can contain headers. So a client can also
use them to send extra data to the server.

Headers take the form ``<Name>: <Value>``


* HTTP 1.0 has 16 valid headers, 1.1 has 46
* Any number of spaces or tabs may separate the *name* from the *value*
* If a header line starts with spaces or tabs, it is considered part of the
  *value* for the previous header
* Header *names* are **not** case-sensitive, but *values* may be

It's well worth being familiar with the possible headers for HTTP.  You can
`read more about them here`_.

 .. _read more about them here: http://www.cs.tut.fi/~jkorpela/http.html

Common HTTP Response Headers
----------------------------

There are a couple of headers we'll talk about immediately, because they are
so common.

The first is the ``Content-Type`` header. It tells the client how to treat the
data that is being returned in the body of the response.

* it uses **mime-type** (Multi-purpose Internet Mail Extensions)
* foo.jpeg - ``Content-Type: image/jpeg``
* foo.png - ``Content-Type: image/png``
* bar.txt - ``Content-Type: text/plain``
* baz.html - ``Content-Type: text/html``

There are *many* `mime-type identifiers`_.

.. _mime-type identifiers: http://www.webmaster-toolkit.com/mime-types.shtml:

The Python standard library provides a module that helps in determining the
mimetype of a given file. It's called ``mimetypes``.

Using it, you can guess the mime-type of a file based on the filename or map a
file extension to a type:

.. code-block:: pycon

    >>> textfile = "/path/to/textfile.txt"
    >>> mimetypes.guess_type(textfile)
    ('text/plain', None)
    >>> import os
    >>> text_extension = os.path.splitext(textfile)
    >>> text_extension
    ('/path/to/textfile', '.txt')
    >>> mimetypes.types_map[text_extension[1]]
    'text/plain'
    >>> imagefile = "/path/to/imagefile.png"
    >>> mimetypes.guess_type(imagefile)
    ('image/png', None)
    >>> image_extension = os.path.splitext(imagefile)
    >>> image_extension
    ('/path/to/imagefile', '.png')
    >>> mimetypes.types_map[image_extension[1]]
    'image/png'


Another common HTTP header is the ``Date`` header. It represents the date and
time that a response was generated.

The value for this header must be expressed in GMT, not local time, and has a
very particular format::

    Fri, 12 Feb 2010 16:23:03 GMT

The Python standard library also provides a way of getting exactly this format.
Since the format is almost exactly the same as that required for email headers,
this method is found in a slightly unexpected module:

.. code-block:: pycon

    >>> import email.utils
    >>> email.utils.formatdate(usegmt=True)
    'Fri, 12 Feb 2010 16:23:03 GMT'


A third common HTTP header is the ``Content-Length`` header, used to inform the
client just how much data to expect in the body of a response.

Since HTTP does not specify a delimiter for a response body (unlike the SMTP,
POP3 and IMAP protocols), this header is particularly important.

The value for the header should correspond to the number of bytes of data that
will be returned. For binary files like images calculating this value is quite
straightforward:

.. code-block:: pycon

    >>> with open('Mars1.jpg', 'rb') as file_handle:
    ...     mars_image = file_handle.read()
    ...
    >>> length = len(mars_image)
    >>> length
    1161387

However, when text is involved it gets a bit more complicated. Best practice in
Python is to keep text that you are working with as ``unicode`` objects:

    >>> body = u'éclaire'
    >>> len(body)
    7

Remember though that a socket can **only** transmit bytes, not decoded unicode
objects, so in Python you must be sure that the content of the response body
you send has been encoded:

    >>> bytes = body.encode('utf-8')
    >>> len(bytes)
    8

Notice that the length of the encoded byte string is *longer* than the decoded
unicode string. This is because the encoded form of the ``é`` character is
actually *two bytes* in length.

When sending text back to a client, it is best practice to include information
about what *codec* was used to encode the bytes you send.

It's tempting to think of the ``Content-Encoding`` header as the proper place
to send this data, but in fact that is used to inform the client of
*compressed* data (.zip or similar).

Instead, the correct way to inform the client of the encoding used is to append
a ``charset <name>`` value to the ``Content-Type`` header::

    Content-Type: text/plain; charset=utf-8



