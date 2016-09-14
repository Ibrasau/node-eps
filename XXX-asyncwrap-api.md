| Title  | AsyncHook API |
|--------|---------------|
| Author | @trevnorris   |
| Status | DRAFT         |
| Date   | 2016-09-14    |

## Description

Since its initial introduction along side the `AsyncListener` API, the internal
class `AsyncWrap` has slowly evolved to ensure a generalized API that would
serve as a solid base for module authors who wished to add listeners to the
event loop's life cycle. Some of the use cases `AsyncWrap` has covered are long
stack traces, continuation local storage, profiling of asynchronous requests
and resource tracking. Though the public API is now exposed as `'async_hooks'`.

It must be clarified that the `AsyncHook` API is not meant to abstract away
Node's implementation details, and is in fact intentional. Observing how
operations are performed, not simply how they appear to perform from the public
API, is a key point in its usability. For a real world example, a user was
having difficulty figuring out why their `http.get()` calls were taking so
long. Because `AsyncHook` exposed the `dns.lookup()` request to resolve the
host being made by the `net` module it was trivial to write a performance
measurement that exposed the hidden culprit. Which was that DNS resolution was
taking much longer than expected.

A small amount of abstraction is necessary to accommodate the conceptual nature
of the API. For example, the `HTTPParser` is not destructed but instead placed
into an unused pool to be used later. Even so, at the time it is placed into
the pool the `destroy()` callback will run and then the id assigned to that
instance will be removed. If that resource is again requested then it will be
assigned a new id and will run `init()` again.


## Goals

The intent for the initial release is to provide the most minimal set of API
hooks that don't inhibit module authors from writing tools addressing anything
in this problem space. In order to remain minimal, all potential features for
initial release will first be judged on whether they can be achieved by the
existing public API. If so then such a feature won't be included.

If the feature cannot be done with the existing public API then discussion will
be had on whether requested functionality should be included in the initial
API. If so then the best course of action to include the capabilities of the
request will be discussed and performed. Meaning, the resulting API may not be
the same as initially requested, but the user will be able to achieve the same
end result.

Performance impact of `AsyncHook` should be zero if not being used, and near
zero while being used. The performance overhead of `AsyncHook` callbacks
supplied by the user should account for essentially all the performance
overhead.


## Terminology

Because of the potential ambiguity for those not familiar with the terms
"handle" and "request" they should be well defined.

Handles are a reference to a system resource. Some resources are a simple
identifier. For example, file system handles are represented by a file
descriptor. Other handles are represented by libuv as a platform abstracted
struct, e.g. `uv_tcp_t`. Each handle can be continually reused to access and
operate on the referenced resource.

Requests are short lived data structures created to accomplish one task. The
callback for a request should always and only ever fire one time. Which is when
the assigned task has either completed or encountered an error. Requests are
used by handles to perform these tasks. Such as accepting a new connection or
writing data to disk.


## API

### Overview

Here is a simple overview of the entire API. All of this API is explained in
more detail further down.

```js
// Standard way of requiring a module. Snake case follows core module
// convention.
const async_hooks = require('async_hooks');

// Return the id of the current execution context. Useful for tracking state
// and retrieving the handle of the current parent without needing to use an
// AsyncHook.
const id = async_hooks.currentId();

// Create a new instance of AsyncHook. All of these callbacks are optional.
const asyncHook = async_hooks.createHook({ init, before, after, destroy });

// Allow callbacks of this AsyncHook instance to fire. This is not an implicit
// action after running the constructor, and must be explicitly run to begin
// executing callbacks.
asyncHook.enable();

// Disable listening for new asynchronous events. Though this will not prevent
// callbacks from firing on asynchronous chains that have already run within
// the scope of an enabled AsyncHook instance.
asyncHook.disable();

// The following are the callbacks that can be passed to createHook().

// init() is called during object construction. The handle may not have
// completed construction when this callback runs. So all fields of the
// handle referenced by "id" may not have been populated.
function init(id, type, parentId, handle) { }

// before() is called just before the handle's callback is called. It can be
// called 0-N times for handles (e.g. TCPWrap), and should be called exactly 1
// time for requests (e.g. FSReqWrap).
function before(id) { }

// after() is called just after the handle's callback has finished, and will
// always fire. Except in the case of an uncaught exception.
function after(id) { }

// destroy() is called when an AsyncWrap instance is destroyed. In cases like
// HTTPParser where the resource is reused, or timers where the handle is only
// a JS object, destroy() will be triggered manually soon after after() has
// completed.
function destroy(id) { }

// The following calls are specific to the embedder API. If any hook emitter is
// used then they must all be used to make sure state proceeds correctly.

// Return new unique id for a constructing handle.
const id = async_hooks.newUid();

// Set the current global id.
async_hooks.setCurrentId(id);

// Call the init() callbacks. Returns an instance of AsyncEvent that will be
// used in other emit calls.
async_hooks.emitInit(id, handle, type[, parentId]);

// Call the before() callbacks. The reason for requiring both arguments is
// explained in further detail below.
async_hooks.emitBefore(id);

// Call the after() callbacks.
async_hooks.emitAfter(id);

// Call the destroy() callbacks.
async_hooks.emitDestroy(id);
```


