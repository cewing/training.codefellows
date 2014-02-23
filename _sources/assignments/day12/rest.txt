*****************************************
Consuming Data from a RESTful Web Service
*****************************************

As an example of a RESTful web service, let's add some more information to our
list of apartment rentals from previous exercises.

We'll use a common, public API provided by Google.

.. class:: incremental center

**Geocoding**

Geocoding with Google APIs
==========================

https://developers.google.com/maps/documentation/geocoding

Open a python interpreter using your ``souptests`` virtualenv:

.. code-block:: bash

    [souptests]
    heffalump:souptests cewing$ python

Then, import the ``requests`` library and prepare to make an HTTP request to
the google geocoding service resource:

.. code-block:: pycon

    >>> import requests
    >>> import json
    >>> from pprint import pprint
    >>> url = 'http://maps.googleapis.com/maps/api/geocode/json'
    >>> addr = '511 Boren Ave. N, Seattle, 98109'
    >>> parameters = {'address': addr, 'sensor': 'false' }
    >>> resp = requests.get(url, params=parameters)
    >>> data = json.loads(resp.text)
    >>> if data['status'] == 'OK':
    ...     pprint(data)
    ...
    {u'results': [{u'address_components': [{u'long_name': u'511',
                                            u'short_name': u'511',
                                            u'types': [u'street_number']},
                                           {u'long_name': u'Boren Avenue North',
                                            u'short_name': u'Boren Ave N',
                                            u'types': [u'route']},
                                           {u'long_name': u'South Lake Union',
                                            u'short_name': u'SLU',
                                            u'types': [u'neighborhood',
                                                       u'political']},
                                           {u'long_name': u'Seattle',
                                            u'short_name': u'Seattle',
                                            u'types': [u'locality',
                                                       u'political']},
                                           {u'long_name': u'King County',
                                            u'short_name': u'King County',
                                            u'types': [u'administrative_area_level_2',
                                                       u'political']},
                                           {u'long_name': u'Washington',
                                            u'short_name': u'WA',
                                            u'types': [u'administrative_area_level_1',
                                                       u'political']},
                                           {u'long_name': u'United States',
                                            u'short_name': u'US',
                                            u'types': [u'country',
                                                       u'political']},
                                           {u'long_name': u'98109',
                                            u'short_name': u'98109',
                                            u'types': [u'postal_code']}],
                   u'formatted_address': u'511 Boren Avenue North, Seattle, WA 98109, USA',
                   u'geometry': {u'location': {u'lat': 47.6235481,
                                               u'lng': -122.336212},
                                 u'location_type': u'ROOFTOP',
                                 u'viewport': {u'northeast': {u'lat': 47.6248970802915,
                                                              u'lng': -122.3348630197085},
                                               u'southwest': {u'lat': 47.6221991197085,
                                                              u'lng': -122.3375609802915}}},
                   u'types': [u'street_address']}],
     u'status': u'OK'}
    >>>

You can also do the reverse, provide a location as latitude and longitude and
receive address informatin back:

.. code-block:: pycon

    >>> location = data['results'][0]['geometry']['location']
    >>> latlng="{lat},{lng}".format(**location)
    >>> parameters = {'latlng': latlng, 'sensor': 'false'}
    >>> resp = requests.get(url, params=paramters)
    >>> data = json.loads(resp.text)
    >>> if data['status'] == 'OK':
    ...     pprint(data)
    ...
    {u'results': [{u'address_components': [{u'long_name': u'511',
                                            u'short_name': u'511',
                                            u'types': [u'street_number']},
                                           {u'long_name': u'Boren Avenue North',
                                            u'short_name': u'Boren Ave N',
                                            u'types': [u'route']},
                                           {u'long_name': u'South Lake Union',
                                            u'short_name': u'SLU',
                                            u'types': [u'neighborhood',
                                                       u'political']},
                                           {u'long_name': u'Seattle',
                                            u'short_name': u'Seattle',
                                            u'types': [u'locality',
                                                       u'political']},
                                           {u'long_name': u'King County',
                                            u'short_name': u'King County',
                                            u'types': [u'administrative_area_level_2',
                                                       u'political']},
                                           {u'long_name': u'Washington',
                                            u'short_name': u'WA',
                                            u'types': [u'administrative_area_level_1',
                                                       u'political']},
                                           {u'long_name': u'United States',
                                            u'short_name': u'US',
                                            u'types': [u'country',
                                                       u'political']},
                                           {u'long_name': u'98109',
                                            u'short_name': u'98109',
                                            u'types': [u'postal_code']}],
                   u'formatted_address': u'511 Boren Avenue North, Seattle, WA 98109, USA',
                   u'geometry': {u'location': {u'lat': 47.6235481,
                                               u'lng': -122.336212},
                                 u'location_type': u'ROOFTOP',
                                 u'viewport': {u'northeast': {u'lat': 47.6248970802915,
                                                              u'lng': -122.3348630197085},
                                               u'southwest': {u'lat': 47.6221991197085,
                                                              u'lng': -122.3375609802915}}},
                   u'types': [u'street_address']},
                  ...
                  ],
     u'status': u'OK'}
    >>>

