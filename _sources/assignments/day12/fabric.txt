***************************
Task Automation with Fabric
***************************

.. note:: For this tutorial, you will want to have the `documentation for fabric`_ at hand

.. _documentation for fabric: http://docs.fabfile.org/en/1.8/index.html

We've looked at creating and destroying new AWS EC2 instances using the Python
wrapper for the AWS API, ``boto``.

Once you have a server, you want to do something useful with it. Possible tasks
might include:

* installing system packages you require to support your application
* generating configuration for those packages
* installing your application software
* ...

Manually performing these tasks is tedious. Once you have figured out how to
configure a system properly for your application, why would you want to do it
over and over again?

Instead, you should automate these tasks.

Automation ensures that you can accomplish a task with a single command, which
encourages actually doing that task more often.

One package available for Python to accomplish this sort of task automation is
`fabric`_. It provides a uniform interface for performing tasks either locally
or on one or more remote server.

.. _fabric: http://docs.fabfile.org/en/latest/

``Fabric`` uses SSH to handle communications between your local machine and the
system you are working on, so any command you can run from a terminal is
available to be automated.

In this quick tutorial, we'll walk through the process of automating the
provisioning of a new AWS EC2 instance and then installing some software on the
new system.

Setting Up
==========

To begin with, let's create an environment suitable for exploring ``fabric``
and using ``boto`` to set up remote servers.

Begin by creating a new virtualenv project:

.. code-block:: bash

    heffalump:~ cewing$ mkproject fabrictests
    New python executable in fabrictests/bin/python
    Installing setuptools, pip...done.
    Creating /Users/cewing/projects/fabrictests
    Setting project for fabrictests to /Users/cewing/projects/fabrictests
    [fabrictests]
    heffalump:fabrictests cewing$

Once that's handled, let's install *both* boto *and* fabric:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ pip install boto fabric
    Downloading/unpacking boto
      Downloading boto-2.25.0-py2.py3-none-any.whl (1.1MB): 1.1MB downloaded
    Downloading/unpacking fabric
      Downloading Fabric-1.8.2.tar.gz (251kB): 251kB downloaded
    ...
    ... (lots of c-compilation stuff)
    Successfully installed boto fabric paramiko pycrypto ecdsa
    Cleaning up...
    [fabrictests]
    heffalump:fabrictests cewing$

Notice that fabric, since it operates over SSH, requires the Python ssh library
``paramiko``.

Because ``fabric`` will use SSH to connect to **all** hosts, including your
local machine, you'll need to enable SSH access for your computer.


One More Thing
==============

We are going to set up an HTTP server on our ubuntu instance.  In order to
prove that we've done it, we need to be able to connect to that server via HTTP
(port 80).

Before you go any further, open your AWS web console. Navigate to the EC2
dashboard, and then select the 'Security Groups' controls.

Find your ``ssh-access`` security group and add a new TCP rule. Allow HTTP
access via port 80 from any address.

Don't forget to apply your rules changes after adding the new rule, or it won't
take effect.


Fabric Basics
=============

``Fabric`` operates by providing a single command ``fab``.

This command looks for a file in your present working directory called
``fabfile`` or ``fabfile.py``. If there is none, it will throw an error:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab

    Fatal error: Couldn't find any fabfiles!

    Remember that -f can be used to specify fabfile path, and use -h for help.

    Aborting.
    [fabrictests]
    heffalump:fabrictests cewing$

Let's start by creating a new fabfile here in the ``fabrictests`` project
directory:

.. code-block:: bash

    fabrictests]
    heffalump:fabrictests cewing$ touch fabfile.py
    [fabrictests]
    heffalump:fabrictests cewing$

What goes into that file?

Python code.

Let's start by creating a very simple test Python function we can use to show
how fabric works in general.

Open ``fabfile.py`` in your editor and add the following code:

.. code-block:: python

    from fabric.api import run

    def host_type():
        run('uname -s')

Once you've saved this, the name of the new Python function becomes available
as a ``subcommand`` for the ``fab`` command:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab -H localhost host_type
    [localhost] Executing task 'host_type'
    [localhost] run: uname -s
    [localhost] Login password for 'cewing':
    [localhost] out: Darwin
    [localhost] out:


    Done.
    Disconnecting from localhost... done.
    [fabrictests]
    heffalump:fabrictests cewing$

Notice that you're local machine has prompted you to login with a password.
That's a good thing. If you `enable publickey login`_ then ``fabric`` will skip
this part for you. This also means that you won't be sending your password
across a connection, so that's nice.

.. _enable publickey login: http://stackoverflow.com/questions/7260/how-do-i-setup-public-key-authentication


The Fabric Environment
----------------------

Having to specify a host to connect to is a bit awkward, so ``fabric`` provides
a mechanism for setting this up automatically.

The ``fabric.api`` module provides a symbol ``env`` that allows you to share
state between different fabric tasks.  This state can be mutated locally, both
at module level and from within a given task.

You can hang pretty much any symbol onto this object, which is really just a
fancy Python dictionary.

