*******************************
An Introduction to Django Views
*******************************

In Django, the code that builds a page that you can see is called a *view*.

A *view* can be defined as a *callable* that takes a request and returns a
response.

This should sound pretty familiar to you. It is much like the way Flask defines
views.  The biggest difference is that in Flask, the ``request`` is a *local
global* you import, where in Django is is explicitly passed as an argument to
the view callable.

Classically, Django views were functions. Version 1.3 added support for
Class-based Views (a class with a ``__call__`` method is a callable). We are
going to start with functional views.

Apps as URLs
============

An app (and the functionality it provides) project is defined by the urls a
user can visit.

We've talked already about Django **urlconfs** and how we can bolt *all* the
urls that are provided by a single app into the root urlconf in our Django
project.

When you start work on the views for a new Django app, considering the
architecture of your URLs can be a good way to envision the functionality of
the app.

In general, an *app* that serves any sort of views should contain its own
urlconf. The *project* urlconf should *include* these where possible.

You can start this process by adding urls to your app. To begin, you need to
add a new ``urls.py`` file in your app package. You'll create the URLs for your
app here.

.. code-block:: python

    from django.conf.urls import patterns, url

    urlpatterns = patterns('myapp.views',
        url(r'^$',
            'stub_view',
            name="app_index"),
    )

A Word On Prefixes
------------------

The ``patterns`` function takes a first argument called the *prefix*.

When it is not the empty string, it is added to any view names in ``url()``
calls in the same call to ``patterns``.

In a root urlconf like the one in ``mysite``, this isn't too useful

But in ``myapp.urls`` it allous you to refer to views by simple function name.

This mean ther is no need to import every view function into ``urls.py``. This
can alleviate circular imports and reduce repetetive typing.


Stub Views
----------

One of the techniques I use when writing Django apps is to create a simple view
function that I can use in place of the *real* view functions I have yet to
write. I call it a *stub* and it allows me to think about the architecture of
my app as a whole before I commit any code to disk.

Here's an example that will print any args or kwargs the stub is called with:

.. code-block:: python

    from django.http import HttpResponse

    def stub_view(request, *args, **kwargs):
        body = "Stub View\n\n"
        if args:
            body += "Args:\n"
            body += "\n".join(["\t%s" % a for a in args])
        if kwargs:
            body += "Kwargs:\n"
            body += "\n".join(["\t%s: %s" % i for i in kwargs.items()])
        return HttpResponse(body, content_type="text/plain")

When you have such a view in place, you can use it as the view callable for any
urls you create.  You'll be able to load "pages" of your app in a browser and
view information about how they were called. This can help you to find the best
way to express the architecture you seek.

As an example, consider a simple blog app, like the microblog we created for
Flask. A simple url space for such an app might include:

* A page to view a list of blog posts
* A page to view a single blog post
* A page to allow adding a post

A simple urlconf for such a set of views might look like this:

.. code-block:: python

    from django.conf.urls import patterns, url

    urlpatterns = patterns('blogapp.views',
        url(r'^$',
            'stub_view',
            name="list_posts"),
        url(r'^posts/(\d+)/$', 
            'stub_view', 
            name="view_post"),
        url(r'^posts/add/$',
            'stub_view',
            name="add_post"),
    )

Capturing Path Data in URLs
---------------------------

Notice that for the view of a single post, you need to capture the id of the
post. Django uses regular expression syntax to accomplish this.

In the above urls, ``(\d+)`` captures one or more digits to serve as the
post_id.

You can also use a *named capture group* in a Django urlconf:

.. code-block:: python

    r'^posts/(?P<post_id>\d+)/$'

How you declare a capture group in your url pattern regexp influences how it
will be passed to the view callable. If you try to load the above "view_post"
view using the *unnamed* capture group you'll find that the id captured is
passed to the view as a positional argument. If you use the *named* capture
group, it becomes a keyword argument.

In order for app urls to be found, you need to include them in your project
urlconf:

.. code-block:: python

    urlpatterns = patterns('',
        url(r'^approot/', include('myapp.urls')),
    )

Once you've done so, you'll be able to view the urls for the new app relative
to the "/approot/" base.


Testing Views
=============

In writing tests for views, it's often nice to be able to have a few items to
test. This can be a good use for the TestCase ``setUp`` method. 


Considerr, for example, a blog where people can view posts that have been
published, but not those that have not. 

To test such a scenario, we might want to create a series of sample posts, some
with a publication date and some without:

.. code-block:: python

    class FrontEndTestCase(TestCase):
        """test views provided in the front-end"""
        fixtures = ['some_fixture.json', ]

        def setUp(self):
            self.now = datetime.datetime.utcnow().replace(tzinfo=utc)
            self.timedelta = datetime.timedelta(15)
            author = User.objects.get(pk=1)
            for count in range(1,11):
                post = Post(title="Post %d Title" % count,
                            text="foo",
                            author=author)
                if count < 6:
                    # publish the first five posts
                    pubdate = self.now - self.timedelta * count
                    post.published_date = pubdate
                post.save()

Then a test of the list view of the blog might look something like this:

.. code-block:: python

        Class FrontEndTestCase(TestCase): # already here
        # ...
        def test_list_only_published(self):
            resp = self.client.get('/')
            self.assertContains(resp, "Recent Posts")
            for count in range(1,11):
                title = "Post %d Title" % count
                if count < 6:
                    self.assertContains(resp, title, count=1)
                else:
                    self.assertNotContains(resp, title)

