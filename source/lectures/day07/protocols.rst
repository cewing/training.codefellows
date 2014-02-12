***************************
Network Protocols in Python
***************************

During this walkthrough we'll learn the basics of network protocols.

We'll take a look at sample interactions in common network protocols and
compare and contrast them. Doing so can often help us to understand in what
situations one protocol would be more appropriate than another.

Finally, we'll interact with one of these protocols using a Python
implementation from the standard library. These library implementations will
help you to use a protocol without needing to remember the exact specifics of
each.


What is a Protocol?
===================

a set of rules or conventions governing communications


Protocols IRL
-------------

Life has lots of sets of rules for how to do things.

* What do you say when you get on the elevator?

* What do you do on a first date?

* What do you wear to a job interview?

* What do (and don't) you talk about at a dinner party?

* ...?

.. image:: img/icup.png
    :align: center
    :width: 58%

http://blog.xkcd.com/2009/09/02/urinal-protocol-vulnerability/


Protocols In Computers
----------------------

Digital life has lots of rules too:

.. class:: incremental

* how to say hello

* how to identify yourself

* how to ask for information

* how to provide answers

* how to say goodbye


Protocol Examples
=================

What does this look like in practice?

* SMTP (Simple Message Transfer Protocol)
  http://tools.ietf.org/html/rfc5321#appendix-D

* POP3 (Post Office Protocol)
  http://www.faqs.org/docs/artu/ch05s03.html

* IMAP (Internet Message Access Protocol)
  http://www.faqs.org/docs/artu/ch05s03.html

* HTTP (Hyper-Text Transfer Protocol)
  http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol


The SMTP Protocol
-----------------

SMTP (Say hello and identify yourself)::

    S: 220 foo.com Simple Mail Transfer Service Ready
    C: EHLO bar.com
    S: 250-foo.com greets bar.com
    S: 250-8BITMIME
    S: 250-SIZE
    S: 250-DSN
    S: 250 HELP

SMTP (Ask for information, provide answers)::

    C: MAIL FROM:<Smith@bar.com>
    S: 250 OK
    C: RCPT TO:<Jones@foo.com>
    S: 250 OK
    C: RCPT TO:<Green@foo.com>
    S: 550 No such user here
    C: DATA
    S: 354 Start mail input; end with <CRLF>.<CRLF>
    C: Blah blah blah...
    C: ...etc. etc. etc.
    C: .
    S: 250 OK

SMTP (Say goodbye)::

    C: QUIT
    S: 221 foo.com Service closing transmission channel


SMTP Characteristics
--------------------

* Interaction consists of commands and replies
* Each command or reply is *one line* terminated by <CRLF>
* The exception is message payload, terminated by <CRLF>.<CRLF>
* Each command has a *verb* and one or more *arguments*
* Each reply has a formal *code* and an informal *explanation*


The POP3 Protocol
-----------------

POP3 (Say hello and identify yourself)::

    C: <client connects to service port 110>
    S: +OK POP3 server ready <1896.6971@mailgate.dobbs.org>
    C: USER bob
    S: +OK bob
    C: PASS redqueen
    S: +OK bob's maildrop has 2 messages (320 octets)

POP3 (Ask for information, provide answers)::

    C: STAT
    S: +OK 2 320
    C: LIST
    S: +OK 1 messages (120 octets)
    S: 1 120
    S: .

POP3 (Ask for information, provide answers)::

    C: RETR 1
    S: +OK 120 octets
    S: <server sends the text of message 1>
    S: .
    C: DELE 1
    S: +OK message 1 deleted

POP3 (Say goodbye)::

    C: QUIT
    S: +OK dewey POP3 server signing off (maildrop empty)
    C: <client hangs up>


POP3 Characteristics
--------------------

* Interaction consists of commands and replies
* Each command or reply is *one line* terminated by <CRLF>
* The exception is message payload, terminated by <CRLF>.<CRLF>
* Each command has a *verb* and one or more *arguments*
* Each reply has a formal *code* and an informal *explanation*

The codes don't really look the same, though, do they?


One Other Difference
--------------------

The exception to the one-line-per-message rule is *payload*

In both SMTP and POP3 this is terminated by <CRLF>.<CRLF>

In SMTP, the *client* has this ability

But in POP3, it belongs to the *server*.  Why?


The IMAP Protocol
=================

IMAP (Say hello and identify yourself)::

    C: <client connects to service port 143>
    S: * OK example.com IMAP4rev1 v12.264 server ready
    C: A0001 USER "frobozz" "xyzzy"
    S: * OK User frobozz authenticated

IMAP (Ask for information, provide answers [connect to an inbox])::

    C: A0002 SELECT INBOX
    S: * 1 EXISTS
    S: * 1 RECENT
    S: * FLAGS (\Answered \Flagged \Deleted \Draft \Seen)
    S: * OK [UNSEEN 1] first unseen message in /var/spool/mail/esr
    S: A0002 OK [READ-WRITE] SELECT completed

IMAP (Ask for information, provide answers [Get message sizes])::

    C: A0003 FETCH 1 RFC822.SIZE
    S: * 1 FETCH (RFC822.SIZE 2545)
    S: A0003 OK FETCH completed

IMAP (Ask for information, provide answers [Get first message header])::

    C: A0004 FETCH 1 BODY[HEADER]
    S: * 1 FETCH (RFC822.HEADER {1425}
    <server sends 1425 octets of message payload>
    S: )
    S: A0004 OK FETCH completed

IMAP (Ask for information, provide answers [Get first message body])::

    C: A0005 FETCH 1 BODY[TEXT]
    S: * 1 FETCH (BODY[TEXT] {1120}
    <server sends 1120 octets of message payload>
    S: )
    S: * 1 FETCH (FLAGS (\Recent \Seen))
    S: A0005 OK FETCH completed

IMAP (Say goodbye)::

    C: A0006 LOGOUT
    S: * BYE example.com IMAP4rev1 server terminating connection
    S: A0006 OK LOGOUT completed
    C: <client hangs up>


IMAP Characteristics
--------------------

* Interaction consists of commands and replies
* Each command or reply is *one line* terminated by <CRLF>
* Each command has a *verb* and one or more *arguments*
* Each reply has a formal *code* and an informal *explanation*


IMAP Differences
----------------

* Commands and replies are prefixed by 'sequence identifier'
* Payloads are prefixed by message size, rather than terminated by reserved
  sequence

Compared with POP3, what do these differences suggest?


Protocols in Python
===================

Let's try this out for ourselves

Fire up your python interpreters and prepare to type.

Begin by importing the ``imaplib`` module from the Python Standard Library:

.. code-block:: pycon

    >>> import imaplib
    >>> dir(imaplib)
    ['AllowedVersions', 'CRLF', 'Commands', 
     'Continuation', 'Debug', 'Flags', 'IMAP4', 
     'IMAP4_PORT', 'IMAP4_SSL', 'IMAP4_SSL_PORT', 
     ...
     'socket', 'ssl', 'sys', 'time']
    >>> imaplib.Debug = 4

Setting ``imap.Debug`` shows us what is sent and received

I've prepared a server for us to use, we'll need to set up a client to speak
to it. Our server requires SSL for connecting to IMAP servers, so let's
initialize an IMAP4_SSL client and authenticate:

.. code-block:: pycon

    >>> conn = imaplib.IMAP4_SSL('mail.webfaction.com')
      57:04.83 imaplib version 2.58
      57:04.83 new IMAP4 connection, tag=FNHG
      ...
    >>> conn.login(username, password)
      12:16.50 > IMAD1 LOGIN username password
      12:18.52 < IMAD1 OK Logged in.
    ('OK', ['Logged in.'])

We can start by listing the mailboxes we have on the server:

.. code-block:: pycon

    >>> conn.list()
      00:41.91 > FNHG3 LIST "" *
      00:41.99 < * LIST (\HasNoChildren) "." "INBOX"
      00:41.99 < FNHG3 OK List completed.
    ('OK', ['(\\HasNoChildren) "." "INBOX"'])

To interact with our email, we must select a mailbox from the list we received
earlier:

.. code-block:: pycon

    >>> conn.select('INBOX')
      00:00.47 > FNHG2 SELECT INBOX
      00:00.56 < * FLAGS (\Answered \Flagged \Deleted \Seen \Draft)
      00:00.56 < * OK [PERMANENTFLAGS (\Answered \Flagged \Deleted \Seen \Draft \*)] Flags permitted.
      00:00.56 < * 2 EXISTS
      00:00.57 < * 0 RECENT
      00:00.57 < * OK [UNSEEN 2] First unseen.
      00:00.57 < * OK [UIDVALIDITY 1357449499] UIDs valid
      00:00.57 < * OK [UIDNEXT 3] Predicted next UID
      00:00.57 < FNHG2 OK [READ-WRITE] Select completed.
    ('OK', ['2'])

We can search our selected mailbox for messages matching one or more criteria.
The return value is a string list of the UIDs of messages that match our
search:

.. code-block:: pycon

    >>> conn.search(None, '(FROM "cris")')
      18:25.41 > FNHG5 SEARCH (FROM "cris")
      18:25.54 < * SEARCH 1
      18:25.54 < FNHG5 OK Search completed.
    ('OK', ['1'])
    >>>

Once we've found a message we want to look at, we can use the ``fetch``
command to read it from the server. IMAP allows fetching each part of
a message independently:

.. code-block:: pycon

    >>> conn.fetch('1', '(BODY[HEADER])')
    ...
    >>> conn.fetch('1', '(BODY[TEXT])')
    ...
    >>> conn.fetch('1', '(FLAGS)')


Python Means Batteries Included
-------------------------------

So we can download an entire message and then make a Python email message
object:

.. code-block:: pycon

    >>> import email
    >>> typ, data = conn.fetch('1', '(RFC822)')
      28:08.40 > FNHG8 FETCH 1 (RFC822)
      ...

Parse the returned data to get to the actual message:

.. code-block:: pycon

    >>> for part in data:
    ...   if isinstance(part, tuple):
    ...     msg = email.message_from_string(part[1])
    ... 
    >>> 

Once we have that, we can play with the resulting email object:

.. code-block:: pycon

    >>> msg.keys()
    ['Return-Path', 'X-Original-To', 'Delivered-To', 'Received', 
     ...
     'To', 'Mime-Version', 'X-Mailer']
    >>> msg['To']
    'demo@crisewing.com'
    >>> print msg.get_payload()[0]
    If you are reading this email, ...


What Have We Learned?
---------------------

* Protocols are just a set of rules for how to communicate

* Protocols tell us how to parse and delimit messages

* Protocols tell us what messages are valid

* If we properly format request messages to a server, we can get response
  messages

* Python supports a number of these protocols

* So we don't have to remember how to format the commands ourselves


But in every case we've seen, we could do the same thing with a socket and
some strings
