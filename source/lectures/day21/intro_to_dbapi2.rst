***********************************
A 20-minute introduction to DB-API2
***********************************

**An introduction to the standard interface for Pythonic database
interactions.**

History
=======

Despite the norms of SQL, individual databases have lots of differences.

Programmers don't want to have to think about implementation details for
underlying systems.

It would be nice to have a single API to hide these details.

Any package implementing this API would then be interchangeable.

Finalized in 1996, PEP 248 specified DB-API version 1.0 to fulfill this goal:

    "This API has been defined to encourage similarity between the Python
    modules that are used to access databases. By doing this, we hope to
    achieve a consistency leading to more easily understood modules, code that
    is generally more portable across databases, and a broader reach of
    database connectivity from Python."

source: http://www.python.org/dev/peps/pep-0248/

By 2001, PEP 249 brought version 2.0 of the DB-API specification, with
improvements:

* New column types were added to support all basic data types in "modern" SQL
* New API constants were added to help detect differences between
  implementations
* The semantics for calling stored procedures were clarified.
* Class-based exceptions were added to improve error handling possibilities

Discussions are currently underway to push DB-API v3.0, particularly in light
of the changes in Python 3.0

source: http://www.python.org/dev/peps/pep-0249/

Specification, not Implementation
---------------------------------

It is important to remember that PEP 249 is **only a specification**.

There is no code or package for DB-API 2.0 on it's own.

Since 2.5, the Python Standard Library has provided a
`reference implementation of the api <http://docs.python.org/2/library/sqlite3.html>`_
based on the SQLite3 RDBMS.

Before version 2.5, this package was available as ``pysqlite`` (and it still
is).

Installing Implementations
==========================

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

Some Assembly Required
----------------------

Most api packages will require that the development headers for the underlying
database system be available. Without these, the C symbols required for
communication with the db are not present and the wrapper cannot work.

**Moreover**

Some of the db api wrappers have special installation requirements.

The MS SQL package runs only on Windows and requires ``pywin32``. It is
included in versions of ``pywin32`` since v211.

The ``cx_Oracle`` package has binary installers, or can be installed from 
source using distutils::

    $ python setup.py build
    $ python setup.py install

The API
=======

**What is in the DB API?**

Globals
-------

DB-API2 implementations provide the following global values:

**apilevel**
  String constant indicating the api version ("1.0" or "2.0")

**threadsafety**
  Integer constant between 0 and 3 indicating the scope in which threads may
  safely be used

**paramstyle**
  String contstant indicating the style of marker used for parameter
  substitution in SQL expressions

These can be used to tailor your program's expectations.

A Constructor
-------------

DB API provides a constructor ``connect``, which returns a ``Connection``
object:

.. code-block:: python

    connect(parameters)

This can be considered the entry point for the module. Once you've got a
connection, everything else flows from there.

The *parameters* required and accepted by the ``connect`` constructor will
vary from implementation to implementation, since they are highly specific to
the underlying database.

A Connection
------------

Some of the methods of the ``connection`` may not be supported by all
implementations:

**.close()**
  Closes the connection to the database permanently.  Attempts to use the
  connection after calling this will raise a DB-API ``Error``.

**.commit()**
  explicitly commit any pending transactions to the databse.  The method
  should be a no-op if the underlying db does not support transactions.

**.rollback()**
  This optional method causes a transaction to be rolled back to the
  starting point.  It may not be implemented everywhere.

**.cursor()**
  returns a ``Cursor`` object which uses this ``Connection``.

A Cursor - settings
-------------------

You can use a few values to control the rows returned by the cursor:

**.arraysize**
  An integer which controls how many rows are returned at a time by 
  ``.fetchmany`` (and optionally how many to send at a time with 
  ``.executemany``) Defaults to ``1``

**.setinputsizes(sizes)**
  Used to set aside memory regions for paramters passed to an operation

**.setoutputsize(size[, column])**
  Used to control buffer size for large columns returned by an operation
  (``BLOB`` or ``LONG`` types, for example).

The final two methods may be implemented as no-ops

A Cursor - operations
---------------------

The cursor should be used to run operations on the database:

**.execute(operation[, parameters])**
  Prepares and then runs a database operation. Parameter style (seq or
  mapping) and markers are implementation specific (see ``paramstyle``)

**.executemany(operation[, seq_of_params])**
  Prepares and the runs an operations once for each set of parameters
  provided (this replaces the old v1 behavior of passing a seq to
  ``.execute``).

**.callproc(procname[, parameters])**
  Calls a stored DB procedure with the provided parameters. Returns a
  modified version of the provided parameters with ``output`` and
  ``input/output`` parameters replaced

A Cursor - attributes
---------------------

These attributes of ``Cursor`` can help you learn about the results of
operations:

**.rowcount**
  Tells how many rows have been returned or affected by the last
  operation. The number will be ``-1`` if no operation has been performed.

**.description**
  Returns a sequence of 7-item sequences describing each of the columns in
  the result row(s) returned (None if no operation has been performed):

* name
* type_code
* display_size (optional)
* internal_size (optional)
* precision (optional)
* scale (optional)
* null_ok (optional)

A Cursor - results
------------------

The return value ``.execute`` or ``.executemany`` is undefined and should not
be used.  These methods are the way to get results after an operation:

**.fetchone()**
  Returns the next row from a result set, and ``None`` when none remain.

**.fetchmany([size=cursor.arraysize])**
  Returns a sequence of *size* rows (or fewer) from a result set. An empty
  sequence is returned when no rows remain. Defaults to ``arraysize``.

**.fetchall()**
  Returns all (remaining) rows from a result set.  This behavior *may* be
  affected by ``arraysize``.

Note that each of these methods will raise a DB API ``Error`` if no operation
has been performed (or if no result set was produced)

Data Types & Constructors
-------------------------

The DB-API provides types and constructors for data:

* **Date(year, month, day)** - constructs an object holding a date value
* **Time(hour, min, sec)** - constructs an object holding a time value
* **Timestamp(y, m, d, h, min, s)** - constructs an object holding a
  timestamp

Each of the above has a corresponding **<name>FromTicks(ticks)** which
returns the same type given a single integer argument (seconds since the
*epoch*)

* **Binary(string)** - constructs an object to hold long binary string
  data
* **STRING** - a type to describe columns that hold string values
  (``CHAR``)
* **BINARY** - a type to describe long binary columns (``BLOB``, ``RAW``)
* **NUMBER** - a type to describe numeric columns
* **DATETIME** - a type to describe date/time/datetime columns
* **ROWID** - a type to describe the ``Row ID`` column in a database

SQL ``NULL`` values are represented by Python's ``None``

Exceptions
----------

The DB API specification requires implementations to create the following
hierarchy of custom Exception classes::

    StandardError
    ├──Warning
    └──Error
       ├──InterfaceError (a problem with the db api)
       └──DatabaseError (a problem with the database)
          ├──DataError (bad data, values out of range, etc.)
          ├──OperationalError (the db has an issue out of our control)
          ├──IntegrityError
          ├──InternalError
          ├──ProgrammingError (something wrong with the operation)
          └──NotSupportedError (the operation is not supported)


Aside from some custom extensions not required by the specification, that's
it.

