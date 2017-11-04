---
layout: default
title: Waiting
---

Locks and condition variables go together. `synchronized` (when used
properly) makes sure that a group of reads and writes isn't interleaved
with the actions of another thread. `notifyAll` and `wait` let threads
wait for a condition to become true.

`retry` inside `atomic` fills the same role as `wait` inside
`synchronized`, but `retry` is safer and more powerful. With
transactional waiting the STM automatically identifies the `Ref` changes
that should lead to wakeup, so you don't have to call `notifyAll`. This
means that there is no chance of lost wakeup. It also means that you can
wait for any condition to become true, not just those that were
identified ahead of time.

Search with backtracking {#search}
------------------------

To explain how `retry` works (and why it's called `retry`), it is
helpful to think of optimistic concurrency control as a search with
backtracking. As an atomic block executes it is searching forward. At
each step, if the STM discovers that the current search can't proceed
then it backs up (rolls back) and tries again. In this analogy the
transaction is attempting to find a path of `Ref` reads that avoid the
`Ref` writes performed by other threads.

To make this more concrete, consider the following two atomic blocks
that each access a pair of `Ref`-s. The `sum` method returns their sum,
and the `transfer` method increments one and decrements the other.
`transfer` should not affect the value returned from `sum`.

{% highlight scala %}
val (x, y) = (Ref(10), Ref(0))

def sum = atomic { implicit txn => 
  val a = x()
  val b = y()
  a + b
}

def transfer(n: Int) {
  atomic { implicit txn =>
    x -= n
    y += n
  }
}
{% endhighlight %}

If `sum` and `transfer(2)` are run simultaneously and `sum` is unlucky,
it might look like this

{% highlight java %}
// sum                                // transfer(2)
atomic                                atomic
|  begin txn attempt                  |  begin txn attempt
|  |  read x -> 10                    |  |  read x -> 10
|  |     :                            |  |  write x <- 8
|  |                                  |  |  read y -> 0
|  |     :                            |  |  write y <- 2
|  |                                  |  commit
|  |  read y -> x read is invalid     +-> ()
|  roll back
|  begin txn attempt
|  |  read x -> 8
|  |  read y -> 2
|  commit
+-> 10
{% endhighlight %}

The first time that `sum` tries to read `y`, the STM detects that the
value previously read from `x` is no longer correct. This means that the
transaction attempt (search) has reached a dead end and the system
should roll back (backtrack) and try again. On the second attempt to
compute the sum both of the reads are consistent and so the transaction
can commit (the search is successful).

Retry
-----

When an atomic block calls `retry` it is signalling that the current
search is a dead end, even though all of the reads and writes are
consistent. The STM will backtrack and then try again. If some of the
`Ref`-s read by the transaction have changed before the new attempt,
then the atomic block may take a different path that avoids the call to
`retry`. *The condition that `retry` waits for is implicitly embedded in
the control path of the atomic block.*

For example, to wait until `x` is greater than 10, just write
`if (x() <= 10) retry`. The code that updates `x` doesn't need to know
ahead of time what conditions are interesting to its users, which
decreases coupling. The awaited condition might include inputs from
several `Ref`-s. For example, to wait until `x`, `y` or `z` is non-zero
write `if (x() == 0 && y() == 0 && z() == 0) retry`.

This sounds like busy-waiting, so what about efficiency? It would be
very wasteful to roll back and reexecute continuously. Fortunately,
there is no point in trying again until one of the accessed `Ref`-s has
changed, and the STM knows which `Ref`-s were accessed. Internally
`retry` is implemented using blocking constructs, so there is no
busy-waiting.

Note that `retry` has a return type of `Nothing`, which indicates that
it does not return normally.

Alternatives
------------

While `retry` lets you tell the STM about a dead end in the backtracking
search, `orAtomic`[^1] lets you give multiple search paths. `orAtomic`
lets you chain atomic blocks that are alternative solutions to the
search. If the first alternative calls `retry` then the second will be
tried, if the second calls `retry` then the third will be tried, and so
on. (Alternatives are only considered after an explicit dead end using
`retry`, not when the transaction's reads are inconsistent.) You can
chain alternatives directly using `orAtomic`, or you can pass a
collection of atomic blocks to `atomic.oneOf`.

The following code waits until one of `x`, `y` or `z` is non-zero,
subtracts one from the first of the list that is non-zero, and records
what was done in the `String` `msg`. The second block will only be
attempted if the first block calls `retry`, and the third block will
only be attempted if the second block calls `retry`. If all of the
blocks call `retry` then the thread is blocked until one of the `Ref`-s
is changed, and then the chain of alternatives will be retried.

{% highlight scala %}
val msg = atomic { implicit txn =>
  if (x() == 0)
    retry
  x -= 1
  "took one from x"
} orAtomic { implicit txn =>
  if (y() == 0)
    retry
  y -= 1
  "took one from y"
} orAtomic { implicit txn =>
  if (z() == 0)
    retry
  z -= 1
  "took one from z"
}
{% endhighlight %}

Timeouts
--------

**Composable timeouts are an innovative feature of ScalaSTM, so we're
especially interested in your constructive criticism.**

There are several reasons why you might want `retry` to time out.
Perhaps error logging or handling code should be invoked; perhaps the
waiting thread should shut down if it doesn't receive any work for a
while; or perhaps you are using the STM to implement a higher-level
interface that includes timeouts in its specification.

There are two ways to limit the amount of time that `retry` will wait.
If you would like to set a policy for all calls to `retry`, you can use
a modififed `TxnExecutor` that will cause timed-out retries to throw an
`InterruptedException`. If timeout is not an exceptional behavior you
can supply a timeout when waiting by using the `retryFor` operation,
which falls through if the timeout has expired.

### Using a custom TxnExecutor

The `atomic` keyword defined in the `scala.concurrent.stm` package
object is actually just a method that returns a `TxnExecutor` instance.
The `apply(block)` method on the returned instance is actually
responsible for actually executing the atomic block. `TxnExecutor` also
has a `withRetryTimeout` method that returns a new executor that will
apply the retry timeout to all of the atomic block it executes. You can
apply the retry timeout directly when you call atomic, you can capture
and reuse the new executor with a new name, or you can use
`TxnExecutor.transformDefault` to change the system-wide default
executor returned by `atomic`.

{% highlight scala %}
atomic.withRetryTimeout(1000) { implicit txn =>
  // any retries in this atomic block will wait for at most 1000 milliseconds
}

val myAtomic = atomic.withRetryTimeout(1, TimeUnit.SECONDS)
myAtomic { implicit txn =>
  // this atomic block has a timeout of 1 seconds
}
myAtomic { ... }

TxnExecutor.transformDefault( _.withRetryTimeout(1000) )
atomic { implicit txn =>
  // all atomic blocks now default to a 1 second timeout
}
{% endhighlight %}

### Using retryFor

If timeouts are part of the normal behavior, it may be convenient to
handle the timeout without an exception. `retryFor(timeout)` acts like
`retry` if the timeout has not yet expired, otherwise it returns
immediately. Note that `retryFor` and `retry` have different return
types (`Unit` and `Nothing`, respectively), because `retryFor` can
return while `retry` always triggers rollback.

Assuming that `pool` is a transactional pool, the following code waits
up to 100 milliseconds for a pooled instance to be available, after
which the pool size is increased.

{% highlight scala %}
val instance = atomic { implicit txn =>
  if (!pool.hasAvailable) {
    retryFor(100)
    pool.grow()
  }
  pool.take()
}
{% endhighlight %}

Waiting for Views {#await}
-----------------

If the condition you're waiting only involves a single `Ref` you can
also block using `Ref.View.await(pred)`. This method blocks until the
predicate becomes true, then returns.
`Ref.View.tryAwait(duration)(pred)` returns true if the predicate became
true within the specified duration, otherwise it returns false.

`Ref` `await` and `tryAwait` are part of 0.3. In version 0.2 the
functionality of `await` was called `retryUntil`, and there was no
equivalent to `tryAwait`.

[^1]: ScalaSTM's `orAtomic` is referred to as `orElse` in the other STMs
    that support it. We changed the name to avoid confusion with
    `Option.orElse`.