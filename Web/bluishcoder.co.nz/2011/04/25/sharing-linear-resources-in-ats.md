# BLUISH CODER: ATSにおける共有した線形リソース

(元記事は http://bluishcoder.co.nz/2011/04/25/sharing-linear-resources-in-ats.html です)

My previous post on [converting C programs to ATS](http://bluishcoder.co.nz/2011/04/24/converting-c-programs-to-ats.html) had an example of passing a linear resource to a callback function.
The code looked like:

```ats
typedef evhttp_callback (t1:viewtype) = (!evhttp_request1, !t1) -<fun1> void
extern fun evhttp_set_cb {a:viewtype} (http: !evhttp1,
                                       path: string,
                                       callback: evhttp_callback (a),
                                       arg: !a): int = "mac#evhttp_set_cb"
...
val _ = evhttp_set_cb {event_base1} (http,
                                     "/quit",
                                     lam (req, arg) => ignore(event_base_loopexit(arg, null)),
                                     base)
```

The base argument to be passed to the callback is an instance of a linear type:

```ats
absviewtype event_base (l:addr)
viewtypedef event_base0 = [l:addr | l >= null ] event_base l
viewtypedef event_base1 = [l:addr | l >  null ] event_base l
```

For some callback programming I need to pass more than one argument to the callback.
I needed a way to package up a number of linear resources, pass them to the callback, and have the callback possibly consume some of them.
I took a stab at solving this problem myself before [asking for advice](http://sourceforge.net/p/ats-lang/mailman/message/27404524/) on the [ats-lang-users mailing list](https://lists.sourceforge.net/lists/listinfo/ats-lang-users) for comments.
[Hongwei Xi replied](http://sourceforge.net/p/ats-lang/mailman/message/27405027/) with some great advice on how to deal with the issue.

I’ll go through a cut down example of the callback program, my first attempt at solving it, and then onto a solution based on Hongwei’s reply.

## Callback example

The following program, [callback1.dats](http://bluishcoder.co.nz/ats/callback1.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback1.html)) is a small test case for the callback program where I pass one linear resource to the callback as in my libevent example referred to above.
The main code body is:

```ats
absviewtype resource (l:addr)
viewtypedef resource1 = [l:addr | l > null] resource l
extern fun resource_new():resource1 = "mac#resource_new"
extern fun resource_use(r: !resource1):void = "mac#resource_use"
extern fun resource_free(r: resource1):void = "mac#resource_free"

typedef callback = (!resource1) -<fun1> void
extern fun do_something(f: callback, arg: !resource1):void = "mac#do_something"

implement main() = let
  val r = resource_new()
  val () = do_something(lam (r) => resource_use(r), r) 
  val () = resource_free(r)
in
  ()
end
```

In this example I pass my linear object, r, to `do_something` along with a callback.
`do_something` then calls the callback passing my linear object as an argument.
The callback uses and does not consume the object.
My next task was to tackle the issue of passing multiple linear objects.
Something like:

```ats
implement main() = let
  var r = resource_new()
  var r2 = resource_new()
  val () = do_something(lam (r_and_r2) => (resource_use(r);resource_use(r2)),
                        ...want_to_pass_r_and_r2_here...)
  val () = resource_free(r)
  val () = resource_free(r2)
in
  ()
end
```

## Linear Closure Containing Linear Types

My first attempt at passing multiple values to the callback was to use a closure as the argument to the callback function.
I could then close over the multiple objects.
This code is in [callback2.dats](http://bluishcoder.co.nz/ats/callback2.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback2.html)).
The main changes are:

```ats
viewtypedef func = () -<lincloptr1> void
extern fun do_something(f: (func) -<fun1> void, arg: func):void = "mac#do_something"

fun callback(f: func):void = let val () = f() in cloptr_free(f) end

implement main() = let
  extern prfun __borrow {l:addr}
                        (pf: ! resource1 @ l): (resource1 @ l, resource1 @ l -<lin,prf> void)
  var r = resource_new()
  var r2 = resource_new()
  prval (pf, pff) = __borrow(view@ r)
  prval (pf2, pff2) = __borrow(view@ r2)

  val () = do_something(callback,
                        llam () => let
                          val () = resource_use(r)
                          val () = resource_use(r2)
                          prval () = pff(pf)
                          prval () = pff2(pf2)
                        in () end) 
  val () = resource_free(r2)
  val () = resource_free(r)
in
  ()
end
```

In this code the actual callback is a function that takes as an argument another callback which takes no arguments and calls it.
This new callback is a closure.
The code closes over the value of r and r2 and calls resource_use on them.
As r and r2 are linear resources I can’t use an ordinary closure to close over them.
I use a ‘linear closure containing linear types’ type of closure, identified by the <lincloptr1> tag in:

```ats
viewtypedef func = () -<lincloptr1> void
```

This tells the type system that this closure can contain references to linear resources in the enclosing scope.
The other thing I have to do is use the proof system to explicitly ‘borrow’ the linear object from the enclosing scope.
This is done using this code:

```ats
extern prfun __borrow {l:addr} (pf: ! resource1 @ l): (resource1 @ l, resource1 @ l -<lin,prf> void)
...
prval (pf, pff) = __borrow(view@ r)
```

The __borrow proof function takes a view to the resource and returns a new proof variable for this view along with a proof function that we need to call to return the borrowed resource.
Inside the closure we call this proof function, passing it the new proof variable we got, to say that we have finished borrowing the resource.
This is what this code does:

```ats
prval () = pff(pf)
```

If the code doesn’t `__borrow` the resource then the type system expects the linear resources in the closure to be consumed.
In this example we don’t want to do that as we consume them later, outside of the closure, using resource_free.
Think of `__borrow` as “Dear type system, I’m borrowing this resource temporarily in another function, I’ll let you know when I’m done with it”.

When using closures memory has to be allocated to store the enclosed environment.
I discuss this in my [Closures in ATS](http://bluishcoder.co.nz/2010/06/20/closures-in-ats.html) post.
A linear closure has that memory automatically freed by the caller.
In this case however the caller is a function implemented in C so ATS can’t insert the call to free the memory.
This results in a type check failure if I don’t manually free the memory which is what the call to cloptr_free is doing in this code.
If I was calling the callback from ATS code then I wouldn’t need it and no type check error would occur.
I wrote more about linear closures around linear resource in [lin and llam with closures](http://bluishcoder.co.nz/2010/08/02/lin-and-llam-with-closures.html).

This example works and is flexible in that I can pass any objects along, captured within the closure.
Complications arise when there are objects that need to be consumed.

## Using a container dataviewtype

If I want to consume the resources I pass to the callback I can use a dataviewtype to hold the resources I pass.
The code for this is in [callback3.dats](http://bluishcoder.co.nz/ats/callback3.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback3.html)).
The changed parts of this code are:

```ats
dataviewtype container = Container of (resource1, resource1)
typedef callback = (container) -<fun1> void
extern fun do_something(f: callback, arg: container):void = "mac#do_something"

fun mycallback(c: container): void = let
  val ~Container (r,r2) = c
  val () = resource_use(r)
  val () = resource_use(r2)
  val () = resource_free(r)
  val () = resource_free(r2)
in
  ()
end

implement main() = let
  val r = resource_new()
  val r2 = resource_new()

  val c = Container(r, r2)
  val () = do_something(mycallback, c)
in
  ()
end
```

In this code the Container dataviewtype holds the two resources.
Ownership of r and r2 are passed to the Container.
The argument for the callback is Container rather than !Container to indicate that the container itself gets destroyed.
The following line does a deconstructing bind of the objects in the container to r and r2 and destroys the container (The ~ is the notation that does this):

```ats
val ~Container (r,r2) = c
```

Hopefully that code is easier to understand than the linear closure example previously.
The next approach is how to deal with passing objects where some are to be consumed and some aren’t.

## Using call-by-reference

The approach here is to create a record containing the resources I want to share.
I pass this record around using call by reference.
The equivalent of the container example above, but using call-by-reference, is in [callback4.dats](http://bluishcoder.co.nz/ats/callback4.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback4.html)).
The changed code is:

```ats
viewtypedef env (l1:addr, l2:addr) = @{r1= resource l1, r2= resource l2}
viewtypedef env = [l1,l2:agz] env(l1, l2)
typedef callback = (&env) -<fun1> void
extern fun do_something(f: callback, arg: &env):void = "mac#do_something"

fun mycallback(e: &env): void = let
  val () = resource_use(e.r1)
  val () = resource_use(e.r2)
in
  ()
end

implement main() = let
  var env: env(null, null)
  val () = env.r1 := resource_new()
  val () = env.r2 := resource_new()

  val () = do_something(mycallback, env)

  val () = resource_free(env.r1)
  val () = resource_free(env.r2)
in
  ()
end
```

The &env syntax in the argument to a function means pass that argument by reference.
In this example the linear resources are owned by the env instance and a reference to that is passed around.
`env` is a flat record (the { says it’s a record, the @ says it’s flat vs ' for boxed).
A flat record is like a struct in C.
In fact the generated C code from ATS uses a C struct allocated on the stack.

## Consuming only some resources

Extending the previous ‘call-by-reference’ example I show how to consume some of the resources but not others.
See [callback5.dats](http://bluishcoder.co.nz/ats/callback5.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback5.html)) for the full code with the changed bits below:

```ats
viewtypedef env (l1:addr) = @{r1= resource l1, r2= Option_vt (resource1)}
viewtypedef env = [l1:agz] env(l1)
typedef callback = (&env) -<fun1> void
extern fun do_something(f: callback, arg: &env):void = "mac#do_something"

fun mycallback (e: &env): void = let
  val () = resource_use(e.r1)
  val () = case+ e.r2 of
           | ~Some_vt(x) => let
                              val () = resource_use(x)
                              val () = resource_free(x)
                              val () = e.r2 := None_vt
                            in () end
           | ~None_vt () => let
                              val () = e.r2 := None_vt
                            in () end
in
 ()
end

implement main() = let
  var env: env(null)
  val () = env.r1 := resource_new()
  val () = env.r2 := Some_vt (resource_new())

  val () = do_something(mycallback, env)

  val () = resource_free(env.r1)
  val () = case+ env.r2 of 
           | ~Some_vt (x) => resource_free(x)
           | ~None_vt () => ()
in
  ()
end
```

For resources that can be consumed I use the `Option_vt` viewtype defined in the ATS prelude.
This has constructors `Some_vt` and `None_vt`.
The former holds a value and the latter means no value is contained.
The `_vt` suffix is the prelude’s naming standard to say that this is a viewtype.
There also exists an equivalent to Option for datatypes (garbage collectable objects).
We use a viewtype here since we a holding on to values that are viewtypes.

The callback uses case to check if a value is held by the record.
if it is `Some_vt` I use the the resource, free it, and assign `None_vt` back to say there is no longer a resource being held.
In the main body I do the same to free the resource if a resource is being held.
Note the use of the `~` in the patterns to consume and free the `Option_vt` resources.

## Summary

This final example most closely follows the way some C programs work, allocating a context object to hold semi-global objects.
And passing that around as needed.
What ATS buys us is the checking at compile time that these objects are correctly destroyed and not used again.

An example of this type of C program is [download.c](http://bluishcoder.co.nz/ats/download.c) (originally [based on this code](http://archives.seul.org/libevent/users/Sep-2010/msg00050.html)) which uses [libevent](http://monkey.org/~provos/libevent/) to read the contents of a URL via HTTP.
It allocates a download_context object which contains pointers to other objects, some of which are destroyed and some are supposed to be retained.
My explorations into linear resource sharing was motivated by converting this example.
