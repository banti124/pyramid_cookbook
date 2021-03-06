The Main Function
+++++++++++++++++

Both Pyramid and Pylons have a top-level function that returns a WSGI
application. The Pyramid function is ``main`` in *pyramidapp/\_\_init\_\_.py*.
The Pylons function is ``make_app`` in *pylonsapp/config/middleware.py*. Here's
the main function generated by Pyramid's 'starter' scaffold:


.. literalinclude:: code/starter_main.py
   :linenos:

Pyramid has less boilerplate code than Pylons, so the main function subsumes
Pylons' middleware.py, environment.py, *and* routing.py modules.  Pyramid's
configuration code is just 5 lines long in the default application, while 
Pylons' is 35.

Most of the function's body deals with the Configurator (``config``).
That isn't the application object; it's a helper that will instantiate the
application for us. You pass in the settings as a dict to the constructor (line
6), call various methods to set up routes and such, and finally call
``config.make_wsgi_app()`` to get the application, which the main function
returns. The application is an instance of ``pyramid.router.Router``. (A Pylons
application is an instance of a ``PylonsApp`` subclass.)

Dotted Python names and asset specifications
============================================

Several config methods accept either an object (e.g., a module or callable) or
a string naming the object. The latter is called a *dotted Python name*. It's a
dot-delimited string specifying the absolute name of a module or a top-level
object in a module: "module", "package.module",
"package.subpackage.module.attribute".  Passing string names allows you to
avoid importing the object merely to pass it to a method. 

If the string starts with a leading dot, it's relative to some parent package.
So in this ``main`` function defined in *mypyramiapp/\_\_init\_\_.py*, the
parent package is ``mypyramidapp``.  So the name ".views" refers to
*mypyramidapp/views.py*. (Note: in some cases it can sometimes be tricky to
guess what Pyramid thinks the parent package is.)