However, there are a number of environmental settings that are present from the
get-go, and which ``fabric`` will refer to automatically.

One of these is ``hosts``.  When we invoked the ``fab`` command with ``-H
localhost`` we were pre-populating this env attribute with the string
'localhost'.  We can do this manually.  Update your ``fabfile.py`` as follows:

.. code-block:: python

    from fabric.api import run
    from fabric.api import env

    env.hosts = ['localhost', ]

    def host_type():
        run('uname -s')


Save those changes and you should now be able to run the ``host_type``
subcommand without providing the host explicitly:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab host_type
    [localhost] Executing task 'host_type'
    [localhost] run: uname -s
    [localhost] Login password for 'cewing':
    [localhost] out: Darwin
    [localhost] out:


    Done.
    Disconnecting from localhost... done.
    [fabrictests]
    heffalump:fabrictests cewing$


Fabric with Boto
================

Our interactions with ``boto`` are all oriented around creating and destroying
servers.

In fabric, our orientation will be toward manipulating *a server* once it
exists.

Before we do that, though we need to get a server running.

Any attempt we make at dealing with AWS EC2 will require that we have a
connection, so our first step should be to make a connection to the region we
want to use.  Let's add a function to do this to our ``fabfile.py``

.. code-block:: python

    # add an import
    import boto.ec2

    # add an environmental setting
    env.aws_region = 'us-west-2'

    # add a function
    def get_ec2_connection():
        if 'ec2' not in env:
            conn = boto.ec2.connect_to_region(env.aws_region)
            if conn is not None:
                env.ec2 = conn
                print "Connected to EC2 region %s" % env.aws_region
            else:
                msg = "Unable to connect to EC2 region %s"
                raise IOError(msg % env.aws_region)
        return env.ec2


We can call fabric functions from other fabric functions, so this will be of
use to us as we try to provision, manage and deprovision our servers.

Provisioning an Instance
------------------------

In the ``boto`` walkthrough, we saw how to provision a new instance and set it
running.  Let's convert that code to a fabric task.

We'll need to start by getting a connection, then use that connection to create
a new instance.

.. code-block:: python
    :linenos:

    # add an import at the top
    import time

    # add this function
    def provision_instance(wait_for_running=False, timeout=60, interval=2):
        wait_val = int(interval)
        timeout_val = int(timeout)
        conn = get_ec2_connection()
        instance_type = 't1.micro'
        key_name = 'pk-cpe'
        security_group = 'ssh-access'
        image_id = 'ami-d0d8b8e0'

        reservations = conn.run_instances(
            image_id,
            key_name=key_name,
            instance_type=instance_type,
            security_groups=[security_group, ]
        )
        new_instances = [i for i in reservations.instances if i.state == u'pending']
        running_instance = []
        if wait_for_running:
            waited = 0
            while new_instances and (waited < timeout_val):
                time.sleep(wait_val)
                waited += int(wait_val)
                for instance in new_instances:
                    state = instance.state
                    print "Instance %s is %s" % (instance.id, state)
                    if state == "running":
                        running_instance.append(
                            new_instances.pop(new_instances.index(i))
                        )
                    instance.update()


There's something new in this function.  It accepts arguments. We can pass
arguments like this from the command-line when we run the command:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab provision_instance

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab provision_instance:wait_for_running=1

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab provision_instance:wait_for_running=1,interval=5

The important part to know about this is that arguments passed in from the
command line appear in your function as strings.  You are responsible for
converting them if needed.

Let's provision a new server with this function:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab provision_instance:wait_for_running=1
    [localhost] Executing task 'provision_instance'
    Connected to EC2 region us-west-2
    Instance i-8c424a85 is pending
    Instance i-8c424a85 is pending
    Instance i-8c424a85 is pending
    Instance i-8c424a85 is pending
    Instance i-8c424a85 is pending
    Instance i-8c424a85 is pending
    Instance i-8c424a85 is pending
    Instance i-8c424a85 is running

    Done.
    [fabrictests]
    heffalump:fabrictests cewing$

Once the function completes, you should be able to load your EC2 dashboard and
see the new instance you just created running happily.


Running Shell Commands on a Server
==================================

Now that we have an instance running, we want to run commands on it to
configure it.

Fabric provides the ``run`` command to support this.

But there's an issue.  When you execute a function using fabric, it is actually
run repeatedly, once for each ``host`` you list in the ``env.hosts``
environment setting.

At the moment, our script lists 'localhost', but we don't actually want to run
this command on 'localhost', so we need to get around this.

Fabric provides the ``execute`` command specifically for this purpose.  We can
pass it the list of hosts on which we want to run a particular command, and it
will do so for us.

In order to play with that, though, we need to interactively select the
instance we want to use (after all, we might have more than one, right?).

We'll begin by building a function that allows us to list instances, optionally
filtering for a particular state:


