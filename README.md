PhlyRestfully: ZF2 Module for JSON REST Services
================================================

This module provides structure and code for quickly implementing RESTful APIs
that use JSON as a transport.

It allows you to create RESTful JSON APIs that use the following standards:

- [HAL](http://tools.ietf.org/html/draft-kelly-json-hal-03), used for creating
  hypermedia links
- [Problem API](http://tools.ietf.org/html/draft-nottingham-http-problem-02),
  used for reporting API problems

Resources
---------

A generic Resource class is provided, which provides the following operations:

- `create($data)`
- `update($id, $data)`
- `replaceList($data)`
- `patch($id, $data)`
- `delete($id)`
- `deleteList($data = null)`
- `fetch($id)`
- `fetchAll()`

Each method:

- Ensures the incoming arguments are well-formed (primarily that `$data`
  is an `array` or `object`)
- Triggers an event with the incoming data
- Pulls the last event listener result, and ensures it is well-formed for the
  operation (typically, returns an `array` or `object`; in the case of
  `delete` and `deleteList`, looks for a `boolean`; in the case of `fetchAll`,
  looks for an `array` or `Traversable`)

As such, your job is primarily to create a `ListenerAggregate` for the Resource
which performs the actual persistence operations.

Controller
----------

A generic ResourceController is provided which intercepts incoming requests
and passes data to the Resource. It then inspects the result to generate an
appropriate response.

In cases of errors, a Problem API response payload is generated.

When a resource or collection is returned, a HAL payload is generated with
appropriate links.

In all cases, appropriate HTTP response status codes are generated.

The controller expects you to inject the following:

- Resource
- Route name that resolves to the resource
- Event identifier for allowing attachment via the shared event manager; this
  is passed via the constructor (optional; by default, listens to
  `PhlyRestfully\ResourceController`)
- "Accept" criteria for use with the `AcceptableViewModelSelector` (optional;
  by default, assigns any `*/json` requests to the `RestfulJsonModel`)
- HTTP OPTIONS the service is allowed to respond to, for both collections and
  individual resources (optional; head and options are always allowed; by default,
  allows get and post requests on collections, and delete, get, patch, and put
  requests on resources)
- Page size (optional; for paginated results. Defaults to 30.)

Tying it Together
-----------------

You will need to create at least one factory, and potentially several.

Absolutely required is a unique controller factory for the
ResourceController. As noted in the previous section, you will have to inject
several dependencies. These may be hard-coded in your factory, or pulled as, or 
from, other services.

As a quick example:

```php
'PasteController' => function ($controllers) {
    $services   = $controllers->getServiceLocator();
    $events     = $services->get('EventManager');
    $listener   = new \PasteResourceListener(new PasteMongoAdapter);
    $resource   = new \PhlyRestfully\Resource();
    $resource->setEventManager($events);
    $events->attach($listener);

    $controller = new \PhlyRestfully\ResourceController();
    $controller->setResource($resource);
    $controller->setRoute('paste/api');
    $controller->setCollectionHttpOptions(array(
        'GET',
        'POST',
    ));
    $controller->setResourceHttpOptions(array(
        'GET',
    ));
    return $controller;
}
```

The above example instantiates a listener directly, and attaches it to the
event manager instance of a new Resource intance. That resource instance then
is attached to a new `ResourceController` instance, and the route and HTTP
OPTIONS are provided. Finally, the controller instance is returned.

Routes
------

You should create a segment route for each resource that looks like the
following:

```
/path/to/resource[/[:id]]
```

This single route will then be used for all operations.

If you want to use a route with more segments, and ensure that all captured
segments are present when generating the URL, you will need to hook into the
`HalLinks` plugin. As an example, let's consider the following route:

```
/api/status/:user[/:id]
```

The "user" segment is required, and should always be part of the URL. However,
by default, the `ResourceController` and the `RestfulJsonRenderer` will not have
knowledge of matched route segments, and will not tell the `url()` helper to
re-use matched parameters.

`HalLinks`, however, allows you to attach to its `createLink` event, which gives
you the opportunity to provide route parameters. As an example, consider the
following listeners:

```php
$user    = $matches->getParam('user');
$helpers = $services->get('ViewHelperManager');
$links   = $helpers->get('HalLinks');
$links->getEventManager()->attach('createLink', function ($e) use ($user) {
    $params = $e->getParam('params');
    $params['user'] = $user;
});
```

The above would likely happen in a post-routing listener, where we know we
routed to a specific controller, and can have access to the route matches.
It retrieves the "user" parameter from the route first. Then it retrieves the
`HalLinks` plugin from the view helpers, and attaches to its `createLink`
event; the listener simply assigns the user to the parameters -- which are then
passed to the `url()` helper when creating a link.


Collections
-----------

Collections are resources, too, which means they may hold more than simply the
set of resources they encapsulate.

By default, the `ResourceController` simply returns a `HalCollection` with the
collection of resources; if you are using a paginator for the collection, it will
also set the current page and number of items per page to render.

You may want to name the collection of resources you are representing. By default,
we use "items" as the name; you should use a semantic name. This can be done
by either directly setting the collection name on the `HalCollection` using the
`setCollectionName()` method, or calling the same method on the controller.

You can also set additional attributes. This can be done via a listener; 
typically, a post-dispatch listener, such as the following, would be a
reasonable time to alter the collection instance. In the following, we update
the collection to include the count, number per page, and type of objects
in the collection.

```php
$events->attach('dispatch', public function ($e) {
    $result = $e->getResult();
    if (!$result instanceof RestfulJsonModel) {
        return;
    }
    if (!$result->isHalCollection()) {
        return;
    }
    $collection = $result->getPayload();
    $paginator  = $collection->collection;
    $collection->setAttributes(array(
        'count'         => $paginator->getTotalItemCount(),
        'per_page'      => $collection->pageSize,
        'resource_type' => 'status',
    ));
}, -1);
```

Embedding Resources
-------------------

To follow the HAL specification properly, when you embed resources within
resources, they, too, should be rendered as HAL resources. As an example,
consider the following object:

```javascript
{
    "status": "this is my current status",
    "type": "text",
    "user": {
        "id": "matthew",
        "url": "http://mwop.net",
        "github": "weierophinney"
    },
    "id": "afcdeb0123456789afcdeb0123456789"
}
```

In the above, we have an embedded "user" object. In HAL, this, too, should
be treated as a resource.

To accomplish this, simply assign a `HalResource` value as a resource value.
As an example, consider the following pseudo-code for the above example:

```php
$status = new Status(array(
    'status' => 'this is my current status',
    'type'   => 'text',
    'user'   => new HalResource(new User(array(
        'id'     => 'matthew',
        'url'    => 'http://mwop.net',
        'github' => 'weierophinney',
    ), 'matthew', 'user')),
));
```

When this object is used within a `HalResource`, it will be rendered as an
embedded resource:

```javascript
{
    "_links": {
        "self": "http://status.dev:8080/api/status/afcdeb0123456789afcdeb0123456789"
    },
    "status": "this is my current status",
    "type": "text",
    "id": "afcdeb0123456789afcdeb0123456789",
    "_embedded": {
        "user": {
            "_links": {
                "self": "http://status.dev:8080/api/user/matthew"
            },
            "id": "matthew",
            "url": "http://mwop.net",
            "github": "weierophinney"
        },
    }
}
```

This will work in collections as well.

I recommend converting embedded resources to `HalResource` instances either
during hydration, or as part of your `Resource` listener's mapping logic.

Hydrators
---------

You can specify hydrators to use with the objects you return from your resources
by attaching them to the `HalLinks` view helper/controller plugin. This can be done
most easily via configuration, and you can specify both a map of class/hydrator
service pairs as well as a default hydrator to use as a fallback. As an example,
consider the following `config/autoload/phlyrestfully.global.config.php` file:

```php
return array(
    'phlyrestfully' => array(
        'renderer' => array(
            'default_hydrator' => 'Hydrator\ArraySerializable',
            'hydrators' => array(
                'My\Resources\Foo' => 'Hydrator\ObjectProperty',
                'My\Resources\Bar' => 'Hydrator\Reflection',
            ),
        ),
    ),
    'service_manager' => array('invokables' => array(
        'Hydrator\ArraySerializable' => 'Zend\Stdlib\Hydrator\ArraySerializable',
        'Hydrator\ObjectProperty'    => 'Zend\Stdlib\Hydrator\ObjectProperty',
        'Hydrator\Reflection'        => 'Zend\Stdlib\Hydrator\Reflection',
    ));
);
```

The above specifies `Zend\Stdlib\Hydrator\ArraySerializable` as the default
hydrator, and maps the `ObjecProperty` hydrator to the `Foo` resource, and the
`Reflection` hydrator to the `Bar` resource. Note that you need to define
invokable services for the hydrators; otherwise, the service manager will be
unable to resolve the hydrator services, and will not map any it cannot resolve.

Specifying Alternate Identifiers For URL Assembly
-------------------------------------------------

With individual resource endpoints, the identifier used in the URI is given to
the `HalResource`, regardless of the structure of the actual resource object.
However, with collections, the identifier has to be derived from the individual
resources they compose.

If you are not using the key or property "id", you will need to provide a
listener that will derive and return the identifier. This is done by attaching
to the `getIdFromResource` event of the `PhlyRestfully\Plugin\HalLinks` class.

Let's look at an example. Consider the following resource structure:

```javascript
{
    "name": "mwop",
    "fullname": "Matthew Weier O'Phinney",
    "url": "http://mwop.net"
}
```

Now, let's consider the following listener:

```php
$listener = function ($e) {
    $resource = $e->getParam('resource');
    if (!is_array($resource)) {
        return false;
    }

    if (!array_key_exists('name', $resource)) {
        return false;
    }

    return $resource['name'];
};
```

The above listener, on encountering the resource, would return "mwop", as that's
the value of the "name" property.

There are two ways to attach to this listener. First, we can grab the `HalLinks`
plugin/helper, and attach directly to its service manager:

```php
// Assume $services is the application ServiceManager instance
$helpers = $services->get('ViewHelperManager');
$links   = $helpers->get('HalLinks');
$links->getEventManager()->attach('getIdFromResource', $listener);
```

Alternately, you can do so via the `SharedEventManager` instance:

```php
// Assume $services is the application ServiceManager instance
$sharedEvents = $services->get('SharedEventManager');

// or, if you have access to another event manager instance:
$sharedEvents = $events->getSharedManager();

// Then, connect to it:
$sharedEvents('PhlyRestfully\Plugin\HalLinks', 'getIdFromResource', $listener);
```

Controller Events
-----------------

Each of the various REST endpoint methods - `create()`, `delete()`,
`deleteList()`, `get()`, `getList()`, `patch()`, `update()`, and `replaceList()`
\- trigger both a `{methodname}.pre` and a `{methodname}.post` event. The "pre"
event is executed after validating arguments, and will receive any arguments
passed to the method; the "post" event occurs right before returning from the
method, and receives the same arguments, plus the resource or collection, if
applicable.

These methods are useful in the following scenarios:

- Specifying custom HAL links
- Aggregating additional request parameters to pass to the resource object

As an example, if you wanted to add a "describedby" HAL link to every resource
or collection returned, you could do the following:

```php
// Methods we're interested in
$methods = array(
    'create.post',
    'get.post',
    'getList.post',
    'patch.post',
    'update.post',
    'replaceList.post',
);
// Assuming $sharedEvents is a ZF2 SharedEventManager instance
$sharedEvents->attach('My\Namespaced\ResourceController', $methods, function ($e) {
    $resource = $e->getParam('resource', false);
    if (!$resource) {
        $resource = $e->getParam('collection', false);
    }

    if (!$resource instanceof \PhlyRestfully\LinkCollectionAwareInterface) {
        return;
    }

    $link = new \PhlyRestfully\Link('describedby');
    $link->setRoute('api/docs');
    $resource->getLinks()->add($link);
});
```

Upgrading
=========

If you were using version 1.0.0 or earlier (the version presented at PHP
Benelux 2013), you will need to make some changes to your application to get it
to work.

- First, the terminology has changed, as have some class names, to reference
  "resources" instead of "items"; this is more in line with RESTful terminology.
    - As such, if you had any code using `PhlyRestfully\HalItem`, it should now
      reference `PhlyRestfully\HalResource`. Similarly, in that class, you will
      access the actual resource object now from the `resource` property
      instead of the `item` property. (This should only affect those post-1.0.0).
    - If you want to create link for an individual resource, use the
      `forResource` method of `HalLinks`, and not the `forItem` method.
    - `InvalidItemException` was renamed to `InvalidResourceException`.
- A number of items were moved from the `RestfulJsonModel` to the
  `RestfulJsonRenderer`.
    - Hydrators
    - The flag for displaying exception backtraces; in fact, you can use
      the `view_manager.display_exceptions` configuration setting to set
      this behavior.
- All results from the `ResourceController` are now pushed to a `payload`
  variable in the view model. 
    - Additionally, `ApiProblem`, `HalResource`, and `HalCollection` are
      first-class objects, and are used as the `payload` values.
- The `Links` plugin was renamed to `HalLinks`, and is now also available as
  a view helper.


LICENSE
=======

This module is licensed using the BSD 2-Clause License:

```
Copyright (c) 2013, Matthew Weier O'Phinney
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

- Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.
- Redistributions in binary form must reproduce the above copyright notice,
  this list of conditions and the following disclaimer in the documentation
  and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```
