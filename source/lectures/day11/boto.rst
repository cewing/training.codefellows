************************************
Managing AWS EC2 Instances with Boto
************************************

Amazon Web Services provides an API for interacting directly with cloud
resources from the command line. Using this API allows you to automate
deployment tasks.

Automated tasks are easy.

Easy tasks get done more often.

Therefore, using the API means you can and will perform deployment tasks like
setting up and running a testing environment with a much higher frequency. This
will improve the quality of your work.

AWS in Python
=============

`Boto`_ is the Python binding of the AWS API. It's a solid, well-built package
that provides control over most of the available services in AWS.

.. _Boto: http://boto.readthedocs.org/

Today, we'll spend some time learning a basic interaction with AWS EC2 using
boto.

Practice Safe Development
-------------------------

To get started, let's install boto in a virtual environment:

.. code-block:: pycon

    $ mkproject bototests
    New python executable in bototests/bin/python
    Installing setuptools, pip...done.
    Creating /Users/cewing/projects/bototests
    Setting project for bototests to /Users/cewing/projects/bototests
    [bototests]
    heffalump:bototests cewing$ pip install boto
    Downloading/unpacking boto
      Downloading boto-2.25.0-py2.py3-none-any.whl (1.1MB): 1.1MB downloaded
    Installing collected packages: boto
    Successfully installed boto
    Cleaning up...
    [bototests]
    heffalump:bototests cewing$ pwd
    /Users/cewing/projects/bototests
    [bototests]
    heffalump:bototests cewing$

Configuration
-------------

Next, we want to configure boto so that it has access to the security
credentials it needs.

Boto will look for configuration in a configuration file in your home
directory.  This file is called ``.boto``. Create it, if it doesn't exist:

.. code-block:: bash

    $ ls ~/.b*
    /Users/cewing/.bash_history

    /Users/cewing/.buildout:
    default.cfg downloads   eggs        extends

    /Users/cewing/.bundler:
    tmp
    [bototests]
    heffalump:bototests cewing$ touch ~/.boto

Open this file in your text editor and add the following lines:

.. code-block:: ini

    [Credentials]
    aws_access_key_id = YOURACCESSKEY
    aws_secret_access_key = YOURSECRETKEY

You will want to secure that file from easy access by making it readable and
writable only by yourself:

.. code-block:: bash

    [bototests]
    heffalump:bototests cewing$ chmod 600 ~/.boto


Create Your First EC2 Instance
==============================

You are ready now to create your first instance.

Getting Connected
-----------------

First, we are going to make a connection to the EC2 service.  When we do so, we
have to designate the AWS region to which we are connecting.  All AWS resources
are tied to a region in some fashion.

We'll set up stream logging so that we can see more information about what is
happening:

.. code-block:: pycon

    [bototests]
    heffalump:bototests cewing$ python
    ...
    >>> import boto
    >>> boto.set_stream_logger('boto')
    >>> import boto.ec2
    >>> ec2 = boto.ec2.connect_to_region('us-west-2')
    2014-02-14 17:32:56,641 boto [DEBUG]:Using access key found in config file.
    2014-02-14 17:32:56,641 boto [DEBUG]:Using secret key found in config file.
    >>> ec2
    EC2Connection:ec2.us-west-1.amazonaws.com
    >>> 

The next step is to find an AMI that you want to use. AMIs are machine images
that Amazon uses in order to create a cloud server of a particular type.

I generally use Ubuntu Linux when creating cloud servers with AWS, and I like
to choose images from `Alestic`_, which is sort of the 'official' face of Ubuntu
in EC2.

.. _Alestic: http://alestic.com

At the top right of the Alestic homepage is a tool for finding AMI ids in a
given AWS region.  We've connected to us-west-2, so we need one for that
region. The tool reports that an EBS-store image for Ubuntu 12.04 Precise is
'ami-d0d8b8e0'. Let's use that.

.. code-block:: pycon

    >>> image_id = 'ami-d0d8b8e0'

We also need to designate a key pair name and the name of a security group. Use
the key pair name and security group you created as part of the assignment to
get an AWS account. If you followed the instructions explicitly, these should
be ``pk-aws`` and ``ssh-access``.

.. code-block:: pycon

    >>> key_pair = 'pk-aws'
    >>> security_group = 'ssh-access'

Finally, we need to designate exactly what type of instance to create. AWS
instances come in all shapes and sizes, but the only on that is in the free
usage tier is the ``t1.micro`` instance.