### `async_hooks`

The object returned from `require('async_hooks')`.


#### `async_hooks.currentId()`

Return the id of the current execution context. Useful to track the
asynchronous execution chain. It can also be used to propagate state without
needing to use the `AsyncHook` constructor.


#### `async_hooks.createHook(callbacks)`

* `callbacks` {Object}
* Return: {AsyncHook}

`createHook()` returns an `AsyncHook` instance that contains information about
the callbacks that are to fire during specific asynchronous events in the
lifetime of the event loop. The focal point of these calls centers around the
lifetime of the `AsyncWrap` C++ class. These callbacks will also be called to
emulate the lifetime of handles and requests that do not fit this model. For
example, `HTTPParser` instances are recycled to improve performance. So the
`destroy()` callback would be called manually after a connection is done using
it. Just before it's placed back into the unused resource pool.

All callbacks are optional. So, for example, if only resource cleanup needs to
be tracked then only the `destroy()` callback needs to be passed. The
specifics of all functions that can be passed to `callbacks` is in the section
`Hook Callbacks`.

**Error Handling**: If any `AsyncHook` callbacks throw the application will
print the stack trace and exit. The exit path does follow that of any uncaught
exception, except for the fact that it is not catchable by an uncaught
exception handler, so any `'exit'` callbacks will fire. Unless the application
is run with `--abort-on-uncaught-exception`. In which case a stack trace will
be printed and the application will exit, leaving a core file.

The reason for this error handling behavior is that these callbacks are running
at potentially volatile points in an object's lifetime. For example during
class construction and destruction. Because of this, it is deemed necessary to
bring down the process quickly as to prevent an unintentional abort in the
future. This is subject to change in the future if a comprehensive analysis is
performed to ensure an exception can follow the normal control flow without
unintentional side effects.


#### `asyncHook.enable()`

* Return: {AsyncHook} A reference to `asyncHook`.

Enable the callbacks for a given `AsyncHook` instance. When a hook is enabled
it is added to a pool of hooks to execute. These hooks do not propagate with
a single asynchronous chain but will be completely disabled when `disable()`
is called.

Current reason for this is because of the performance impact of tracking hooks
individually against every handle. Combined with the fact that we're now
tracking `process.nextTick()` calls. Which are used liberal throughout core.
These two things placed substantial burden on performance. If full support
of `nextTick()` was removed or somehow modified then the individual branching
may be possible.

Callbacks are not implicitly enabled after an instance is created. The reason
for this is to not make any assumptions about the user's use case. Since
constructing the `asyncHook` during startup, but not using it until later is, a
reasonable use case. This API is meant to err on the side of requiring explicit
instructions from the user.

To help simplify this `enable()` returns the instance of itself so the calls
can be chained.

```js
const async_hooks = require('async_hooks');

async_hooks.createHook(callbacks).enable();
```


#### `asyncHook.disable()`

Disable the callbacks for a given `AsyncHook` instance from the global pool of
hooks to be executed. Once a hook has been disabled it will not fire again
until enabled.


### Hook Callbacks

Key events in the lifetime of asynchronous events have been categorized into
four areas. On instantiation, before/after the callback is called and when the
instance is destructed. For cases where resources are reused, instantiation and
destructor calls are emulated.


#### `init(id, type, parentId, handle)`

* `id` {Number}
* `type` {String}
* `parentId` {Number}
* `handle` {Object}

Called when a class is constructed that has the possibility to trigger an
asynchronous event. This does not mean the instance will trigger a
`before()`/`after()` event before `destroy()` is called. Only that the
possibility exists.

