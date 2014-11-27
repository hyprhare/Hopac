# Inverting Event Streams is a Choice with Concurrent Jobs

Concurrent ML, on which Hopac is based, is a concurrent programming language
designed by John Reppy in late 80s and early 90s.  The primary motivation for
the design of Concurrent ML was programming interactive systems such as
graphical user interfaces.  The hypothesis was that interactive systems are
naturally concurrent and programming them in sequential languages results in
awkward program structures.  As it happened, use of concurrent languages for
programming user interfaces did not gain much traction.  Instead, one of the
recent developments in user interface programming has been the introduction of
*event stream combinators*.  A prominent example of an event stream combinator
library would be
[The Reactive Extensions aka Rx](http://msdn.microsoft.com/en-us/data/gg577609.aspx).

Event streams are a powerful abstraction and can significantly help to reduce
the so called *callback hell*.  Nevertheless, event stream combinators do not
fundamentally change the fact that control is still in the event processing
*framework* rather than in the client program.  Adhering to the *Hollywood
-principle*, the event streams call your callbacks and not the other way around.
This makes it difficult to directly match control flow to program state.
Callbacks examine and *modify mutable program state* and the resulting program
can be difficult to understand and reason about.

One of the primary motivations for using lightweight threads for user interfaces
has been that they have allowed one to invert back the inversion of control
caused by events so that the control will be in client code.  Once simple
control flow is recovered, it is possible to directly match control flow with
program state, allowing the use of simple programming techniques such as
*lexical binding*, *recursion*, and *immutable data* structures that are easy to
understand and reason about.

In this note we will show that the selective synchronous operations aka
alternatives of Hopac are powerful enough to invert back the inversion of
control caused by event stream combinators.  The discovery of this simple
programming technique was triggered when Lev Gorodinski
[asked](https://twitter.com/eulerfx/status/534014908095287296) how "to do
non-deterministic merge" for a Hopac based sequence he was working on.  This
technique differs from Lev Gorodinski's approach only in that the streams are
directly represented as selective synchronous operations that are memoized.

## About Rx style Event Stream Combinators

Before we look at how event streams can be inverted, let's take a moment to
consider what Rx is all about.  We use Rx as an example, but pretty much any
event stream combinator library would do.  There are two central interfaces in
Rx. `IObservable`[*](http://msdn.microsoft.com/en-us/library/dd990377%28v=vs.103%29.aspx)

```fsharp
type IObservable<'x> =
  abstract Subscribe: IObserver<'x> -> IDisposable
```

represents a source or stream of events and
`IObserver`[*](http://msdn.microsoft.com/en-us/library/dd783449%28v=vs.110%29.aspx)

```fsharp
type IObserver<'x> =
  abstract OnCompleted: unit -> unit
  abstract OnError: exn -> unit
  abstract OnNext: 'x -> unit
```

represents a sink or a callback attached to an `IObservable`.

The power of Rx comes from a large number of
[combinators for observables](http://msdn.microsoft.com/en-us/library/system.reactive.linq.observable%28v=vs.103%29.aspx).

Using the Rx combinators, one can merge, filter, append, map, throttle, ... and
more to produce a stream of events that our application is interested in.  This
is quite nice, because such stream combinations can be declarative without any
direct use of shared mutable state.  Ultimately one has to *subscribe* a
callback to be called on events from the combined stream.  However, this also
means that Rx calls the callback function, which must then return to Rx.  So, in
practice, the callback function must be programmed in terms of shared mutable
state.

Could we invert back the inversion of control, allowing the use of programming
techniques such as lexical binding, recursion and immutable data structures,
while also supporting powerful combinators for combining events?

## Inverting Event Streams

An intrinsic part of the education of a functional programmer is the
introduction of the concept of *lazy streams*.  There are many ways to design
such lazy streams.  Here is one:

```fsharp
type Stream<'x> =
  | Nil
  | Cons of Value: 'x * Next: Streams<'x>
and Streams<'x> = Lazy<Stream<'x>>
```

All the combinators that an event stream combinator library like Rx supports can
easily be implemented for lazy streams&mdash;except for the fact that lazy
streams have no concept of time or choice.  Fortunately, to recover a concept of
time or choice, we can simply replace the `Lazy<_>` type constructor with the
`Alt<_>`[*](http://vesakarvonen.github.io/Hopac/Hopac.html#def:type%20Hopac.Alt)
type constructor:

```fsharp
type Stream<'x> =
  | Nil
  | Cons of Value: 'x * Next: Streams<'x>
and Streams<'x> = Alt<Stream<'x>>
```

We'll refer to the above kind of streams as *choice streams*, because they
support a kind of non-deterministic choice via the `Alt<_>` type constructor.

It is straightforward to convert all lazy stream combinators to choice stream
combinators.  For example, given a suitably defined bind operation `>>=`

```fsharp
let (>>=) (xL: Lazy<'x>) (x2yL: 'x -> Lazy<'y>) : Lazy<'y> =
  lazy (x2yL (xL.Force ())).Force ()
```

for the `Lazy<_>` type constructor and a lazy `cons` helper function

```fsharp
let cons x xs = lazy Cons (x, xs)
```

we could write lazy stream `append` as

```fsharp
let rec append (ls: Streams<'x>) (rs: Streams<'x>) : Streams<'x> =
  ls >>= function Nil -> rs
                | Cons (l, ls) -> cons l (append ls rs)
```

Similarly, given a suitably defined `cons` helper function

```fsharp
let cons x xs = Job.result (Cons (x, xs))
```

we can give an implementation of `append` for choice streams:

```fsharp
let rec append (ls: Streams<'x>) (rs: Streams<'x>) : Streams<'x> =
  ls >>=? function Nil -> upcast rs
                 | Cons (l, ls) -> cons l (append ls rs)
```

However, what is really important is that time is a part of the representation
of choice streams and using combinators such as
`Alt.choose`[*](http://vesakarvonen.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.choose)
and
`<|>?`[*](http://vesakarvonen.github.io/Hopac/Hopac.html#def:val%20Hopac.Alt.Infixes.%3C|%3E?)
it is possible to construct streams that may include timed operations and make
non-deterministic choices between multiple streams.

Here is a first attempt at implementing a `merge` combinator:

```fsharp
let rec merge (ls: Streams<'x>) (rs: Streams<'x>) : Streams<'x> =
  mergeSwap ls rs <|>? mergeSwap rs ls
and mergeSwap (ls: Streams<'x>) (rs: Streams<'x>) : Streams<'x> =
  ls |>>? function
     | Nil -> upcast rs
     | Cons (l, ls) -> cons l (merge rs ls)
```

There is no nice way to write the above kind of `merge` for lazy streams.
However, there is one problem with the above implementation.  Can you spot it?

The above version of `merge` produces an *ephemeral* stream: it could produce
different results each time it is examined.  We don't want that, because it
could lead to nasty inconsistencies when the stream is consumed by multiple
clients.  So, in practice, when writing choice stream combinators, we will make
sure that streams always produce the same results and in this case we can
*memoize* the stream.

To memoize choice streams, we introduce a couple of auxiliary memoizing
combinators using the lazy
`Promise.Now.delayAsAlt`[*](http://vesakarvonen.github.io/Hopac/Hopac.html#def:val%20Hopac.Promise.Now.delayAsAlt)
combinator:

```fsharp
let memo x = Promise.Now.delayAsAlt x
let (>>=*) x f = x >>= f |> memo
let (|>>*) x f = x |>> f |> memo
let (<|>*) x y = x <|> y |> memo
```

Using the above memoizing choice combinator, `<|>*`, we can now implement a
memoized `merge` as:

```fsharp
let rec merge (ls: Streams<'x>) (rs: Streams<'x>) : Streams<'x> =
  mergeSwap ls rs <|>* mergeSwap rs ls
and mergeSwap (ls: Streams<'x>) (rs: Streams<'x>) : Streams<'x> =
  ls |>>? function
     | Nil -> upcast rs
     | Cons (l, ls) -> cons l (merge rs ls)
```

What about `append`?  If we assume that both the streams given to `append`
already always produce the same results, then the resulting stream will always
be the same.  Avoiding memoization can bring some performance benefits, but we
can also write a memoizing version of `append` as follows:

```fsharp
let rec append (ls: Streams<'x>) (rs: Streams<'x>) : Streams<'x> =
  ls >>=* function Nil -> upcast rs
                 | Cons (l, ls) -> cons l (append ls rs)
```

As can actually be seen from the above two examples already, given a choice
stream or any number of choice streams, one can process elements from such a
stream by means of a simple loop.

When multiple streams need to be processed concurrently, one can spawn a
separate lightweight thread (or job) for each such concurrent activity.

Stream producers can be written in various ways.  One way is to write a loop
that simply constructs the stream using lazy promises&mdash;just like the stream
combinators above do.  For example, a sequence can be lazily converted to choice
stream using the below `ofSeq` function:

```fsharp
let rec ofEnum (xs: IEnumerator<'x>) = memo << Job.thunk <| fun () ->
  if xs.MoveNext () then
    Cons (xs.Current, ofEnum xs)
  else
    xs.Dispose ()
    Nil

let ofSeq (xs: seq<_>) = memo << Job.delay <| fun () ->
  upcast ofEnum (xs.GetEnumerator ())
```

Another way is to represent the tail of a stream using a write once variable aka
an
`IVar<_>`[*](http://vesakarvonen.github.io/Hopac/Hopac.html#def:type%20Hopac.IVar)
and have the producer
`fill`[*](http://vesakarvonen.github.io/Hopac/Hopac.html#def:val%20Hopac.IVar.fill)
that write once variable with a new stream node (containing a new write once
variable).  This way one can convert ordinary imperative event sources to choice
streams.  For example, the following scoped `subscribingTo` function subscribes
to an `IObservable<_>` and calls a job constructor with the resulting stream:

```fsharp
let subscribingTo (xs: IObservable<'x>) (xs2yJ: Streams<'x> -> #Job<'y>) = job {
  let streams = ref (ivar ())
  use unsubscribe = xs.Subscribe {new IObserver<_> with
    override this.OnCompleted () = !streams <-= Nil |> start
    override this.OnError (e) = !streams <-=! e |> start
    override this.OnNext (value) =
      let next = ivar ()
      !streams <-= Cons (value, next) |> start
      streams := next}
  return! !streams |> xs2yJ :> Job<_>
}
```

Choice streams combinators are lazy.  Once a stream consumer stops pulling
elements from the stream and the variable referring to the stream is no longer
reachable, the stream can be garbage collected.  Once a stream producer is
garbage collected, threads waiting on the end of the associated stream can be
garbage collected.

Errors in choice streams are handled in the usual way.  The `memo` combinator we
made above uses a
`Promise`[*](http://vesakarvonen.github.io/Hopac/Hopac.html#def:type%20Hopac.Promise)
underneath.  If a choice stream producer raises an exception, it will be
captured by a promise and ultimately reraised when the promise is examined by a
choice stream consumer.

As can also be seen from the above examples, choice stream combinators look
quite familiar.  A functional programmer should not find it difficult to write
new combinators based on existing combinators or even to write new combinators
from scratch.

So, basically, with choice streams we can use functional programming to combine
event streams and we can also use functional programming to consume streams.
With Rx, we can use functional programming techniques to combine streams, but we
must then resort to imperative programming to consume streams.  To put it
another way, with choice streams, you can have your cake and eat it too.  With
Rx, you can have your cake, and then let Rx feed it to you.

## Further

There is an experimental library based on the above technique.  See:

* [Streams.fsi](https://github.com/VesaKarvonen/Hopac/blob/master/Libs/Hopac.Experimental/Streams.fsi)
* [Streams.fs](https://github.com/VesaKarvonen/Hopac/blob/master/Libs/Hopac.Experimental/Streams.fs)

## Related work

Tomas Petricek has previously introduced the concept of
[F# asynchronous sequences](http://tomasp.net/blog/async-sequences.aspx/).
Building on top of the asynchronous workflows or `async` of F# Petricek's
streams fail to capture the full power of event streams.  The F# `async`
mechanism does not directly provide for a non-deterministic choice operator or
memoization (promises or futures).  In particular, asynchronous sequences do not
provide a `merge` combinator.  Hopac directly provides both a choice operator
and promises, which makes it straightforward to extend the power of streams.