*******************************************
Scraping Apartment Listings from Craigslist
*******************************************

Work through this exercise to create a Python script to extract a list of
apartment rentals from Craigslist.

While working, you should use the virtualenv project we created in class for
learning about the BeautifulSoup package.

.. code-block:: bash

    heffalump:~ cewing$ workon souptests
    [souptests]
    heffalump:souptests cewing$

Begin by creating a new Python file, call it ``scraper.py``. Open it in your
editor.

Step 1: Fetch Search Results
============================

The first step is to use the ``requests`` library to fetch a set of search
results from the Craigslist site.

In order to do so, we will need to assemble a *query* that fits with the search
form present on the `Seattle apartment listing page`_.

.. _Seattle apartment listing page: https://seattle.craigslist.org/apa/

Use your browser's devtools to determine the name of the various inputs
available in the search form on that page.  You should end up with a list that
includes at least the following:

* keywords: ``query=keyword+values+here``
* price: ``minAsk=NNN maxAsk=NNN``
* bedrooms: ``bedrooms=N`` (N in range 1-8)

You'll also discover, if you submit a search, that the URL to be used for a
search request is in fact this:

    http://seattle.craigslist.org/search/apa

Our goal is to write a Python function that will return the search results HTML
from a query to craigslist. This function should:

* It will accept one keyword argument for each of the possible query values
* It will build a dictionary of request query parameters from incoming keywords
* It will make a request to the craigslist server using this query
* It will return the body of the response if there is no error
* It will raise an error if there is a problem with the response

Here is one possible solution for this query:

.. code-block:: python

    import requests

    def fetch_search_results(
        query=None, minAsk=None, maxAsk=None, bedrooms=None
    ):
        search_params = {
            key: val for key, val in locals().items() if val is not None
        }
        if not search_params:
            raise ValueError("No valid keywords")

        base = 'http://seattle.craigslist.org/search/apa'
        resp = requests.get(base, params=search_params, timeout=3)
        resp.raise_for_status()  # <- no-op if status==200
        return resp.content, resp.encoding


Write the results of your search to a file, ``apartments.html`` so that you can
work on it without needing to hammer the craigslist servers.

Write a ``read_search_results`` function which reads this file from disk and
returns the content and encoding in the same way as the above function. Then
you can switch between the two without altering the API. I leave this exercise
to you.

Step 2: Parse Search Results
============================

Next, we need a function ``parse_source`` to set up the HTML as DOM nodes for
scraping. It will need to:

* Take the response body from the previous method (or some other source)
* Parse it using BeautifulSoup
* Return the parsed object for further processing

This function can be quite simple. Add it to ``scraper.py``:

.. code-block:: python

    # add this import at the top
    from bs4 import BeautifulSoup

    # then add this function lower down
    def parse_source(html, encoding='utf-8'):
        parsed = BeautifulSoup(html, from_encoding=encoding)
        return parsed

In order to see the results we have at this point, we'll need to make our
``scraper.py`` executable by adding a ``__main__`` block.  For:

.. code-block:: python

    # add another import at the top
    import sys

    if __name__ == '__main__':
        if len(sys.argv) > 1 and sys.argv[1] == 'test':
            html, encoding = read_search_results()
        else:
            html, encoding = fetch_search_results(
                minAsk=500, maxAsk=1000, bedrooms=2
            )
        doc = parse_source(html, encoding)
        print doc.prettify(encoding=encoding)

Now, you can execute your scraper script in one of two ways:

1. ``python scraper.py`` will fetch results directly from Craigslist.
2. ``python scraper.py test`` will use your stored results from file.


Step 3: Extract Listing Information
===================================

You are going to build a function that extracts useful information from each of
the listings in the parsed HTML search results. From each listing, we should
extract the following information:

* Location data (latitude and longitude)
* Source link (to craiglist detailed listing)
* Description text
* Price and size data

You'll be building this function one step at a time, to simplify the task.

3a: Find Individual Listings
----------------------------

The first job is to find the container that holds each individual listing. Use
your browser's devtools to identify the container that holds each individual
listing. Then, write a function that takes in the parsed HTML and returns a
list of the apartment listing container nodes.  Call this function
``extract_listings``:

.. code-block:: python

    def extract_listings(parsed):
        listings = parsed.find_all('p', class_='row')
        return listings

If you update your ``__main__`` block to use this new function, you can verify
the results visually:


.. code-block:: python

    if __name__ == '__main__':
        if len(sys.argv) > 1 and sys.argv[1] == 'test':
            html, encoding = read_search_results()
        else:
            html, encoding = fetch_search_results(
                minAsk=500, maxAsk=1000, bedrooms=2
            )
        doc = parse_source(html, encoding)
        listings = extract_listings(doc) # add this line
        print len(listings)              # and this one
        print listings[0].prettify()     # and this one too

Call your script from the command line (in test mode), to see your results:

.. code-block:: bash

    [souptests]
    heffalump:souptests cewing$ python scraper.py test
    100
    <p class="row" data-latitude="47.4924143400595" data-longitude="-122.235904626445" data-pid="4345117401">
      ...
    </p>

    [souptests]
    heffalump:souptests cewing$


