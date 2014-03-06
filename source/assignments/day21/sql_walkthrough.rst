*************************
SQL Persistence in Python
*************************

In this tutorial, you'll walk through some basic concepts of data persistence
using the most common Python DBAPI2 connector for the PostgreSQL database,
``pyscopg2``.


Persistence
===========

There are many models for persistance of data.

* Flat files
* Relational Database (SQL RDBMs like PostgreSQL, MySQL, SQLServer, Oracle)
* Object Stores (Pickle, ZODB)
* NoSQL Databases (CouchDB, MongoDB, etc)

It's also one of the most contentious issues in app design.

For this reason, it's one of the things that most Small Frameworks leave
undecided.

`PEP 249 <http://www.python.org/dev/peps/pep-0249/>`_ describes a
common API for database connections called DB-API 2.

The goal was to

    achieve a consistency leading to more easily understood modules, code
    that is generally more portable across databases, and a broader reach
    of database connectivity from Python

source: http://www.python.org/dev/peps/pep-0248/

It is important to remember that PEP 249 is **only a specification**.

There is no code or package for DB-API 2 on it's own.

Since 2.5, the Python Standard Library has provided a `reference
implementation of the api <http://docs.python.org/2/library/sqlite3.html>`_
based on SQLite3.

Before Python 2.5, this package was available as ``pysqlite``

To use the DB API with any database other than SQLite3, you must have an
underlying API package available.

Implementations are available for:

* PostgreSQL (**psycopg2**, txpostgres, ...)
* MySQL (**mysql-python**, PyMySQL, ...)
* MS SQL Server (**adodbapi**, pymssql, mxODBC, pyodbc, ...)
* Oracle (**cx_Oracle**, mxODBC, pyodbc, ...)
* and many more...

source: http://wiki.python.org/moin/DatabaseInterfaces

Most db api packages can be installed using typical Pythonic methods::

    $ pip install psycopg2
    $ pip install mysql-python
    ...

However, most api packages will require that the development headers for the
underlying database system be available. Without these, the C symbols required
for communication with the db are not present and the wrapper cannot work.

During class, we worked on getting PostgreSQL installed on our machines. During
this tutorial, we'll be using that RDBMS and the ``psycopg2`` implementation of
DBAPI2.

Setting Up
==========

The first step in working with a PostgreSQL database is to create a database.
Installing the PostgreSQL Software initializes the database system, but does
not create a database for you to use. You must do this manually.

Begin by activating the virtualenv we created in class:

.. code-block:: bash

    heffalump:psycopg2 cewing$ workon psycopg2
    [psycopg2]
    heffalump:psycopg2 cewing$

Next, be sure that your PostgreSQL is running:

.. code-block:: bash

    heffalump:psycopg2 cewing$ psql
    psql: could not connect to server: No such file or directory
        Is the server running locally and accepting
        connections on Unix domain socket "/tmp/.s.PGSQL.5432"?
    heffalump:psycopg2 cewing$ pg_ctl -D /usr/local/var/postgres/ -l /usr/local/var/postgres/server.log start
    server starting
    heffalump:psycopg2 cewing$

Next, we need to create a database, using the command that PostgreSQL provides
for this purpose, ``createdb``:

.. code-block:: bash

    heffalump:psycopg2 cewing$ createdb psycotest
    heffalump:psycopg2 cewing$

Finally, check to be sure that the database is now present, using the psql
command:

.. code-block:: bash

    heffalump:psycopg2 cewing$ psql -d psycotest
    psql (9.3.2)
    Type "help" for help.

    psycotest=# \d
    No relations found.
    psycotest=# \l
                                    List of databases
        Name     | Owner  | Encoding |   Collate   |    Ctype    | Access privileges
    -------------+--------+----------+-------------+-------------+-------------------
     cewing      | cewing | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
     dvdrental   | cewing | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
     nngroup.com | cewing | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
     postgres    | cewing | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
     psycotest   | cewing | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
     template0   | cewing | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/cewing        +
                 |        |          |             |             | cewing=CTc/cewing
     template1   | cewing | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/cewing        +
                 |        |          |             |             | cewing=CTc/cewing
    (7 rows)

    psycotest=# \q
    [psycopg2]
    heffalump:psycopg2 cewing$

