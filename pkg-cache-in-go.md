# Notes on pkg/cache in Go

After some discussion on [D56573](https://phabricator.khanacademy.org/D56573), I poked at starting to implement instance-cache in Go. This page includes some notes on what we might do differently.

## Implementation of in-memory caches

### In-memory caches in Python

To understand how to port our in-memory caches to Go, it’s important to understand our in-memory caches in Python: `request_cache` and `instance_cache`. Each of our caches provides two interfaces: raw `get`/`set` and the function-decorator (`@cache`/`@cache_with_key_params_fxn`) interface.

The raw `get`/`set` API, for an in-memory cache, is basically just that of a `dict` (which is indeed the implementation); my first question was: what do our in-memory caches provide beyond that? The answer, as far as I can tell, is:

- Thread-safety (via a lock)
- Automating integration to our unit test resetting (via an `@test_resetter` function that clears the cache)
- For `instance_cache`, expiry (via a timestamp stored with the value; only a few callers use this)
- For `request_cache`, request-locality (by building on `RequestLocal`)

There’s also one caller (`setting_model.py`) that uses functionality to iterate over the cache, but this is sort of a special case. In general, not many callers use this API: there are 13 distinct non-test files that call `request_cache.set`, and 6 for `instance_cache.set`. A lot of places that want to do instance-level initialization just do it with their own file-level global.

The function-decorator interface, of course, is widely used. It provides:

- Basic check/set/return decorator semantics
- Fill-once locking (within a single instance), such that if two threads try to fill the same cache, only one does it
- Converting arguments and function name into a cache key
- Incorporating standard request-/instance-globals like content publish into the cache key (note that `request_cache` has a different implementation of this for speed)
- Utilities to to get/set/delete for function
- A way for a function to return a special object that will be returned to its caller but not cached

### The get/set API in Go

One obvious way to proceed in Go is like in Python: we wrap up a `map` with a lock (for thread-safety). Sadly, in Go, we’re already in trouble: our map’s values, at least, will have to be `interface{}`. While the function-decorator API may be able to cover this up internally (see next section), with `get`/`set`, there’s no way around it: callers will have to do the cast, and hope they get it right.

One improvement we can make right out of the box is to use a `sync.Map` instead of an ordinary `map`. Per the [documentation](https://godoc.org/sync#Map), it’s a `map` that’s thread-safe to access; and our use case seems to fit one of the common ones where it’s faster than a `map` with a lock. But the existence and convenience of `sync.Map` also means that thread-safety is no longer an advantage of using the cache library. There are also more alternatives for thread-safe initialization: `sync.Once` does what a lot of callers will likely want, and would allow those callers to do everything in a type-safe way. (We might want to provide an API with different error handling, as described in this [proposal](https://github.com/golang/go/issues/22098) The implementation is fairly simple, mostly duplicating `sync.Once`.)

For request-cache, we’d likely use the `context.Value` API, which allows us to tie a particular value to the request context. Here, we have some advantage over the builtin: `context.Value` is O(n) in the number of values attached to this context (!) although it’s unclear if the number of raw `set` calls we’d want to make would be high enough for this to matter. The semantics are also a bit different: in place modifications to a pre-created context value propagate up the stack as well as down, and are shared by different call-subtrees of a request; whereas adding values to the context can propagate them only downward, not up. (We probably want the latter semantics for “mutable” request-globals like FMS locale – and indeed those semantics are kinda sorta similar to what `LocalizedTopicTree` and friends aim to provide using `RequestLocal`.)

Note that most callers of `instance_cache.set` are storing their own map or list, and so they already need to do their own locking (or decide they don’t care) for access to its keys. Others use just a single key, and so safe initialization is not at issue; they could more easily do init-time initialization, or `sync.Once`. The only ones which write many separate keys of their own are:

- `content/frozen_model_store.py`: Writes one key per frozen-sha. This one might benefit from its own map, so it could iterate over it to do some pruning, although we haven’t currently implemented that.

- `gcloud/google_api_util.py`: Writes several keys, one per URL. (In practice there are just a few keys used.)

The latter, plus `dev/event_tracker.py`, use expiration.

Originally, I was thinking that with thread-safety more or less gone, the benefit of having somebody else handle the details (expiry, test-resetting) is less than the cost of losing type-checking, especially for cases that can use a concrete `map` type rather than a `map` with `interface{}` or equivalently `sync.Map`. But it’s not clear the latter ever happens, and having automatic test-resetting is actually pretty nice both in terms of boilerplate reduction and decreasing the potential for flaky tests. So in [D56573](https://phabricator.khanacademy.org/D56573) we’re going with instance-cache.

### The decorator API in Go

To write an `@cache` style API, we’ll need to use reflection. Using `MakeFunc` and `Value.Call`, we can construct a variadic function which does what we want while still being mostly typechecked. In particular, our code (omitting considerations like namespacing) will look something like:

```go
func Cache(function interface{}) interface{} {
  functionValue := reflect.ValueOf(wrapped)
  return reflect.MakeFunc(functionValue.Type(), func(args []reflect.Value) []reflect.Value {
    // check
    key = getKey(functionValue, args)
    ret = cache.get(key) // implementations elided
    if ret != nil {
      return ret
    }
    
    // set
    ret = functionValue.Call(args) // TODO: might need to use CallSlice in some cases
    cache.set(key, ret)
    
    // return
    return ret
  }).Interface()
}

// Use like:
func _uncachedExpensive(a int, b string) bool {
  // ...
}
var Expensive = Cache(_uncachedExpensive).(func(int, string) bool)
```

Note that while the `Cache` call itself is not typechecked, as long as its argument and return value are of the same (function) type, the application code is typechecked. (We could probably lint that the argument and return of `Cache` must have the same type.)

Getting namespace parameters is another issue. It seems likely that we’ll need to pull these from the `Context`, which means we have to have access to said `Context`. We may just want to require that cached functions take `Context` as their first argument (which, again, we could lint for).

Note: there’s now a lot more discussion of the `Cache` decorator in [D57247](https://phabricator.khanacademy.org/D57247).

## Argument serialization

We can probably stick with what python does here – namely say instance-cache and request-cache arguments have to be serializable, but we will auto-stringify for shared caches? Of course, we should fix the dict/set sorting (or perhaps just refuse to serialize those as cache keys – it’s usually a bad idea to use any structure that’s not of fixed size).

### Instance cache and its relationship to priming

Having an easy-to-use in-memory cache available for the use of any function anywhere is really nice. It’s so nice that lots of random people throw random things in instance cache that aren’t actually worth caching! Infrastructure has a [periodic task](https://docs.google.com/document/d/1ayKP4pOPMM12ef4VHP3HjsjnF80oDGswiA3hQWLrmEY/edit#) to look for such things, but until such time they just waste memory.  Meanwhile, we go through a bunch of effort to [prefill certain caches](https://khanacademy.atlassian.net/wiki/spaces/INFRA/pages/121700954/Caches%2C+priming%2C+and+their+failure+modes) that we expect every request will need.

My first claim is that anything worth instance-caching is worth prefilling when we can. (This sets aside the issue of caching things per publish commit, which we end up handling specially anyway.) In particular, since each request is equally likely to go to each instance, the fact that a value was recently used doesn’t really change whether it’s likely to be used soon.

Well, mostly; the exceptions will help explain the rule. One is the case where we don’t know a priori whether a value is worth caching; for example some per-topic data may be relevant for some fixed set of topics, and not others; lazy filling lets us discover that list automatically. The other is the case where requests meaningfully change over time due to usage patterns. This is rare for us, but it may become more common with services: while caching data about a particular user that happens to be online is useless with 2000 instances of webapp running, it may be helpful with 10 instances of some particular service.

I still argue, though, that it’s worth encouraging authors using instance-cache to consider whether they should be priming their caches.  We could say you have to explicitly disable priming, although then I worry we just make priming slow. Or we could add some more instrumentation such that we log the hit rates for items some small fraction of the time; since Go is fast this could presumably be done pretty cheaply. (Say on 1% of instances we log, paying the overhead of logging; and on 1% of requests we spawn a goroutine that, in the background, logs all our hit rates.)