If you look through your test listings file using your browser's devtools,
you'll notice that only *some* of them actually have latitude and longitude,
Because the specs for our scraper require this data, we want to filter out any
listings that do not.

``BeautifulSoup`` allows us to filter our searches using HTML attributes with
the ``attrs`` argument. One way of doing this is to provide a specific value
for a given attribute:

.. code-block:: python

    doc.filter('p', attrs={'data-longitude': "47.4924143400595"})

It should be pretty clear though, that each of our listings is located in a
different place, and this type of filtering won't really help much. Happily,
you can also provide ``True`` as the value for a given attribute. By doing so,
you are telling ``BeautifulSoup`` that you want to match any node that **has
that attribute**, regardless of the specific value.

Let's use this to enhance our ``extract_listings`` function so that it only
returns the listings that have location attributes:

.. code-block:: python

    def extract_listings(parsed):
        location_attrs = {'data-latitude': True, 'data-longitude': True}
        listings = parsed.find_all('p', class_='row', attrs=location_attrs)
        return listings

Calling the script from the command line now should return a slightly different
number of results:

.. code-block:: bash

    [souptests]
    heffalump:souptests cewing$ python scraper.py test
    74
    <p class="row" data-latitude="47.4924143400595" data-longitude="-122.235904626445" data-pid="4345117401">
      ...
    </p>

    [souptests]
    heffalump:souptests cewing$

3b: Extract Location Data
-------------------------

You've used the location data to filter results. Now that only those results
that have locations are being listed, let's begin scraping that data out of the
HTML page.

In ``BeautifulSoup``, the HTML attributes of a given tag are found as the
``attrs`` attribute of the ``Tag`` object. This attribute is a dictionary and
it is certain to be present, even if it is empty. The names of the attributes
are the keys of this dictionary, and the HTML values are the values.

We've already said that there is a certain set of data we want to preserve
about each listing. We could create some custom Python class to represent a
listing, and perhaps in some situations that would be appropriate, but for this
simple script we will just build a dictionary that represents each listing.

Let's update our ``extract_listings`` function to build a dictionary for each
listing, and begin by populating it with the location data we extract:

.. code-block:: python

    def extract_listings(parsed):
    location_attrs = {'data-latitude': True, 'data-longitude': True}
    listings = parsed.find_all('p', class_='row', attrs=location_attrs)
    extracted = []
    for listing in listings:
        location = {key: listing.attrs.get(key, '') for key in location_attrs}
        this_listing = {
            'location': location,
        }
        extracted.append(this_listing)
    return extracted

Since the return value of this function has now changed from a list of ``Tag``
objects to a list of dictionaries, we will also need to update our ``__main__``
block:

.. code-block:: python

    if __name__ == '__main__':
        import pprint                                  # add this import
        if len(sys.argv) > 1 and sys.argv[1] == 'test':
            html, encoding = read_search_results()
        else:
            html, encoding = fetch_search_results(
                minAsk=500, maxAsk=1000, bedrooms=2
            )
        doc = parse_source(html, encoding)
        listings = extract_listings(doc)
        print len(listings)
        pprint.pprint(listings[0])                     # update this line

And now, executing this script at the command line should return the following:

.. code-block:: bash

    [souptests]
    heffalump:souptests cewing$ python scraper.py test
    74
    {'location': {'data-latitude': u'47.4924143400595',
                  'data-longitude': u'-122.235904626445'}}
    [souptests]
    heffalump:souptests cewing$

3c: Extract Description and Link
--------------------------------

We used the ``find_all`` method of a ``Tag`` above to extract *all* the
listings from our parsed document. We can use the ``find`` method of each
listing to find *the first* item that matches our search filter.

Use your browser's devtools to find the element in each listing that contains
the descriptive text about the listing. What kind of HTML tag is it? What other
useful bit of information is present in that tag?

Use this information to enhance our ``extract_listings`` function so that each
dictionary it produces also contains a ``description`` and ``link``:

.. code-block:: python

    def extract_listings(parsed):
        location_attrs = {'data-latitude': True, 'data-longitude': True}
        listings = parsed.find_all('p', class_='row', attrs=location_attrs)
        extracted = []
        for listing in listings:
            location = {key: listing.attrs.get(key, '') for key in location_attrs}
            link = listing.find('span', class_='pl').find('a') # add this
            this_listing = {
                'location': location,
                'link': link.attrs['href'],                    # add this too
                'description': link.string.strip(),            # and this
            }
            extracted.append(this_listing)
        return extracted

Note that we are calling the string ``strip`` method on the value we get for
the description from the ``string`` attribute of the ``Tag``.

The most obvious reason is that we don't want extra whitespace.

The second reason is more subtle, but also more important. The values returned
by ``string`` are **not** simple unicode strings.

They are actually instances of the ``NavigableString`` class:

.. code-block:: python

    >>> listing.find('span', class_='pl').find('a').string.__class__
    <class 'bs4.element.NavigableString'>

These class instances contain not only the text, but also instance attributes
that connect them to the DOM nodes that surround them. These attributes take
quite a bit of memeory. Calling ``strip`` or casting them to ``unicode`` with
the ``unicode`` type object converts them, saving memory.

