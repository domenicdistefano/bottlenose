Bottlenose
==========

Description
-----------

Bottlenose is a thin Python wrapper over the Amazon Product Advertising API.
There is practically no overhead, and no magic (unless you add it yourself).

Before you get started, make sure you have both Amazon Product Advertising and
AWS accounts (yes, they are separate -- confusing, I know).

Features
--------

* Compatible with Python versions 2.4 and up
* Support for CA, CN, DE, ES, FR, IT, JP, UK, and US Amazon endpoints
* No requirements, except simplejson for Python pre-2.6
* Configurable query parsing
* Configurable throttling for batches of queries
* Configurable query caching
* Configurable error handling and retry

Usage
-----

#### 1. Available Search Methods:

```python
# Required
amazon = bottlenose.Amazon(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_ASSOCIATE_TAG)

# Search for a Specific Item
response = amazon.ItemLookup(ItemId="B007OZNUCE")

# Search for Items by Keywords
response = amazon.ItemSearch(Keywords="Kindle 3G", SearchIndex="All")

# Search for Images for an item
response = amazon.ItemLookup(ItemId="1449372422", ResponseGroup="Images")

# Search for Similar Items
response = amazon.SimilarityLookup(ItemId="B007OZNUCE")
```

#### 2. Available Shopping Related Methods:

```python
# Required
amazon = bottlenose.Amazon(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_ASSOCIATE_TAG)

# Create a cart
response = amazon.CartCreate(...)

# Adding to a cart
response = amazon.CartAdd(CartId, ...)

# Get a cart by ID
response = amazon.CartGet(CartId, ...)

# Modifying a cart
response = amazon.CartModify(ASIN,CartId,...)

# Clearing a cart
response = amazon.CartClear(CartId, ...)
```

#### 3. Sample Code

    >>> import bottlenose
    >>> amazon = bottlenose.Amazon(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_ASSOCIATE_TAG)
    >>> response = amazon.ItemLookup(ItemId="0596520999", ResponseGroup="Images",
        SearchIndex="Books", IdType="ISBN")
    <?xml version="1.0" ?><ItemLookupResponse xmlns="http://webservices.amazon...

Here's another example.

    >>> response = amazon.ItemSearch(Keywords="Kindle 3G", SearchIndex="All")
    <?xml version="1.0" ?><ItemSearchResponse xmlns="http://webservices.amazon...

Bottlenose can also read your credentials from the environment automatically;
just set `$AWS_ACCESS_KEY_ID`, `$AWS_SECRET_ACCESS_KEY` and
`$AWS_ASSOCIATE_TAG`.

Any valid API call from the following is supported (in addition to any others
that may be added in the future). Just plug in appropriate request parameters
for the operation you'd like to call, and you're good to go.

    BrowseNodeLookup
    CartAdd
    CartClear
    CartCreate
    CartGet
    CartModify
    ItemLookup
    ItemSearch
    SellerListingLookup
    SellerListingSearch
    SellerLookup
    SimilarityLookup

You can refer here for a full listing of API calls to be made from Amazon.
- [Amazon API Quick Reference Card](http://s3.amazonaws.com/awsdocs/Associates/2011-08-01/prod-adv-api-qrc-2011-08-01.pdf)

-------

For more information about these calls, please consult the [Product Advertising
API Developer Guide](http://docs.amazonwebservices.com/AWSECommerceService/latest/DG/index.html).

Parsing
-------

By default, API calls return the response as a raw bytestring. You can change
this with the `Parser` constructor argument. The parser is a callable that
takes a single argument, the response as a raw bytestring, and returns the
parsed response in a format of your choice.

For example, to parse responses with BeautifulSoup:

```python
from bs4 import BeautifulSoup

amazon = bottlenose.Amazon(Parser=BeautifulSoup)
```

Throttling/Batch Mode
---------------------

Amazon strictly limits the query rate on its API (by default, one query
per second per associate tag). If you have a batch of non-urgent queries, you
can use the `MaxQPS` argument to limit them to no more than a certain rate;
any faster, and bottlenose will `sleep()` until it is time to make the next
API call.

Generally, you want to be just under the query limit, for example:

```python
amazon = bottlenose.Amazon(MaxQPS=0.9)
```

If some other code is also querying the API with your associate tag (for
example, a website backend), you'll want to choose an even lower value
for MaxQPS.

Caching
-------

You can often get a major speedup by caching API queries. Use the `CacheWriter`
and `CacheReader` constructor arguments.

`CacheWriter` is a callable that takes two arguments, a cache url, and the
raw response (a bytestring). It will only be called after successful queries.

`CacheReader` is a callable that takes a single argument, a cache url, and
returns a (cached) raw response, or `None` if there is nothing cached.

The cache url is the actual query URL with authentication information removed.
For example:

    http://ecs.amazonaws.com/onca/xml?Keywords=vacuums&Operation=ItemSearch&Region=US&ResponseGroup=SearchBins&SearchIndex=All&Service=AWSECommerceService&Version=2011-08-01

Example code:

```python
def write_query_to_db(cache_url, data):
    ...

def read_query_from_db(cache_url):
    ...

amazon = bottlenose.Amazon(CacheWriter=write_query_to_db,
                           CacheReader=read_query_from_db)
```

Note that Amazon's [Product Advertising API Agreement](https://affiliate-program.amazon.com/gp/advertising/api/detail/agreement.html)
only allows you to cache queries for up to 24 hours.

Error Handling
--------------

Sometimes the Amazon API returns errors; for example, if you have gone over
your query limit, you'll get a 503. The `ErrorHandler` constructor argument
gives you a way to keep track of such errors, and to retry queries when you
receive a transient error.

`ErrorHandler` should be a callable that takes a single argument, a dictionary
with these keys:

 * api_url: the actual URL used to call the API
 * cache_url: `api_url` minus authentication information
 * exception: the exception raised (usually an `HTTPError` or `URLError`)

If your `ErrorHandler` returns true, the query will be retried. Here's some
example code that does exponential backoff after throttling:

```python
import random
import time
from urllib2 import HTTPError

def error_handler(err):
    ex = err['exception']
    if isinstance(ex, HTTPError) and ex.code == 503:
        time.sleep(random.expovariate(0.1))
        return True

amazon = bottlenose.Amazon(ErrorHandler=error_handler)
```

License
-------

Apache License, Version 2.0. See LICENSE for details.
