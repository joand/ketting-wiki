Let's get started with Ketting!

The Basics
----------

Ketting has two major primitives that are important to learn:

* A `Resource`, which represents a single endpoint or url.
* A `State`, which represents the result of a `GET` request, or what
  you would submit with a `PUT` request.

Let's assume that we have a REST API that manages blog articles.
Each article is encoded in JSON and might look like this:

```json
{
  "title": "First article",
  "body": "..."
}
```

Our API's root (or home) is at `https://api.example/`, and for now
our goal is to:

1. Grab the article located at `/article/1`
2. Update the title.
3. Submit the result back to the server.

_Note that the following examples are written with Typescript in mind, so if
you use plain Javascript instead, `import` might have to be replaced by
`require`._

```typescript
import { Client } from 'ketting';

// Set up the client
const client = new Client('https://api.example');

// Get the resource. Note that this does not make any HTTP requests.
const myArticleRes = client.go('/article/1');

// Get the state. This does a GET request
const myArticleState = await myArticleRes.get();

// Lets update the title
myArticleState.data.title = 'Hello world v2';

// And save the article again
await myArticleRes.put(myArticleState);
```

The State Object
----------------

The `State` object represents either a response body, or a request body
in cases like `PUT`.

Often you will get a State object this way:

```typescript
const state = await resource.get();
```

The following properties and functions are available on the `State` object:

* `uri` - This contains the absolute URI the state belongs to.
* `data` - This is the parsed response body. Usually the result of
  `JSON.parse()`, but if you received HTML this will be a string, and in the
  case of binaries this will be a `Buffer`.
* `contentHeaders()` - Returns a list of HTTP headers that directly relate or
  describe the state. Notably, all the `Content-*` headers such as
  `Content-Type` and `Content-Language`.
* `timestamp` - When the state was generated.
* `serializeBody()` - Turns the state into an object that can be used in a HTTP
  request.
* `clone()` - Creates a deep copy of the State object.
* `action()` - Covered in [Hypermedia](Hypermedia).
* `links` - All Weblinks that were encoded in the HTTP Link header, or in the
   response body. Covered further in [Hypermedia](Hypermedia).

In short, the State object attempts to describe a full state you might
get back from the server, and provide a convenient API to work with it.

Resources
---------

### PUT, GET and Caching

We covered that `resource.get()` gives you a State object, and `.put()` can
send the state back to the server.

For this to work with Ketting, the assumption is that *what you `PUT`* to
an endpoint on a server more or less has the same shape as what you would
`GET` immediately after.

This assumption allows Ketting to do a lot of caching.

In the following example, only 1 GET request is done:

```typescript
const state1 = await resource.get();

// Get the state again, but this time it's served from cache.
// Note that this state object is a *copy* and not the same object.
const state2 = await resource.get();

// Make a change
state2.data.title = 'Hello again';

// Submit the state back to the server
await resource.put(state2);

// When doing a PUT request, ketting will immediately place the new
// state object in the cache. The following call to .get() is again
// served from cache, but it will reflect the state from the last
// time we called .put()

const state3 = await resource.get();
// Will output 'hello again'
console.log(state3.data.title);
```

The following three cache-related operations are also available:

```typescript
// This cleans the state cache for the resource.
resource.clearCache();

// Updates the internal cache *without* doing a request.
resource.updateCache(state);

// Do a GET request but skip the cache
const state = await resource.refresh();
```

### A note on PUT (and other methods)

You don't need a full State object to do a `PUT` request. Sometimes
you just want to do a fresh `PUT` request with data from your
application.

The `PUT` request also accepts an incomplete state object as such:

```typescript
await resource.put({
  data: {
    foo: 'This is the new body for PUT'
  }
});
```

### Creating resources with POST

A common pattern in REST apis is to create new resources with `POST`.
Ketting has a special method for this: `postFollow()`.

For this to work correctly, your server should:

1. Accept `POST` requests
2. If successful, return a `201 Created` status code.
3. The server must also return a `Location` header that points to the newly
   created resource.

Many REST services work this way, and if yours does too, this is how this
might work:

```typescript
const newArticle = {
  title: 'Second post!',
  body: '....',
};

const newResource = await collectionResource.postFollow({
  data: newArticle
});
```

The `postFollow()` function will return a new `Resource` object with
all the functions you expect, like `.get()` and `.post()`.

### RPC-like operations with POST and PATCH

If you just want to do a regular `POST` request, but to do an RPC-like
operation like "Sending an email" or "Submitting a form", there is also
a regular `.post()` method that takes a `State` and returns a `State`,
giving you access to the response body:

```typescript
const result = resource.post({
  data: {
    to: 'mom@example.org',
    subject: 'Sup'
  }
});
```

Similarly, resources have a `.patch()` method which behaves the exact
same.

Calling `.post()`, `.patch()`, `.postFollow()` or any other "unsafe" HTTP
method will expire the state cache.

### DELETE

Delete requires neither a request body, nor returns a response body:

```typescript
await resource.delete();
```

### Events

It's possible to subscribe to state changes. To get notified whenever the
State cache gets updated, subscribe to the `update` event:

```typescript
resource.on('update', state => {

  console.log('The resource just updated');
  console.log('New data: ', state.data);

})
```

Other events:

* `delete`, which gets triggered upon calling `DELETE`.
* `stale` gets triggered when the cache is no longer deemed valid.


### fetch()

Resources also have a `fetch()` function that behaves similar to the normal
[browser fetch][fetch], but it does not take an `uri` argument. This can be
useful for more complex requests.

When using this `fetch()` function, the cache will still be expired for
'unsafe' methods.

### go()

You saw `client.go()` before, but resources also have a `.go()` method. This
method does the same thing, but when passing a relative uri here, the absolute
uri will be calculated based on the resource's uri.

### follow() / followAll()

This set of functions is used for Ketting's hypermedia features, and are
covered in the [Hypermedia](Hypermedia) docs.

Next steps
----------

Depending on what your goals are, these are good next reads:

* Ketting's [Hypermedia](Hypermedia) features.
* [Authentication](Authentication).
* [React integration](React)
* [Optimizing many requests](Optimizing).

[fetch]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
