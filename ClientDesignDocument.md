# Client Design #

The new Python client library will be used solely for discovery based APIs and will have no backward compatibility with non-discovery based APIs. Learning from the past several years of development with the V1 and V2 versions of the client libraries, this version of the library with eschew hand-coded per-API classes.

## Requirements ##

  * A modular design that allows the following components to be pluggable:
    * Model
    * HTTP Client
    * Storage Model for Auth Keys
    * HTTP Cache
  * Native support for OAuth 2.0

In addition the following things would be nice to have, but not required:
  * Python 3.x support by 2to3.

## Overview ##

At the highest level the interaction begins with building a service model to interact with the API. The service model is built from the discovery document.

```
def build(
  service,
  version, 
  http = httplib2.Http(), 
  discoveryServiceUrl = DISCOVERY_URI, 
  model = JsonModel()
)
```

**service**

> Name of the API, such as plus, moderator, or latitude.

**version**

> Version of the API to use.

**http**

> An httplib2 compatible instance that also is expected to handle authentication.

**discoveryServiceUrl**

> URI Template for the discovery service.

**model**

> The data model converts from the wire format into data objects.  Also responsible for instantiating prototype models.

## Example ##

Here is an example interaction using the client library. The example below uses a simple JsonModel with serializes the JSON wire format using simplejson. Note that the below example does not show using the Command Pattern for requests.

Also see the `sample.py` file in the v3 directory for example usage.

```
>>> import apiclient
>>> import oauth_wrap
>>> http = oauth_wrap.get_wrapped_http() # Handles Auth
>>> service = discovery.build("plus", "v1", http)
>>> help(service)

Help on Service in module discovery object:

class Service(__builtin__.object)
|  Top level interface for a service
|  
|  Methods defined here:
|  
|  __init__(self, http=<httplib2.Http object at 0xb714030c>)
|  
|  activities = method(self, **kwargs)
|      A description of how to use this function
|  
|  comments = method(self, **kwargs)
|      A description of how to use this function
|  
|  feeds = method(self, **kwargs)
|      A description of how to use this function
|  
|  groups = method(self, **kwargs)
|      A description of how to use this function
|  
|  people = method(self, **kwargs)
|      A description of how to use this function
|  
|  photos = method(self, **kwargs)
|      A description of how to use this function
|  

>>> activities = service.activities()
>>> help(activities)

 class Resource(__builtin__.object)
  |  A class for interacting with a resource.
  |  
  |  Methods defined here:
  |  
  |  __init__(self)
  |  
  |  delete = method(self, **kwargs)
  |      A description of how to use this function
  |      
  |      scope - A parameter
  |      alt - A parameter
  |      postId - A parameter
  |      userId - A parameter
  |  
  |  get = method(self, **kwargs)
  |      A description of how to use this function
  |      
  |      alt - A parameter
  |      postId - A parameter
  |      userId - A parameter
  |  
  |  insert = method(self, **kwargs)
  |      A description of how to use this function
  |      
  |      body - A parameter
  |      userId - A parameter
  |      alt - A parameter
  |      preview - A parameter
  |      resource - A parameter
  |  
  |  list = method(self, **kwargs)
  |      A description of how to use this function
  |      
  |      c - A parameter
  |      userId - A parameter
  |      max_results - A parameter
  |      scope - A parameter
  |      alt - A parameter
  |      max_comments - A parameter
  |      max_liked - A parameter
  |  
  |  search = method(self, **kwargs)
  |      A description of how to use this function
  |      
  |      c - A parameter
  |      pid - A parameter
  |      lon - A parameter
  |      q - A parameter
  |      max_results - A parameter
  |      radius - A parameter
  |      bbox - A parameter
  |      lat - A parameter
  |      alt - A parameter
  |  
  |  update = method(self, **kwargs)
  |      A description of how to use this function
  |      
  |      body - A parameter
  |      userId - A parameter
  |      abuseType - A parameter
  |      scope - A parameter
  |      alt - A parameter
  |      postId - A parameter
  |  
  |  

>>> help(activities.list)

| Help on method method in module __main__:
| 
| method(self, **kwargs) method of __main__.Resource instance
|     A description of how to use this function
|     
|     c - A parameter
|     userId - A parameter
|     max_results - A parameter
|     scope - A parameter
|     alt - A parameter
|     max_comments - A parameter
|     max_liked - A parameter
 

>>> activitylist = activities.list(scope='@self', userId='@me')
>>> activitylist

{u'kind': u'plus#activityFeed', u'links': {u'self': [{u'href': u'https://www.googleapis.com/plus/v1/activities/118170877744527614111/@self?alt=json', u'type': u'application/json'}], u'hub': [{u'href': u'http://pubsubhubbub.appspot.com/'}]}, u'title': u'Google plus Self Feed for Joe Gregorio', u'items': [{u'kind': u'plus#activity', u'links': {u'liked': [{u'count': 0, u'href': u'https://www.googleapis.com/plus/v1/activities/118170877744527614111...
u'tag:google.com,2010:plus:z133s5h4bqbxczdh304cfb5bqovngjr51ko0k'}], u'updated': u'2010-06-14T17:08:31.511Z', u'id': u'tag:google.com,2010:plus-feed:self:posted:118170877744527614111'}

>>> activitylist['items'][0]['title']
First Post!

>>> newitem = {
    u'kind': u'plus#activity', 
    u'object': {
      u'content': u'Just a test note to show that insert is working.', 
      u'type': u'note', 
    }
  }

>>> activity = p.activities().insert(userId='@me', body=newitem)
>>> print activity['published']
2010-07-02T17|28| 59.000Z

```