Executing the script from the command line now should show you that you have
succeeded:

.. code-block:: bash

    [souptests]
    heffalump:souptests cewing$ python scraper.py test
    74
    {'description': u'2 BEDROOM 2 BATHROOM Zero Down   Rent with Option to Buy',
     'link': u'/oly/apa/4345117401.html',
     'location': {'data-latitude': u'47.4924143400595',
                  'data-longitude': u'-122.235904626445'}}
    [souptests]
    heffalump:souptests cewing$


3d: Extract Price and Size
--------------------------

Again, use your browser devtools to find the container that holds *both* the
price of a listed apartment, and the text that describes its size.

What's different about these two? 

The price data is contained inside a convenient container of its own. The size,
however, is not. It is just some text found in the main container **after** the
``Tag`` that holds the price.  You can see this by dropping a breakpoint into
your ``extract_listings`` function and inspecting the DOM:

.. code-block:: pycon

    > /Users/cewing/projects/souptests/scraper.py(39)extract_listings()
    -> this_listing = {
    (Pdb) l2 = listing.find('span', class_='l2')
    (Pdb) print l2.prettify()
    <span class="l2">
     <span class="price">
      $960
     </span>
     / 3br
     <span class="pnr">
      <small>
       (Seattle98102)
      </small>
      <span class="px">
       <span class="p">
        pic
        <a class="maptag" data-pid="4345117401" href="#">
         map
        </a>
       </span>
      </span>
     </span>
    </span>

    (Pdb)

If you try to get at that text by using the ``string`` attribute of the ``l2``
span tag, you'll see that it just isn't there:

.. code-block:: pycon

    (Pdb) print l2.string
    None
    (Pdb)

Likewise, if you use the ``text`` attribute to get *all* the text in the tag,
you end up with more than you really want:

.. code-block:: pycon

    (Pdb) print l2.text
      $960 / 3br -    (Seattle98102)   pic map
    (Pdb)

You *could* parse this string to extract what you want, but why? There's an
easier way.

All text in a DOM document is really contained in instances of the
``NavigableString`` class.

We've already talked about how this class contains references to the DOM nodes
around it. These references allow us to *navigate* the DOM, moving from one
node to the next directly instead of simply searching for what we want.
``BeautifulSoup`` supports navigating from node to node in a number of ways:

* **into** (or down to the next DOM tree level):

  * ``Tag.children`` (iterator with immediately contained elements)
  * ``Tag.descendants`` (generator returning **all** contained elements)

* **out** (or up to the next DOM tree level):

  * ``Tag.parent``: (the tag containing this tag)

  * ``Tag.parents``: (generator returning all containers above this tag,
    closest first)

* **across** (or within the same DOM tree level):

  * ``Tag.next_sibling`` (the node immediately following this one)
  * ``Tag.next_siblings`` (generator returning **all nodes** at this level
    after this one)
  * ``Tag.previous_sibling`` (the node immediately before this one)
  * ``Tag.previous_siblings`` (generator returning **all nodes** at this level
    before this one)

In this case, that ability can help us a great deal. Looking carefully, you
might notice that the text describing the size of an apartment is located just
after the ``span`` that contains our price. This means we can use navigation
methods starting from the span containing the price to get where we want to be:

.. code-block:: pycon

    (Pdb) price_node = listing.find('span', class_='l2').find('span', class_='price')
    (Pdb) price_node
    <span class="price">$960</span>
    (Pdb) price_node.next_sibling
    u' / 3br -  '
    (Pdb) price_node.next_sibling.strip()
    u'/ 3br -'
    (Pdb) price_node.next_sibling.strip(' \n-/')
    u'3br'
    (Pdb)

Type 'quit' at your pdb prompt to exit the debugger and then remove the
breakpoint from your code.

Now update ``extract_listings`` to include the information we've just found:

.. code-block:: python

    def extract_listings(parsed):
        location_attrs = {'data-latitude': True, 'data-longitude': True}
        listings = parsed.find_all('p', class_='row', attrs=location_attrs)
        extracted = []
        for listing in listings:
            location = {key: listing.attrs.get(key, '') for key in location_attrs}
            link = listing.find('span', class_='pl').find('a')
            price_span = listing.find('span', class_='price')   # add me
            this_listing = {
                'location': location,
                'link': link.attrs['href'],
                'description': link.string.strip(),
                'price': price_span.string.strip(),             # and me
                'size': price_span.next_sibling.strip(' \n-/')  # me too
            }
            extracted.append(this_listing)
        return extracted


And now executing your script from the command line should show these new
elements for a listing:

.. code-block:: bash

    [souptests]
    heffalump:souptests cewing$ python scraper.py test
    74
    {'description': u'2 BEDROOM 2 BATHROOM Zero Down   Rent with Option to Buy',
     'link': u'/oly/apa/4345117401.html',
     'location': {'data-latitude': u'47.4924143400595',
                  'data-longitude': u'-122.235904626445'},
     'price': u'$960',
     'size': u'3br'}
    [souptests]
    heffalump:souptests cewing$