The ``psql`` command opens an interactive shell in PostgreSQL. While you are in
this shell you are working directly in the database you designated with the
``-d`` command flag.

This shell provides a number of special commands.  In the session above we can
see three of them:

* *\d* describes the tables in a database. It can also take the name of one
  table as an argument, in which case it describes the columns in that table.
* *\l* lists all the databases present in the server.
* *\q* exits from the terminal and returns you to your normal shell session.

There is `much more to learn about psql`_ but that will get you going for now.

.. _much more to learn about psql: http://www.postgresql.org/docs/9.3/static/app-psql.html

Data Definition Layer
---------------------

A database is nothing without tables, so we need to create some.

The set of SQL commands that create and modify tables within a database is
called the **Data Definition Layer**.

We'll be creating and working with a simple two-table database today.

In your ``psycopg2`` project folder, create a new file called ``book_ddl.sql``.

Add the following to that file:

.. code-block:: sql

    CREATE TABLE IF NOT EXISTS author(
        authorid serial PRIMARY KEY,
        name VARCHAR (255) NOT NULL
    );

    CREATE TABLE IF NOT EXISTS  book(
        bookid serial PRIMARY KEY,
        title VARCHAR (255) NOT NULL,
        authorid INTEGER REFERENCES author ON UPDATE NO ACTION ON DELETE NO ACTION
    );

These two SQL statements tell the database engine to create two tables, if they
do not already exist.

Each table then has a set of ``columns``. These columns define the types of
data that the table is concerned with.

In both tables we have a ``PRIMARY KEY`` column.  This column is used to
identify rows in the database and must contain unique values.  The data type
``serial`` helps to ensure this as it automatically assigns integer values
starting with 1 and counting upwards.

In both tables we also have a column containing ``VARCHAR`` data. This type
requires that we designate the maximum size of the data that will be held here.
Each of these columns is marked as ``NOT NULL``, meaning that a value is
required.

Finally, in the ``book`` table there is an ``INTEGER`` column which
``REFEREMCES`` a column in the other table. This creates a *Foreign Key*
relationship between the two tables.

Relations such as this are central to SQL databases and are the primary reason
such systems are called **RDBMSs**, or Relational Database Management Systems.

To create our tables, we have to run the commands in this file.  The simplest
way to accomplish this is to feed the file directly to the ``psql`` command,
like so:

.. code-block:: bash

    [psycopg2]
    heffalump:psycopg2 cewing$ psql -d psycotest < book_ddl.sql
    CREATE TABLE
    CREATE TABLE
    [psycopg2]
    heffalump:psycopg2 cewing$

Now, we can re-open our database shell, and see that we have tables:

.. code-block:: psql

    [psycopg2]
    heffalump:psycopg2 cewing$ psql -d psycotest
    psql (9.3.2)
    Type "help" for help.

    psycotest=# \d
                    List of relations
     Schema |        Name         |   Type   | Owner
    --------+---------------------+----------+--------
     public | author              | table    | cewing
     public | author_authorid_seq | sequence | cewing
     public | book                | table    | cewing
     public | book_bookid_seq     | sequence | cewing
    (4 rows)

    psycotest=# \d author
                                         Table "public.author"
      Column  |          Type          |                         Modifiers
    ----------+------------------------+-----------------------------------------------------------
     authorid | integer                | not null default nextval('author_authorid_seq'::regclass)
     name     | character varying(255) | not null
    Indexes:
        "author_pkey" PRIMARY KEY, btree (authorid)
    Referenced by:
        TABLE "book" CONSTRAINT "book_authorid_fkey" FOREIGN KEY (authorid) REFERENCES author(authorid)

    psycotest=# \d book
                                        Table "public.book"
      Column  |          Type          |                       Modifiers
    ----------+------------------------+-------------------------------------------------------
     bookid   | integer                | not null default nextval('book_bookid_seq'::regclass)
     title    | character varying(255) | not null
     authorid | integer                |
    Indexes:
        "book_pkey" PRIMARY KEY, btree (bookid)
    Foreign-key constraints:
        "book_authorid_fkey" FOREIGN KEY (authorid) REFERENCES author(authorid)

    psycotest=# \q
    [psycopg2]
    heffalump:psycopg2 cewing$

