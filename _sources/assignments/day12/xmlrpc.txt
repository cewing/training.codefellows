*****************************************
Producing and Consuming XML-RPC in Python
*****************************************

Walk through the following exercise to get an idea of how XML-RPC services can
be generated and consumed from within a Python ecosystem.

A Simple Function-Based Service
===============================

Begin by creating a file, ``xmlrpc_server.py``.

Into it, type the following code:

.. code-block:: python

    #!/usr/bin/env python
    # -*- coding: utf-8 -*-
    from SimpleXMLRPCServer import SimpleXMLRPCServer

    address = ('localhost', 50000)
    server = SimpleXMLRPCServer(address)


    def multiply(a, b):
        """return the product of two numbers"""
        return a * b
    server.register_function(multiply)


    if __name__ == '__main__':
        try:
            print "Server running on %s:%s" % address
            print "Use Ctrl-C to Exit"
            server.serve_forever()
        except KeyboardInterrupt:
            server.server_close()
            print "Exiting"

The call to server.register_function tells the server that we want this
``multiply`` function to be available to the network.

We can run a client from a terminal. First, open one terminal and run the
xmlrpc_server.py script:

    $ python xmlrpc_server.py

Then, open another terminal and start up python:

.. code-block:: python

    >>> import xmlrpclib
    >>> proxy = xmlrpclib.ServerProxy('http://localhost:50000', verbose=True)
    >>> proxy.multiply(3, 24)
    ...
    72
    >>> 

By passing the optional ``verbose`` argument to our ServerProxy, we have
indicated that we want to see the traffic that goes back and forth.

If you format the HTTP request you sent to the service, it looks like this:

.. code-block:: http

    POST /RPC2 HTTP/1.0
    Host: localhost:50000
    User-Agent: xmlrpclib.py/1.0.1 (by www.pythonware.com)
    Content-Type: text/xml
    Content-Length: 192
    
    <?xml version='1.0'?>
    <methodCall>
     <methodName>multiply</methodName>
     <params>
      <param>
       <value><int>3</int></value>
      </param>
      <param>
       <value><int>24</int></value>
      </param>
     </params>
    </methodCall>

And the response that was returned by the service is also a simple HTTP
response with an XML body:

.. code-block:: http

    HTTP/1.0 200 OK
    Server: BaseHTTP/0.3 Python/2.6.1
    Date: Sun, 13 Jan 2013 03:38:00 GMT
    Content-type: text/xml
    Content-length: 121

    <?xml version='1.0'?>
    <methodResponse>
     <params>
      <param>
       <value><int>72</int></value>
      </param>
     </params>
    </methodResponse>

Class-Based Service
===================

It is also possible to register methods defined on a Python class as a service.

In your ``xmlrpc_server.py`` file, add the following class definition:

.. code-block:: python

    class SimpleMathService(object):
        """provide simple mathematical operations"""
        
        def __init__(self):
            self.usage = {}

            def divide(self, a, b):
                """return the quotient of two numeric values"""
                count = self.usage.setdefault('divide', 0)
                count += 1
                return a / b

        def get_usage(self):
            """show usage statistics"""
            stats = ""
            for op, ct in self.usage:
                stats += "%s: used %d times\n"
            if not stats:
                stats = "No operations have yet been used"
            return stats
        
        def _private_method(self):
            """this method is private because it starts with '_'
            """
            return "This method is not callable on the published service"

Then, comment out the earlier ``multiply`` function and the line that registers
it with the server:

.. code-block:: python

    # def multiply(a, b):
    #     return a * b
    # server.register_function(multiply)

Finally, register the new class with the server instead:

.. code-block:: python

    server.register_instance(SimpleMathService())

Back in your server terminal, quit the server and then restart it.

Then, try calling ``multiply`` again:

.. code-block:: pycon

    >>> proxy.multiply(3, 24)
    ...
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1224, in __call__
        return self.__send(self.__name, args)
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1578, in __request
        verbose=self.__verbose
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1264, in request
        return self.single_request(host, handler, request_body, verbose)
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1297, in single_request
        return self.parse_response(response)
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1473, in parse_response
        return u.close()
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 793, in close
        raise Fault(\*\*self._stack[0])
    xmlrpclib.Fault: <Fault 1: '<type \'exceptions.Exception\'>:method "multiply" is not supported'>
    >>> 