.. code-block:: python

    def list_aws_instances(verbose=False, state='all'):
        conn = get_ec2_connection()

        reservations = conn.get_all_reservations()
        instances = []
        for res in reservations:
            for instance in res.instances:
                if state == 'all' or instance.state == state:
                    instance = {
                        'id': instance.id,
                        'type': instance.instance_type,
                        'image': instance.image_id,
                        'state': instance.state,
                        'instance': instance,
                    }
                    instances.append(instance)
        env.instances = instances
        if verbose:
            import pprint
            pprint.pprint(env.instances)

If we run this command from the command line, we should see our running
instance:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab list_aws_instances:verbose=1,state=running
    [localhost] Executing task 'list_aws_instances'
    Connected to EC2 region us-west-2
    [{'id': u'i-ab5159a2',
      'image': u'ami-d0d8b8e0',
      'instance': Instance:i-ab5159a2,
      'state': u'running',
      'type': u't1.micro'}]

    Done.
    [fabrictests]
    heffalump:fabrictests cewing$

Nice.

Now we can create a function that will take the output of that function and
give us a choice of running instances on which to take a given action:

.. code-block:: python

    # add an import
    from fabric.api import prompt

    def select_instance(state='running'):
        if env.active_instance:
            return

        list_aws_instances(state=state)

        prompt_text = "Please select from the following instances:\n"
        instance_template = " %(ct)d: %(state)s instance %(id)s\n"
        for idx, instance in enumerate(env.instances):
            ct = idx + 1
            args = {'ct': ct}
            args.update(instance)
            prompt_text += instance_template % args
        prompt_text += "Choose an instance: "

        def validation(input):
            choice = int(input)
            if not choice in range(1, len(env.instances) + 1):
                raise ValueError("%d is not a valid instance" % choice)
            return choice

        choice = prompt(prompt_text, validate=validation)
        env.active_instance = env.instances[choice - 1]['instance']

The ``fabric.api.prompt`` function allows us to use an interaction like
``raw_input`` to ask for input from the fabric user. Here, we build a list of
the available instances that are 'running', and then ask the user to choose
among them. After a valid choice is made, we can then hang that instance on our
``env`` so that we can access it from other functions. We can also
short-circuit this behavior by checking first to see if we've already selected
an instance.

Noe, let's set up a function that will run a given command on the server we
select:

.. code-block:: python

    # add an import
    from fabric.api import execute

    def run_command_on_selected_server(command):
        select_instance()
        selected_hosts = [
            'ubuntu@' + env.active_instance.public_dns_name
        ]
        execute(command, hosts=selected_hosts)

This function begins by asking the user to select an instance.  Then it sets up
a list of hosts that includes that server (with the username 'ubuntu'
hard-coded). Finally, it attempts to execute the command we provide as an
argument on the list of hosts we just built.

In order for us to run a command on this server, we need to ensure that the
keypair we set up for our AWS instances is available to the ssh agent. We can
do that at the system level on our local machine:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ ssh-add ~/.ssh/pk-aws.pem
    Identity added: /Users/cewing/.ssh/pk-cpe.pem (/Users/cewing/.ssh/pk-cpe.pem)
    [fabrictests]
    heffalump:fabrictests cewing$

Now our local ssh agent knows that this is a file we might use to connect. When
the AWS server asks us to use public-key authentication, the agent will then
try this key along with any others the agent knows about. If no known key
works, ssh will bomb out.

``Execute`` will take the name of a fabric command to run. We need a command
that it will run that will install nginx for us and then start it.

.. code-block:: python

    # add an import
    from fabric.api import sudo

    def _install_nginx():
        sudo('apt-get install nginx')
        sudo('/etc/init.d/nginx start')

Finally, we need to wrap this function in a function we might call from the
command line that will run it on the server we select:

.. code-block:: python

    def install_nginx():
        run_command_on_selected_server(_install_nginx)

Now, if we run this fabric command from our command line, we can get nginx installed on our AWS instance:

.. code-block:: bash

    [fabrictests]
    heffalump:fabrictests cewing$ fab install_nginx
    [localhost] Executing task 'install_nginx'
    Connected to EC2 region us-west-2
    Please select from the following instances:
     1: running instance i-ab5159a2
    Choose an instance: 1
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] Executing task '_install_nginx'
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] sudo: apt-get install nginx
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out:
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out: Reading package lists... 0%
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out:
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out: Reading package lists... 0%
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out:
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out: Reading package lists... 24%
    ...
    ...
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out: Setting up nginx-full (1.1.19-1ubuntu0.5) ...
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out: Setting up nginx (1.1.19-1ubuntu0.5) ...
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out: Processing triggers for libc-bin ...
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out: ldconfig deferred processing now taking place
    [ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com] out:


    Done.
    Disconnecting from ubuntu@ec2-54-185-44-188.us-west-2.compute.amazonaws.com... done.
    [fabrictests]
    heffalump:fabrictests cewing$


At this point, you should be able to open a web browser and point it at your
EC2 instance and see the default nginx web page.  Remember, the public_dns_name
of your instance is the way to get at it, so use that in your browser.

Stopping and Terminating an Instance
====================================

Challenge yourself to add two more functions:

1. Select a running instance to stop, and then stop it with boto
2. Select a stopped instance to terminate, and then terminate it with boto

Good Luck!