Interacting With the Database
=============================

Once all that is in place, we're ready to interact with our database using
``psycopg2``.

Connections and Cursors
-----------------------

We'll begin by getting connected. Connecting to any database consists of
providing a specially-formatted string to the connector, called a **DSN** or
Data Source Name.

Each different type of database uses a different format for this string.  In
PostgreSQL it is typically a set of ``key=value`` pairs where the keys come
from a `defined set of possible keys`_.

.. _defined set of possible keys: http://www.postgresql.org/docs/current/static/libpq-connect.html#LIBPQ-PARAMKEYWORDS

There are a lot of possible keywords, but the ones you are most likely to see
and use are:

* **dbname**: the name of the database in the server you want to connect with.
* **host**: the hostname on which the server is listening. This can also be a
  pathname to a socket file if the system is using Unix Domain Socket
  connections.
* **port**: the port number on which the server is listening. This can also be
  a socket file extension if the system is using Unix Domain Socket
  connections.
* **user**: The username to use when connecting to the database. Default is the
  system name of the user who is running the connect command.
* **password**: The password of the user. This is only used if the system
  requires password authentication.

We set up our database to allow us to connect directly using *ident*
authorization. So the only parameters we must pass are the dbname and user.

Fire up an interactive Python session and get a connection:

.. code-block:: pycon

    [psycopg2]
    heffalump:psycopg2 cewing$ python
    Python 2.7.5 (default, Aug 25 2013, 00:04:04)
    [GCC 4.2.1 Compatible Apple LLVM 5.0 (clang-500.0.68)] on darwin
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import psycopg2
    >>> conn = psycopg2.connection(dbname="psycotest", user="cewing")
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    AttributeError: 'module' object has no attribute 'connection'
    >>> conn = psycopg2.connect(dbname="psycotest", user="cewing")
    >>> conn
    <connection object at 0x7fafc8e005c0; dsn: 'user=cewing dbname=psycotest', closed: 0>
    >>> 

A connection represents our tie to the database. But to interact with it, we
want to use a *cursor*:

.. code-block:: pycon

    >>> cur = conn.cursor()
    >>> cur
    <cursor object at 0x10a370718; closed: 0>
    >>> 

The cursor is a local representation of the state of the database. You can
execute statements on it, and see the results of those statements, but until
you **commit** a transaction, the changes are not persisted to the system on
disk.

Simple Inserts and Selects
--------------------------

Use your cursor to insert a new record into the ``author`` table:

.. code-block:: pycon

    >>> insert = "INSERT INTO author (name) VALUES('Iain M. Banks');"
    >>> cur.execute(insert)
    >>> cur.rowcount
    1
    >>> 

Notice that we ``execute`` a statement using the cursor. After this is done, we
can interrogate the curosr to find out what happened. In this case, we can
learn that one row was inserted.

**NOTE**:

Every so often, you will make an error in typing an SQL command. When you try
to execute the statement, you'll be informed of the error. This is nice. It's
important to note, though, that many kinds of errors can result in the current
transaction with the database being "aborted".

When this happens, you'll see error messages like this:

.. code-block:: pycon

    >>> cur.execute(insert)
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    psycopg2.InternalError: current transaction is aborted, commands ignored until end of transaction block

There is nothing to fear here. You simply have to end a transaction block so
that you can start interacting with the database again. The safest way is to
roll back the transaction, which ensures that nothing since the last commit
will be saved:

.. code-block:: pycon

    >>> conn.rollback()

(more about transactions soon)

We can also retrieve from the database the information we just inserted, using
a ``SELECT`` statement:

.. code-block:: pycon

    >>> query = "SELECT * from author;"
    >>> cur.execute(query)
    >>> cur.fetchall()
    [(1, 'Iain M. Banks')]
    >>> 

You'll see that our select query found one row in the database.  The row is
returned as a tuple with as many values as there are columns in the query. We
asked for all columns (\*) and so we get two. 