## Transport Layer ##

Transport will be implemented by httplib2, which supports caching, etags, and a pluggable authentication system.

## Authentication Layer ##

Authentication will be handled by using the python-oauth2 open source library. The demo code shows how that can be wrapped around an httplib2 instance.

## Asynchronous and Batch ##

The `apiclient.build` method starts constructing a simple direct interface to the API, which means that every call executes synchronously, only returning when it is done. This is a good default and easy to learn for people new to the library. There will also be a more complex `apiclient.build_async` method that constructs a more complex interface to the API, which uses the Command pattern for request objects. The asynchronous interface will allow batching request objects into a single batch request, and also support submitting requests objects into an asynchronous queue to be processed. Jeff Scudder already has a good start on such objects in an experimental library.

The asynchronous support should include an optional exponential backoff to retry requests that fail with 408, 503, or 504.

## IDE Support ##

Since the objects in the library are constructed dynamically based on the contents of the discovery document there will be very little native support an IDE will be able to provide as far as doc strings and auto completion. It is an open question if, and how, to support IDEs using this library.

## Discovery ##

The objects returned from `apiclient.build()` are constructed from the information in the discovery document. The discovery document should be cached to avoid introducing excessive latency into situations where the calling application may have a short lifetime, such as when running on App Engine.

## Model ##

The model object is responsible for modifying outgoing requests to add an appropriate `Accept` header and `Content-Type` header if appropriate. It is also responsible for serializing model objects to and from the wire format. Finally it is responsible for providing prototype objects for non-GET methods. For instance, in the example above the activities collection the `insert` method takes a body, which is the model object to insert into the collection. There will be an `activities.insert_body()` method the returns an instance of an object to insert into the collection. The prototype object is built up from the discovery schema for that method.

## Python ##

The oldest supported version will be Python 2.4.

## Third Party Libraries ##

The following third-party libraries will be used by the client library:

  * simplejson - MIT License
  * httplib2 - MIT License
  * python-oauth2 - MIT License
  * uri-templates - Apache 2.0

The simplejson library will only be required for Python 2.4 and 2.5 as simplejson is included in Python 2.6 and later.

## JSON-RPC ##

TDB