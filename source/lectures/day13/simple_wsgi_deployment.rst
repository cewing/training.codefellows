******************************************
Deploying a Simple WSGI Application on AWS
******************************************

Let's walk through manually deploying a simple WSGI application to an AWS EC2
instance we've created.

For this exercise we'll use the fabfile you set up as part of the fabric
walkthrough.


Setup
=====

.. code-block:: bash

    heffalump:~ cewing$ workon fabrictests
    [fabrictests]
    heffalump:fabrictests cewing$

In the fabric directory, create a very simple WSGI application. Start by
creating a new file ``myapp.py`` and opening it in your editor. In the file,
type the following Python code:

.. code-block:: python

    # -*- coding: utf-8 -*-
    def app(environ, start_response):
        data = "Hello, World!\n"
        start_response("200 OK", [
            ("Content-Type", "text/plain"),
            ("Content-Length", str(len(data)))
        ])
        return iter([data])

    if __name__ == '__main__':
        from wsgiref.simple_server import make_server
        srv = make_server('localhost', 8080, app)
        srv.serve_forever()

Test your application to be sure it works by running it from the command line:

.. code-block:: bash

    source

Then, if you don't already have one running, provision an AWS instance and
install ``nginx``, using your fabfile:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab provision_instance
    ...
    [fabrictests]
    heffalump:fabrictests cewing$ fab install_nginx
    [localhost] Executing task 'install_nginx'
    Connected to EC2 region us-west-2
    Please select from the following instances:
     1: running instance i-08b12201
    Choose an instance: 1
    [ubuntu@ec2-54-184-162-20.us-west-2.compute.amazonaws.com] Executing task '_install_nginx'
    ...

Next, use the shell command ``scp`` to securely copy your wsgi application to
the new server instance:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ scp -i ~/.ssh/pk-cpe.pem myapp.py ubuntu@ec2-54-184-162-20.us-west-2.compute.amazonaws.com:~/
    ...
    Are you sure you want to continue connecting (yes/no)? yes
    ...
    myapp.py                                      100%  381     0.4KB/s   00:00
    [fabrictests]
    heffalump:fabrictests cewing$


Manual Configuration
====================

We are going to do this manually once together.  Afterwards, take what you
learned and automate these tasks using ``fabric``.

Configure Nginx
---------------

The first step is to configure ``nginx`` to proxy HTTP requests to our simple
application.

If you know, or are more comfortable with the ``Apache`` webserver, you can
also use that. However, the trend I've seen over the last few years is toward
the use of ``nginx`` over the old stand-by.

Nginx stores site configuration on Ubuntu in the ``/etc/nginx/sites-available``
directory.

Let's shell into our new instance and look at what's there:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ ssh -i ~/.ssh/pk-cpe.pem ubuntu@ec2-54-184-162-20.us-west-2.compute.amazonaws.com
    Welcome to Ubuntu 12.04.4 LTS (GNU/Linux 3.2.0-58-virtual x86_64)
    ...
    Last login: Wed Feb 26 19:10:01 2014 from 199.231.242.170
    ubuntu@ip-10-254-159-140:~$ ls /etc/nginx/sites-available/
    default
    ubuntu@ip-10-254-159-140:~$ more /etc/nginx/sites-available/default
    # You may add here your
    # server {
    #   ...
    # }
    # statements for each of your virtual hosts to this file
    ...

    ubuntu@ip-10-254-159-140:~$

The ``sites-available`` directory will hold individual site configuration for
**all** sites that **might** be available on a server.

**Active** site configuration is listed in the ``/etc/nginx/sites-enabled``:

.. code-block:: bash

    ubuntu@ip-10-254-159-140:~$ ls /etc/nginx/sites-enabled/
    default
    ubuntu@ip-10-254-159-140:~$ ls -l /etc/nginx/sites-enabled/
    total 0
    lrwxrwxrwx 1 root root 34 Feb 26 19:09 default -> /etc/nginx/sites-available/default
    ubuntu@ip-10-254-159-140:~$

Notice that in fact, although ``default`` is in that directory too, it's
actually a soft link to the file in ``sites-available``.

Let's move aside the existing ``default`` config and replace it with a simple
one of our own.

On your local machine, in the ``fabtests`` directory, make a new file
``simple_nginx_config``. Open that file in your editor and add the following:

.. code-block:: nginx

    server {
        listen 80;
        server_name http://ec2-54-184-162-20.us-west-2.compute.amazonaws.com/;
        access_log  /var/log/nginx/test.log;

        location / {
            proxy_pass http://127.0.0.1:8000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

Now, copy that file up to your server too:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ scp -i ~/.ssh/pk-cpe.pem simple_nginx_config ubuntu@ec2-54-184-162-20.us-west-2.compute.amazonaws.com:~/
    simple_nginx_config                           100%  363     0.4KB/s   00:00
    [fabrictests]
    heffalump:fabrictests cewing$

Next, on the server, move the original default configuration file aside and put
your new one in its place:

.. code-block:: bash

    ubuntu@ip-10-254-159-140:~$ ls
    myapp.py  simple_nginx_config
    ubuntu@ip-10-254-159-140:~$ sudo mv /etc/nginx/sites-available/default /etc/nginx/sites-available/default.orig
    ubuntu@ip-10-254-159-140:~$ sudo mv simple_nginx_config /etc/nginx/sites-available/default
    ubuntu@ip-10-254-159-140:~$ ls -l /etc/nginx/sites-enabled/
    total 0
    lrwxrwxrwx 1 root root 34 Feb 26 19:09 default -> /etc/nginx/sites-available/default
    ubuntu@ip-10-254-159-140:~$

Once that's complete, you can restart nginx to have it pick up your changes:

.. code-block:: bash

    ubuntu@ip-10-254-159-140:~$ sudo /etc/init.d/nginx restart
    Restarting nginx: nginx: [warn] server name "http://ec2-54-184-162-20.us-west-2.compute.amazonaws.com/" has suspicious symbols in /etc/nginx/sites-enabled/default:3
    nginx.
    ubuntu@ip-10-254-159-140:~$

If you now try to load the public DNS name for your EC2 instance, you'll see
that nginx has updated and is now throwing an error::

    http://ec2-54-184-162-20.us-west-2.compute.amazonaws.com

This should tell you **Bad Gateway**.  That's the error that means "I am a
proxy, but the thing I'm proxying to is not running!"

Running a WSGI Server
---------------------

Let's make our wsgi app run, so we can fix that.

On your server, run the wsgi app:

.. code-block:: bash

    ubuntu@ip-10-254-159-140:~$ python myapp.py

And now reload your web browser and verify that you can see "Hello, World!"


Automation
==========

The steps we took there allowed us to upload an application and some
configuration to our server, apply the configuration to the ``nginx`` web
server we installed, and then run our WSGI application in a terminal to get a
response via public DNS.

Your next task is to automate this process using fabric.

For this task, You'll want to look at two important sub-packages in ``fabric``:

* `fabric.contrib.files`_
* `fabric.contrib.project`_

.. _fabric.contrib.files: http://docs.fabfile.org/en/1.8/api/contrib/files.html
.. _fabric.contrib.project: http://docs.fabfile.org/en/1.8/api/contrib/project.html