.. code-block:: pycon

    >>> instance_type = 't1.micro'

Starting an Instance
--------------------

Finally we are ready to run an instance. Using your open ec2 connection object,
run the following command:

.. code-block:: pycon

    >>> reservations = ec2.run_instances(
    ...     image_id,
    ...     key_name=key_name,
    ...     instance_type=instance_type,
    ...     security_groups=[security_group])
    >>> 

When the command returns, ``reservations`` will hold a list of all the
instances we have just created (there should be only one). We can pull that
instance out and check its status:

.. code-block:: pycon

    >>> reservations
    Reservation:r-9f78c096
    >>> reservations.instances
    [Instance:i-d0d558d9]
    >>> len(reservations.instances)
    1
    >>> instance = reservations.instances[0]
    >>> instance
    Instance:i-d0d558d9>>> instance.id
    u'i-d0d558d9'
    >>> instance.state
    u'pending'
    >>> instance.update()
    2014-02-14 22:14:00,660 boto [DEBUG]:Method: POST
    ...
    >>> instance.state
    u'running'

You may need to update a couple of times until you see the state change to
``running``. Once it does, you can get the public DNS name of the instance:

.. code-block:: pycon

    >>> instance.public_dns_name
    u'ec2-54-203-88-113.us-west-2.compute.amazonaws.com'

SSH Into the New Instance
-------------------------

You can use that DNS name to ssh into the running instance.  **In another
terminal**, run the following command:

.. code-block:: bash

    $ ssh -i ~/.ssh/pk-aws.pem ubuntu@ec2-your.dns.name.amazonaws.com
    The authenticity of host 'ec2-54-203-88-113.us-west-2.compute.amazonaws.com (54.203.88.113)' can't be established.
    RSA key fingerprint is 56:3e:9c:b3:75:96:4f:11:44:e9:2b:14:3a:02:f8:f2.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added 'ec2-54-203-88-113.us-west-2.compute.amazonaws.com,54.203.88.113' (RSA) to the list of known hosts.
    Welcome to Ubuntu 12.04.4 LTS (GNU/Linux 3.2.0-58-virtual x86_64)

     * Documentation:  https://help.ubuntu.com/

      System information as of Sat Feb 15 06:20:44 UTC 2014

      System load:  0.0               Processes:           58
      Usage of /:   11.1% of 7.87GB   Users logged in:     0
      Memory usage: 8%                IP address for eth0: 10.235.47.92
      Swap usage:   0%

      Graph this data and manage this system at:
        https://landscape.canonical.com/

      Get cloud support with Ubuntu Advantage Cloud Guest:
        http://www.ubuntu.com/business/services/cloud

    0 packages can be updated.
    0 updates are security updates.


    The programs included with the Ubuntu system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
    applicable law.

    To run a command as administrator (user "root"), use "sudo <command>".
    See "man sudo_root" for details.

    ubuntu@ip-10-235-47-92:~$

At this point, you are now working in a shell *on the server you just created*!

Go ahead and run a few simple shell commands and look around a bit.  Not much
to see, but it's fun to prove that you're there.

Finally, disconnect by typing the ``exit`` command and return to your Python
interpreter.

Stop the Instance
-----------------

Okay, that's enough for one day. Now it's time to clean up our toys. Let's
begin by requesting that our instance be stopped:

.. code-block:: pycon

    >>> instance.stop()
    2014-02-14 22:25:25,012 boto [DEBUG]:Method: POST
    ... 
    >>> instance.state
    u'stopping'
    >>> instance.update()
    2014-02-14 22:28:55,768 boto [DEBUG]:Method: POST
    ... 
    >>> instance.state
    u'stopped'

Once the instance is cleanly stopped, we can terminate it, which will
completely destroy it and leave us ready to play again another day:

.. code-block:: pycon

    >>> instance.terminate()
    2014-02-14 22:31:06,801 boto [DEBUG]:Method: POST
    ... 
    >>> instance.state
    u'terminated'
    >>> 


Wrap-up
=======

Boto, the Python wrapper for the AWS API allows us to automate the management
of cloud resources. This type of automation makes the typical tasks of creating
deployments (whether to production, staging or testing) easy and quick. This in
turn lowers the bar to doing what we should all be doing, deploying
consistently and often.

There's much more to learn about AWS and boto, but that's all we have time for
for now.

Please read more in the `boto documentation`_.

.. _boto documentation: http://boto.readthedocs.org/