The order of the values in each tuple is dependent on the query. In this case
we asked for all columns so we get them in the database order (id, name).

Parameterized Statements
------------------------

Inserting static data one row at a time is tedious.

We are software engineers, we can do better than that.

In order to repeat a statement a number of times, with different values, we
must use *parameters*.

In DBAPI2 packages, these parameters are specialized forms of *placeholders*
used in the strings passed to the ``execute`` command. Each database system
uses its own format, but the general idea is the same. You create an SQL
statement with placeholders where you want values to be inserted. Then you call
the 'execute' command with *two* arguments.  Your parameterized statement, and
a tuple containing as many values as you have parameters.

There is also an ``executemany`` method on a cursor object that supports
passing an iterable of tuples. The SQL statement will be run one time for each
tuple in the iterable:

.. code-block:: pycon

    >>> insert = "INSERT INTO author (name) VALUES(%s);"
    >>> authors = [("China Mieville",), ("Frank Herbert",),
    ...            ("J.R.R. Tolkein",), ("Susan Cooper",),
    ...            ("Madeline L'Engle",), ]
    >>> cur.executemany(insert, authors)
    >>> cur.rowcount
    5
    >>> 

And we can read our inserted values back:

.. code-block:: pycon

    >>> cur.execute(query)
    >>> rows = cur.fetchall()
    >>> for row in rows:
    ...   print row
    ...
    (1, 'Iain M. Banks')
    (2, 'China Mieville')
    (3, 'Frank Herbert')
    (4, 'J.R.R. Tolkein')
    (5, 'Susan Cooper')
    (6, "Madeline L'Engle")
    >>> 

RED LETTER WARNING
------------------

**A SUPER IMPORTANT WARNING THAT YOU MUST PAY ATTENTION TO**

The placeholder for psycopg2 is ``%s``.  This placeholder is the same
regardless of the type of data you are passing in as your values.

**Do Not Be Fooled** into thinking that this means you can use string
formatting to build your SQL statements:

.. code-block:: python

    # THIS IS BAD:
    cur.execute("INSERT INTO author (name) VALUES(%s)" % "Bob Dobbins")

This syntax **does not properly escape the values passed in**.

This syntax leaves you wide open to **SQL Injection Attacks**.

If I ever see you using this syntax I will personally take you out behind the
woodshed and tan your hide.

I'm not kidding.

Python provides you with a syntax that is safe from the kinds of attacks that
make you front page news.  Use it properly:

.. code-block:: python

    cur.execute("INSERT INTO author (name) VALUES(%s)", ("Bob Dobbins", ))


Transactions
------------

Thus far, we haven't actually committed a transaction. If we open a second
terminal and fire up the psql shell program, we can see that the data we've
inserted is not yet *in* our database:

.. code-block:: psql

    heffalump:training.python_web cewing$ psql -d psycotest
    psql (9.3.2)
    Type "help" for help.

    psycotest=# \d
                    List of relations
     Schema |        Name         |   Type   | Owner
    --------+---------------------+----------+--------
     public | author              | table    | cewing
     public | author_authorid_seq | sequence | cewing
     public | book                | table    | cewing
     public | book_bookid_seq     | sequence | cewing
    (4 rows)

    psycotest=# select * from author;
     authorid | name
    ----------+------
    (0 rows)

    psycotest=#

In order for the values we've inserted to actually be persisted to the
filesystem, making them available outside the cursor we have, we must commit a
transaction.

We do this using the connection object we first set up:

.. code-block:: pycon

    >>> conn
    <connection object at 0x7fafc8e005c0; dsn: 'user=cewing dbname=psycotest', closed: 0>
    >>> conn.commit()
    >>> 

And now, back in ``psql``, our data is finally on disk:

.. code-block:: psql

    psycotest=# select * from author;
     authorid |       name
    ----------+------------------
            1 | Iain M. Banks
            2 | China Mieville
            3 | Frank Herbert
            4 | J.R.R. Tolkein
            5 | Susan Cooper
            6 | Madeline L'Engle
    (6 rows)