This behavior can be observed by doing something like opening a resource then
closing it before the resource can be used. The following snippet demonstrates
this.

```js
require('net').createServer().listen(8080, function() { this.close() });
```

Every instance is assigned a unique id. Taking advantage of the fraction space
in the 64-bit IEEE 754 that ECMAScript defines as the **Number** type, the
number of id's that can be assigned are `2^53 - 1` (also defined as
`Number.MAX_SAFE_INTEGER`). At this size node can assign a new id every 100
nanoseconds and not run out for over 28 years. Because of this circumstance it
is not deemed necessary to use an alternative approach that would account for
the id to wrap around.

The `type` is a String that represents the type of handle that caused `init()`
to fire. Generally it will be the name of the handle's constructor. Some
examples include `TCP`, `GetAddrInfo` and `HTTPParser`. Users will be able to
define their own `type` when using the public embedder API.

`parentId` is the unique id of either the async resource at the top of the JS
stack when `init()` was called, or the originator of the new resource. For
example, when a connection is made to a TCP server there is no JS stack
available when the connection's constructor is executed. Meaning there would be
no `parentId` available on the JS stack. Instead the TCP server's unique id is
manually passed to the client's constructor and propagated to the `init()`'s
`parentId`.

If `parentId` is 1 then the call parent is the bootstrap phase of the
application. If the `parentId` is 0 then it points to the void and there's a
problem.

The constructed `handle` is passed to `init()`. The structure of any object
passed to `init()` has no guarantee of stability, even through patch updates.
It's meant to be used for debugging the application when additional, and
specific, information is needed. Make sure not to hold a reference to this
indefinitely or else the GC won't be able to collect it after node has removed
its own reference to the handle.


#### `before(id)`

* `id` {Number}

Called just before the return callback is called after completing an
asynchronous request. Or called on handles with events such as receiving a new
connection. For requests, such as `fs.open()`, this should be called exactly
once. For handles, such as a TCP server, this may be called 0-N times.


#### `after(id)`

* `id` {Number}

Called immediately after the return callback is completed. If a callback throws
but is not caught then the process will exit immediately without calling
`after()`.


#### `destroy(id)`

* `id` {Number}

Called either when the class destructor is run (either when manually triggered
or cleaned up by the garbage collector), or if the resource is manually marked
as free. For core classes that have a destructor the callback will fire during
deconstruction. If called via `emitDestroy()` then it will be called when the
implementor deems the resource freed.  In the case of shared or cached
resources, such as `HTTPParser`, `destroy()` will be called manually when the
TCP connection is no longer in need of it. Every subsequent use of a shared
resource will have a new unique id.


## Embedder API

Library developers that handle their own I/O will need to hook into the
`AsyncWrap` API so that all the appropriate callbacks are called. To
accommodate this both a C++ and JS API is provided.


### Native API

```cpp
// Helper class users can inherit from, but is not necessary. It will
// automatically call all four callbacks.
class AsyncHook {
  public:
    AsyncHook(Local<Object> handle, const char* name);
    ~AsyncHook();
    MaybeLocal<Value> MakeCallback(
        const Local<Function> callback,
        int argc,
        Local<Value>* argv);
    Local<Object> handle();
    int64_t get_uid();
  private:
    AsyncHook();
    Persistent<Object> handle_;
    const char* name_;
    int64_t uid_;
}

// Returns the id of the current execution context. If the return value is
// zero then no execution has been set. This will happen if the user handles
// I/O from native code.
int64_t node::GetCurrentId();

// If the native API doesn't inherit from the helper class then the callbacks
// must be triggered manually. This triggers the init() callback. The return
// value is the uid assigned to the handle.
// TODO(trevnorris): This always needs to be called so that the handle can be
// placed on the Map for future query.
int64_t node::EmitAsyncHandleInit(Local<Object> handle);

// Emit the destroy() callback.
void node::EmitAsyncHandleDestroy(int64_t id);

// An API specific to emit before/after callbacks is unnecessary because
// MakeCallback will automatically call them for you.
MaybeLocal<Value> node::MakeCallback(Isolate* isolate,
                                     Local<Object> recv,
                                     Local<Function> callback,
                                     int64_t id,
                                     int argc,
                                     Local<Value>* argv);
```


### JS API

