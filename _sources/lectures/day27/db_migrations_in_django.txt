********************************
Database Migrations with *South*
********************************

In any non-trivial software project involving a database, the Database schema
*will change over time*.

This basic fact of life spurs the need for a tool to handle gracefully the
problem of altering a database when you can have copies in production, on
staging servers, and in developer's local machines.

You've seen how Django supports adding new tables to a database using the
``syncdb`` command. But this management command only creates tables that *do
not yet exist*. It **does not update tables**.

The ``sqlclear <appname>`` command will print the ``DROP TABLE`` statements to
remove the tables for your app.

And ``sql <appname>`` will show the ``CREATE TABLE`` statements, and you can work
out the differences and update manually.

But given existing data, multiple copies of the database and multiple
developers working on different features simultaneously, that's not a great
recipe.

Luckily, there is an app available for Django that helps with this: ``South``.

South allows you to incrementally update your database in a simplified way.

South supports forward, backward and data migrations.

South is ``Alembic`` for the Django ORM.

Using South
===========

South is so useful, that in Django 1.7 it will become part of the core
distribution of Django.

But now it is not.  You'll need to add it, and set up your project to use it.

Install South in the virtualenv you are using for your project:

.. code-block:: bash

    $ workon myproject
    (myproject)$ pip install south
    ...
    Successfully installed south
    Cleaning up...

Like other Django apps, South provides models of its own.  You need to enable
them.

First, add ``south`` to the list of installed apps in ``settings.py``:

.. code-block:: python

    INSTALLED_APPS = (
        ...
        'south', #< -add this line
        'myapp',
    )

Then, run ``syncdb`` to pick up the tables it provides:

.. code-block:: bash

    (myproject)$ python manage.py syncdb
    Syncing...
    Creating tables ...
    Creating table south_migrationhistory
    ...

    Synced:
     ...
     > south
     > myapp

    Not synced (use migrations):
     -
    (use ./manage.py migrate to migrate these)


You might have noticed that the output from ``syncdb`` looks a bit different
this time. This is because Django apps that use South do not use the normal
``syncdb`` command to initialize their SQL.

Instead they use a new command that South provides: ``migrate``.

This command ensures that only incremental changes are made, rather than
creating all of the SQL for an app every time.

Enabling South for an App
-------------------------

If you notice, your app is still in the ``sync`` list. It has not been enabled
to use South.  You'll need to set up South for it.

Adding South to an existing Django app is quite simple. The trick is to do it
**before** you make any new changes to your models.  It is **vitally
important** that your existing database schema exactly match your models,
or things will go badly for you.

If that's all set, then simply use the ``convert_to_south`` management command,
providing the name of your app as an argument:

    .. code-block:: bash

        (djangoenv)$ python manage.py convert_to_south myapp
        ...

After running this command, South will automatically create a first migration
for you that sets up tables looking exactly like what your app has now:

.. code-block:: bash

    myblog/
    ├── __init__.py
    ...
    ├── migrations
    │   ├── 0001_initial.py
    │   ├── 0001_initial.pyc
    │   ├── __init__.py
    │   └── __init__.pyc
    ├── models.py
    ...

South also automatically applies this first migration using the ``--fake``
argument, since the database is already in the proposed state.

Creating Migrations
-------------------

After setting up migrations, you'll make some changes to your database schema.
To get these changes into your database, you have to add a migration.

You use the ``schemamigration`` management command to do so:

.. code-block:: bash

    (djangoenv)$ python manage.py schemamigration myapp --auto
     + Added model myapp.OtherModel
    Created 0002_auto__add_othermodel.py. You can now apply this
    migration with: ./manage.py migrate myapp

Applying Migrations
-------------------

And south, along with making the migration, helpfully tells us what to do next:

.. code-block:: bash

    (djangoenv)$ python manage.py migrate myapp
    Running migrations for myblog:
     - Migrating forwards to 0002_auto__add_othermodel.
     > myapp:0002_auto__add_othermodel
     - Loading initial data for myapp.
    Installed 0 object(s) from 0 fixture(s)

You can even look at the migration file you just applied,
``myapp/migrations/0002_auto__add_othermodel.py`` to see what happened.

**An Important Note About South Migrations**

When you look at the migration file, you'll find that South schema migrations
are in fact Python class objects. They have a ``forwards`` and ``backwards``
method that do what you might expect. But they also have a ``models`` attribute
that represents the state the models in your app should be in when the
migration is complete.

When you have multiple developers working on a project, each of them creating
migrations, it's fairly easy to merge git branches and end up with successive
database migrations that have *different* endpoint states.  This can cause
problems with generating the next migration because South *does not introspect
your database* to discover what really exists.

I strongly encourage you to `read about team workflow`_ when working with
South. In particular the documentation on using *empty migrations* to establish
a fully correct ORM model after merging changes is quite important and useful.

.. _read about team workflow: http://south.readthedocs.org/en/latest/tutorial/part5.html


Data Migrations
---------------

Another use for migrations is to alter data in an existing database.  South
calls these types of migrations *data migrations* and provides an alternate
method for creating them:

.. code-block:: bash

    (djangoenv)$ python manage.py datamigration myapp some_data_changes

This will create a new file in your ``migrations`` directory called
"NNNN_some_data_changes.py". As with a schemamigration file, this file will
have a class instance that has a ``forwards`` and ``backwards`` method.

You will have to write the code for your forwards migration, using the ``orm``
passed in to the argument to access your app models and the Django ORM query
api to get hold of and update records.

You can also write the ``backwards`` method.  But in some cases this type of
reverse migration may be impossible (think of hashing passwords for the classic
irreversable migration).  In that case, you can raise a Python exception like
``RuntimeError``.

Applying a data migration is accomplished in the same way as any other
migration.  Simply use the ``migrate`` management command:

.. code-block:: bash

    (djangoenv)$ python manage.py migrate myapp