Closely associated with this is a *static asset specification*, which names a
non-Python file or directory inside a Python package. A colon
separates the package name from the non-Python subpath:
"myapp:templates/mytemplate.pt", "myapp:static", "myapp:assets/subdir1". 
If you leave off the first part and the colon (e.g., "templates/mytemplate.pt",
it's relative to some current package.

An alternative syntax exists, with a colon between a module and an attribute:
"package.module:attribute". This usage is discouraged; it exists for
compatibility with Setuptools' resource syntax.

Configurator methods
====================

The Configurator has several methods to customize the application. Below are
the ones most commonly used in Pylons-like applications, in order by how
widely they're used.  The full list of methods is in Pyramid's `Configurator
API`_.

.. method:: add_route(...)

   Register a route for URL dispatch.

.. method:: add_view(...)

   Register a view.  Views are equivalent to Pylons' controller actions.

.. method:: scan(...)

   A wrapper for registering views and certain other things. Discussed in the
   views chapter.

.. method:: add_static_view(...)

   Add a special view that publishes a directory of static files. This is
   somewhat akin to Pylons' public directory, but see the static fiels chapter
   for caveats.

.. method:: include(callable, route_prefix=None)

   Allow a function to customize the configuration further.  This is a
   wide-open interface which has become very popular in Pyramid. It has three
   main use cases: 
   
   * To group related code together; e.g., to define your routes in a
     separate module. 

   * To initialize a third-party add-on. Many add-ons provide an include
     function that performs all the initialization steps for you.

   * To mount a subapplication at a URL prefix. A subapplication is just any
     bundle of routes, views and templates that work together. You can use this
     to split your application into logical units. Or you can write generic
     subapplications that can be used in several applications, or mount a
     third-party subapplication.

   If the add-on or subapplication has options, it will typically read them
   from the settings, looking for settings with a certain prefix and
   converting strings to their proper type. For instance, a session manager may
   look for keys starting with "session." or "thesessionmanager." as in
   "session.type". Consult the add-on's documentation to see what prefix it
   uses and which options it recognizes.

   The ``callable`` argument should be a function, a module, or a dotted Python
   name. If it resolves to a module, the module should contain an ``includeme``
   function which will be called. The following are equivalent::

      config.include("pyramid_beaker")
      
      import pyramid_beaker
      config.include(pyramid_beaker)
      
      import pyramid_beaker
      config.include(pyramid_beaker.includeme)

   If ``route_prefix`` is specified, it should be a string that will be
   prepended to any URLs generated by the subconfigurator's ``add_route``
   method. Caution: the route *names* must be unique across the main
   application and all subapplications, and ``route_prefix`` does not touch the
   names. So you'll want to name your routes "subapp1.route1" or
   "subapp1_route1" or such.
        
.. method:: add_subscriber(subscriber, iface=None)

   Insert a callback into Pyramid's event loop to customize how it processes
   requests. The Renderers chapter has an example of its use.

.. method:: add_renderer(name, factory)

   Add a custom renderer. An example is in the Renderers chapter.

.. method:: set_authentication_policy, set_authorization_policy, set_default_permission

   Configure Pyramid's built-in authorization mechanism.

Other methods sometimes used: ``add_notfound_view``, ``add_exception_view``,
``set_request_factory``, ``add_tween``, ``override_asset`` (used in theming).
Add-ons can define additional config methods by calling ``config.add_directive``.


Route arguments
===============

``config.add_route`` accepts a large number of keyword
arguments. They are logically divided into *predicate argumets* and
*non-predicate arguments*.  Predicate arguments determine whether the route matches the
current request. All predicates must succeed in order for the route to be
chosen.  Non-predicate arguments do not affect whether the route matches.

name

    [Non-predicate] The first positional arg; required. This must be a unique
    name for the route. The name is used to identify the route when registering
    views or generating URLs.

pattern

    [Predicate] The second positional arg; required. This is the URL path with
    optional "{variable}" placeholders; e.g., "/articles/{id}" or
    "/abc/{filename}.html". The leading slash is optional. By default the
    placeholder matches all characters up to a slash, but you can specify a
    regex to make it match less (e.g., "{variable:\d+}" for a numeric variable)
    or more ("{variable:.*}" to match the entire rest of the URL including
    slashes). The substrings matched by the placeholders will be available as
    *request.matchdict* in the view.

    A wildcard syntax "\*varname" matches the rest of the URL and puts it into
    the matchdict as a tuple of segments instead of a single string.  So a
    pattern "/foo/{action}/\*fizzle" would match a URL "/foo/edit/a/1" and
    produce a matchdict ``{'action': u'edit', 'fizzle': (u'a', u'1')}``.

    Two special wildcards exist, "\*traverse" and "\*subpath". These are used
    in advanced cases to do traversal on the remainder of the URL.

    XXX Should use raw string syntax for regexes with backslashes (\d) ?

request_method

    [Predicate] An HTTP method: "GET", "POST", "HEAD", "DELETE", "PUT". Only
    requests of this type will match the route. 

request_param

    [Predicate] If the value doesn't contain "=" (e.g., "q"), the request must
    have the specified parameter (a GET or POST variable). If it does contain
    "=" (e.g., "name=value"), the parameter must also have the specified value.

    This is especially useful when tunnelling other HTTP methods via
    POST. Web browsers can't submit a PUT or DELETE method via a form, so it's
    customary to use POST and to set a parameter ``_method="PUT"``. The
    framework or application sees the "_method" parameter and pretends the
    other HTTP method was requested. In Pyramid you can do this with
    ``request_param="_method=PUT``.

xhr

    [Predicate] True if the request must have an "X-Requested-With" header. Some
    Javascript libraries (JQuery, Prototype, etc) set this header in AJAX
    requests to distinguish them from user-initiated browser requests.

custom_predicates

    [Predicate] A sequence of callables which will be called to determine
    whether the route matches the request. Use this feature if none of the
    other predicate arguments do what you need. The request will match the route
    only if *all* callables return ``True``.  Each callable will receive two
    arguments, ``info`` and ``request``. ``request`` is the current request.
    ``info`` is a dict containing the following::
    
        info["match"]  =>  the match dict for the current route
        info["route"].name  =>  the name of the current route
        info["route"].pattern  =>  the URL pattern of the current route

    You can modify the match dict to affect how the view will see it. For
    instance, you can look up a model object based on its ID and put the object
    in the match dict under another key. If the record is not found in the
    model, you can return False.

Other arguments available: accept, factory, header, path_info, traverse.


.. _Configurator API: http://docs.pylonsproject.org/projects/pyramid/en/latest/api/config.html


.. include:: ../links.rst