There is also a JS API to allow for conceptual correctness. For example, if a
request batches calls under the hood this would still allow for each request
to trigger the callbacks individually so that the callback associated with
each bach request execute inside the correct parent.

This cannot be enforced in any way. It is up to the implementer to make sure
all of the callbacks are placed and called at the correct time.


#### `async_hooks.newUid()`

* Return: {Number}

Return a new unique id for a given handle from the same pool of unique id's
used for all internally created handles.

Generally this should be used during object construction. e.g.:

```js
class MyClass {
  constructor() {
    this._event_id = async_hooks.newUid();
  }
}
```


#### `async_hooks.emitInit(id, handle, type[, parentId])`

* `id` {Number} Generated by calling `newUid()`
* `handle` {Object}
* `type` {String}
* `parentId` {Number} **Default:** `currentId()`
* Return: {Undefined}

Emit that a handle is being initialized. `id` should be a value returned by
`async_hooks.newUid()`. Usage will probably be as follows:

```js
class Foo {
  constructor() {
    this.event_id = async_hooks.newUid();
    async_hooks.emitInit(this.event_id, this, 'Foo');
  }
}
```

In the unusual circumstance that the embedder needs to define a different
parent id than `currentId()`, they can pass in that id manually.

It is suggested to have `emitInit()` be the last call in the object's
constructor.


#### `async_hooks.emitBefore(id)`

* `id` {Number} Generated by `newUid()`
* Return: {Undefined}

Trigger listening `before()` callbacks that the `handle` is about to enter its
call stack.

It's currently necessary for embedders to manually do a little state
management.  The following is a template for how the global id state should be
handled:

```js
MyThing.prototype.done = function done() {
  // First call the before() callbacks. So currentId() shows the id of the
  // handle wrapping the id that's been passed.
  async_hooks.emitBefore(this.asyncId);

  // Store the current parent id then set the global current id to the id
  // that was assigned when this object was instantiated.
  const previous_id = async_hooks.currentId();
  async_hooks.setCurrentId(this.asyncId);

  // Run the callback.
  this.callback();

  // Restore the old id.
  async_hooks.setCurrentId(previous_id);

  // Call after() callbacks now that the old id has been restored.
  async_hooks.emitAfter(this.asyncId);
};
```

Reason for this is to simplify the internal implementation and reduce overhead.
If a method can be found to pretty-ify this without incurring additional cost
then it'll probably be used.


#### `async_hooks.emitAfter(id)`

* `id` {Number} Generated by `newUid()`
* Return: {Undefined}

Trigger listening `after()` callbacks that the `handle` has left it's call
stack.

The `id` call stack should always properly unwind. No two `emitAfter()` calls
should be crossed. For example:

```
init # Foo
init # Bar
...
before # Foo
before # Bar
after # Foo   <- Should be called after Bar
after # Bar
```

There is no known scenario where this is acceptable. If one is found then this
restriction may be lifted.


#### `async_hooks.emitDestroy(id)`

* `id` {Number} Generated by `newUid()`
* Return: {Undefined}

Trigger listening `destroy()` callbacks that the `handle`'s resources are no
longer being used, and in cases where the `handle` is attached to native
resources the underlying class may have been destructed.


## API Exceptions

### Reused Resources

Resources like `HTTPParser` are reused throughout the lifetime of the
application. This means node will have to synthesize the `init()` and
`destroy()` calls. Also the id on the class instance will need to be changed
every time the resource is acquired for use.

Though for a shared resource like `TimerWrap` we're not concerned with reassigning
the id because it isn't directly used by the user. Instead it is used internally
for JS created handles that are all assigned their own unique id, and each of
these JS handles are linked to the `TimerWrap` instance.


## Notes

### Native Modules

Existing native modules that call `node::MakeCallback` today will always have
their parent id as `1`. Which is the root id. There will be two native APIs
that users can use. One will be a static API and another will be a class that
users can inherit from.


### Promises

Node currently doesn't have sufficient API to notify calls to Promise
callbacks. In order to do so node would have to override the native
implementation. We are currently working with the V8 team to gain access to
enough internals to integrate Promises into the AsyncWrap API at a future date.


### Immediate Write Without Request

When data is written through `StreamWrap` node first attempts to write as much
to the kernel as possible. If all the data can be flushed to the kernel then
the function exits without creating a `WriteWrap` and calls the user's
callback in `nextTick()`. Meaning detection of the write won't be as
straightforward as watching for a `WriteWrap`.