Notice that in the response there are actually a number of results.  These are
decreasingly specific designations for the location you provided.  The
``types`` values for each indicate the level of geographical specificity for
each result.


Mashup!
=======

Let's create a simple mashup of this data with the apartment listings we built
by scraping Craigslist in an earlier exercise.

Open your ``scraper.py`` file in your editor and add a new function. Call it
``add_address``. This function should:

* take a single listing from our craiglist work
* format the location data provided in that listing properly
* make a reverse geocoding lookup using the google api above
* add the best available address to the listing
* return the updated listing

Can you write this function without looking at the solution below?  Try it.

Solution
--------

Here are the changes I made to ``scraper.py`` to add this function:

.. code-block:: python

    # add an import
    import json

    # and a function
    def add_address(listing):
        api_url = 'http://maps.googleapis.com/maps/api/geocode/json'
        loc = listing['location']
        latlng_tmpl = "{data-latitude},{data-longitude}"
        parameters = {
            'sensor': 'false',
            'latlng': latlng_tmpl.format(**loc),
        }
        resp = requests.get(api_url, params=parameters)
        resp.raise_for_status() # <- this is a no-op if all is well
        data = json.loads(resp.text)
        if data['status'] == 'OK':
            best = data['results'][0]
            listing['address'] = best['formatted_address']
        else:
            listing['address'] = 'unavailable'
        return listing


You'll need to bolt the new function into your script so that the results it
gives are added to each listing.

Make the following changes to your ``__main__`` block:

.. code-block:: python

    if __name__ == '__main__':
        import pprint
        if len(sys.argv) > 1 and sys.argv[1] == 'test':
            html, encoding = read_search_results()
        else:
            html, encoding = fetch_search_results(
                minAsk=500, maxAsk=1000, bedrooms=2
            )
        doc = parse_source(html, encoding)    # above here is the same
        for listing in extract_listings(doc): # change everything below
            listing = add_address(listing)
            pprint(listing)

Give it a whirl, using the test approach so you don't hit Craigslist while
trying it out:

.. code-block:: bash

    [souptests]
    heffalump:souptests cewing$ python scraper.py test
    {'address': u'12339-12399 78th Avenue South, Seattle, WA 98178, USA',
     'description': u'2 BEDROOM 2 BATHROOM Zero Down   Rent with Option to Buy',
     'link': u'/oly/apa/4345117401.html',
     'location': {'data-latitude': u'47.4924143400595',
                  'data-longitude': u'-122.235904626445'},
     'price': u'$960',
     'size': u'3br'}
    {'address': ...
    ...
    [souptests]
    heffalump:souptests cewing$

Nifty, eh?

Reduce Your Footprint
=====================

At the moment, all of our results need to be held in memory at the same time.
In this case it probably isn't too big a deal, but it's good to practice being
kind to your resources.

Update the 'extract_listings' method to turn it into a generator.  Then we can
process a single apartment listing at a time, decreasing the memory
requirements of our script:

.. code-block:: python

    def extract_listings(parsed):
        location_attrs = {'data-latitude': True, 'data-longitude': True}
        listings = parsed.find_all('p', class_='row', attrs=location_attrs)
        # delete the line where you create a list in which to store
        # your listings
        for listing in listings:
            location = {key: listing.attrs.get(key, '') for key in location_attrs}
            link = listing.find('span', class_='pl').find('a')
            price_span = listing.find('span', class_='price')
            this_listing = {
                'location': location,
                'link': link.attrs['href'],
                'description': link.string.strip(),
                'price': price_span.string.strip(),
                'size': price_span.next_sibling.strip(' \n-/')
            }
            # delete the line where you append this result to a list
            yield this_listing  # This is the only change you need to make

When you make that change, each individual listing will be ``yielded`` from the
``extract_listings`` generator, and you will be able to add an address to each
without building all the rest first.

Going Further
=============

This would be a great opportunity for using asynchronous processing as well.

Can you think of a way to handle the adding of an address to each listing using
an asynchronous call using ``gevent`` or ``tornado``?

Consider this a standing challenge.