Note that the *client* has raised an error, but the *server* is still happily
running.

Now, call the new method we defined:

.. code-block:: pycon

    >>> proxy.divide(24, 3)
    send: "POST /RPC2 HTTP/1.1\r\nHost: localhost:50000\r\nAccept-Encoding: gzip\r\nUser-Agent: xmlrpclib.py/1.0.1 (by www.pythonware.com)\r\nContent-Type: text/xml\r\nContent-Length: 191\r\n\r\n<?xml version='1.0'?>\n<methodCall>\n<methodName>divide</methodName>\n<params>\n<param>\n<value><int>24</int></value>\n</param>\n<param>\n<value><int>3</int></value>\n</param>\n</params>\n</methodCall>\n"
    reply: 'HTTP/1.0 200 OK\r\n'
    header: Server: BaseHTTP/0.3 Python/2.7.5
    header: Date: Sun, 23 Feb 2014 00:28:29 GMT
    header: Content-type: text/xml
    header: Content-length: 121
    body: "<?xml version='1.0'?>\n<methodResponse>\n<params>\n<param>\n<value><int>8</int></value>\n</param>\n</params>\n</methodResponse>\n"
    8
    >>> 

Try to call the method we created as private:

.. code-block:: pycon

    >>> proxy._private_method()
    ...
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1224, in __call__
        return self.__send(self.__name, args)
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1578, in __request
        verbose=self.__verbose
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1264, in request
        return self.single_request(host, handler, request_body, verbose)
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1297, in single_request
        return self.parse_response(response)
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 1473, in parse_response
        return u.close()
      File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/xmlrpclib.py", line 793, in close
        raise Fault(\*\*self._stack[0])
    xmlrpclib.Fault: <Fault 1: '<type \'exceptions.Exception\'>:method "_private_method" is not supported'>


Again, note that the *client* raises the error, the *server* responded just
fine and continues to run.

Finally, try calling the other method we defined, to show usage statistics.
What does it give us?

.. code-block:: pycon

    >>> proxy.get_usage()
    ...
    {'divide': 1}
    >>> 

Try setting up another proxy in a third terminal.  Call both methods a few
times from each proxy and notice how the usage statistics increment.  What does
this tell you about a class-based service?


Service Introspection
=====================

XML-RPC supports introspection of a service, allowing you to get information
about the service and how it functions.

The introspection methods that are available for a Python XML-RPC server are
``listMethods``, ``methodHelp``, and ``methodSignature``.

To add support for a given introspection method to your class-based service,
you have to implement a corresponding private method on the class. Add the
following two methods to your ``SimpleMathService`` class in
``xmlrpc_server.py``:

.. code-block:: python

    def _listMethods(self):
        """custom logic for presenting method names to users
        
        list_public_methods is a convenience function from the Python 
        library, but you can make your own logic if you wish.
        """
        return list_public_methods(self)
    
    def _methodHelp(self, method):
        """provide help text for an individual method
        """
        f = getattr(self, method)
        return f.__doc__

In addition, make sure to import the ``list_public_methods`` method from the
``SimpleXMLRPCServer`` module at the top of the server script:

.. code-block:: python

    from SimpleXMLRPCServer import list_public_methods

Finally, just after creating the server, set it up to provide the public
introspection functions (still in ``xmlrpc_server.py``):


.. code-block:: python

    address = ('localhost', 50000)
    server = SimpleXMLRPCServer(address)
    server.register_introspection_functions() # this line is new

Now, restart your server and kick the tires. The introspection functions are
available as attributes of the ``system`` attribute of your proxy:

.. code-block:: pycon

    >>> proxy.system.listMethods()
    ...
    ['divide', 'get_usage', 'system.listMethods', 'system.methodHelp', 'system.methodSignature']
    >>> proxy.system.methodHelp('divide')
    ...
    'return the quotient of two numeric values'
    >>> proxy.system.methodSignature('divide')
    ...
    'signatures not supported'
    >>> 

**Warning**, try it, but I've been unable to get system.methodSignature to work.