Note that we also test to ensure that the unpublished posts are *not* visible.


Writing Views
=============

Let's consider for a moment a view that might make such a test pass. It would
need to gather a list of posts that have been published and then pass them to a
template to be rendered.

.. code-block:: python

    from django.template import RequestContext, loader
    from myapp.models import Post
    
    def list_view(request):
        published = Post.objects.exclude(published_date__exact=None)
        posts = published.order_by('-published_date')
        template = loader.get_template('list.html')
        context = RequestContext(request, {
            'posts': posts,
        })
        body = template.render(context)
        return HttpResponse(body, content_type="text/html")

There are a few steps here to look over.

**Getting Data**

First, you get the list of published posts, ordered by the reverse of the
publication date:

.. code-block:: python

    published = Post.objects.exclude(published_date__exact=None)
    posts = published.order_by('-published_date')

We begin by using the QuerySet API to fetch all the posts that have
``published_date`` set

Remember from your understanding of the Django QuerySet API that at this point
no query has actually been issued to the database.

**Loading a Template**

Next, you get a template using the loader provided by ``django.template``:

.. code-block:: python

    template = loader.get_template('list.html')

Django uses configuration to determine how to find templates. By default,
Django looks in installed *apps* for a ``templates`` directory.

It also provides a setting, ``TEMPLATE_DIRS`` to list specific directories.
This allows you to set Django to look for templates in your *project* directory
or other shared filesystem locations.

**A word on Templates**

Before we move on, a quick word about Django templates. You've seen Jinja2
which was "inspired by Django's templating system". Basically, you already know
how to write Django templates.

The biggest difference is that Django templates **do not** allow any python
expressions. The Django philosophy is to use the view to prepare data in
whatever form is needed.

**Context and Rendering Templates**

Once the data is arranged you pass it in to the template for rendering by
building a context.

.. code-block:: python

    context = RequestContext(request, {
        'posts': posts,
    })
    body = template.render(context)

Django's RequestContext provides common bits, similar to the global context in
Flask. The constructor for this class takes the incoming request as the first
argument.

You can add data of your own to that context so they can be used by the
template. Simply pass a dictionary of additional keys and values as the second
argument.

**Returning a Response**

Finally, we build an HttpResponse and return it.

.. code-block:: python

    return HttpResponse(body, content_type="text/html")

This is, fundamentally, no different from the ``stub_view`` above.

**Hooking up Views**

Once the view function for a given URL endpoint is complete, you can connect it
by providing it's name in the *urlconf*:

.. code-block:: python

    url(r'^$',
        'list_view',
        name="blog_index"),


Common Patterns
---------------

This workflow is a common pattern in Django views:

* get a template from the loader
* build a context, usually using a RequestContext
* render the template
* return an HttpResponse

So common in fact that Django provides two shortcuts for us to use:

* ``render(request, template[, ctx][, ctx_instance])``
* ``render_to_response(template[, ctx][, ctx_instance])``

The difference between the two is that the former will use a RequestContext
automatically. The latter will not. The RequestContext is important when you
want access to things like the current user in your templates.

Knowing these shortcuts allows you to replace most of your view with the
``render`` shortcut.

.. code-block:: python

    from django.shortcuts import render # <- already there
    
    # rewrite our view
    def list_view(request):
        published = Post.objects.exclude(published_date__exact=None)
        posts = published.order_by('-published_date')
        context = {'posts': posts}
        return render(request, 'list.html', context)

Remember though, all you did manually before is still happening. It's just
hidden behind the curtain.

Dealing with Missing Data
-------------------------

When providing views of individual items, it's a common pattern to want to
return a 404 response when the item asked for is not present.

One of the features of the Django ORM is that all models raise a
``DoesNotExist`` exception if calling ``get`` returns nothing.

This exception is actually an attribute of the Model you look for. There's also
an ``ObjectDoesNotExist`` for when you don't know which model you have.

You can use that fact to raise a Not Found exception:

.. code-block:: python

    from django.http import Http404

    def detail_view(request, post_id):
        published = Post.objects.exclude(published_date__exact=None)
        try:
            post = published.get(pk=post_id)
        except Post.DoesNotExist:
            raise Http404
        context = {'post': post}
        return render(request, 'detail.html', context)

Django will handle the rest for you.

You can test this type of scenario as well. Remember the front-end tests we
considered above? Here's a new one to add:

.. code-block:: python

        def test_details_only_published(self):
            for count in range(1,11):
                title = "Post %d Title" % count
                post = Post.objects.get(title=title)
                resp = self.client.get('/posts/%d/' % post.pk)
                if count < 6:
                    self.assertEqual(resp.status_code, 200)
                    self.assertContains(resp, title)
                else:
                    self.assertEqual(resp.status_code, 404)

Class Based Views
=================

Since Django 1.3 the idea of *class-based views* have grown more popular. I
won't cover them explicitly in this quick lecture, but you should
`read more about them`_.

.. _read more about them: https://docs.djangoproject.com/en/dev/topics/class-based-views/

Personally, I still strongly prefer functional views when I can get away with
them. The extra functionality supported by Django's implementation of
class-based views is (IMO) outweighed in most cases by the complexity of
tracking where functionality comes from.
