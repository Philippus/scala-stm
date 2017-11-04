---
layout: default
title: Quick Start
---

As a simple example, we'll build a doubly-linked list that can be safely
used by multiple threads or actors. We'll teach our list how to be a
blocking queue, and then we'll add the ability to select the next
available element from several queues.

Install
-------

If you use `sbt`, add the following dependency to your project
`build.sbt` and run `sbt update`. Maven2 configuration and manual
download links are [here](releases.html).

{% highlight scala %}
libraryDependencies += ("org.scala-stm" %% "scala-stm" % "0.7")
{% endhighlight %}

Use Ref for shared variables {#ref}
----------------------------

In our mutable linked list, we need thread-safety for each node's next
and prev pointers. In general, if it is possible that one thread might
write a variable at the same time that another thread is accessing it
(reading or writing), then the STM needs to be involved by using a
`Ref`. `Ref` is a single mutable cell; there are also transactional
collections (e.g. `TMap` and `TSet`) that are replacements for
collections from `scala.collection.mutable`.

{% highlight scala %}
import scala.concurrent.stm._

class ConcurrentIntList {
  private class Node(val elem: Int, prev0: Node, next0: Node) {
    val isHeader = prev0 == null
    val prev = Ref(if (isHeader) this else prev0)
    val next = Ref(if (isHeader) this else next0)
  }

  private val header = new Node(-1, null, null)
{% endhighlight %}

To make the code simpler we make the list circular, with an extra header
node. On creation, the header's next and prev point to itself. The next
and prev pointers are always non-null.

Wrap your code in atomic {#atomic}
------------------------

If `x` is a `Ref`, then `x()` gets the value stored in `x`, and
`x() = v` sets it to the value `v`.

`Ref`-s can only be read and written inside an atomic block. This is
checked at compile time by requiring that an implicit `InTxn` value be
available. Atomic blocks are functions that take an `InTxn` parameter,
so this requirement can be satisfied by marking the parameter as
implicit.

{% highlight scala %}
  def addLast(elem: Int) {
    atomic { implicit txn =>
      val p = header.prev()
      val newNode = new Node(elem, p, header)
      p.next() = newNode
      header.prev() = newNode
    }
  }
{% endhighlight %}

Compose atomic operations {#compose}
-------------------------

Atomic blocks nest, so you can build compound operations from simple
ones.

{% highlight scala %}
  def addLast(e1: Int, e2: Int, elems: Int*) {
    atomic { implicit txn =>
      addLast(e1)
      addLast(e2)
      elems foreach { addLast(_) }
    }
  }
{% endhighlight %}

Optimize single-operation transactions {#single}
--------------------------------------

`Ref.single` returns an instance of `Ref.View`, which acts just like the
original `Ref` except that it can also be accessed outside an atomic
block. Each method on `Ref.View` acts like a single-operation
transaction, hence the name. `Ref.View` provides several methods that
perform both a read and a write, such as `swap`, `compareAndSet` and
`transform`. If an atomic block only accesses a single `Ref`, it might
be more concise and more efficient to use a `Ref.View`.

{% highlight scala %}
  //def isEmpty = atomic { implicit t => header.next() == header }
  def isEmpty = header.next.single() == header
{% endhighlight %}

Wait for conditions to change {#retry}
-----------------------------

Use the `retry` keyword when an atomic block can't complete on its
current input state. Calling `retry` inside an atomic block will cause
it to roll back, wait for one of its inputs to change, and then retry
execution. This is roughly analogous to a call to `wait` where ScalaSTM
automatically generates the matching `notifyAll`. As part of its
implementation of optimistic concurrency the STM keeps track of an
atomic block's *read set*, the set of `Ref`-s that have been read during
the transaction. This means that the STM can efficiently block the
current thread until another thread has written to an element of its
read set, at which time the atomic block can be retried. This makes it
trivial to wait for complex conditions, and eliminates lost wakeups.

To demonstrate, we'll add a function to our list that waits until the
list is non-empty, then removes and returns the first element.

{% highlight scala %}
  def removeFirst(): Int = atomic { implicit txn =>
    val n = header.next()
    if (n == header)
      retry
    val nn = n.next()
    header.next() = nn
    nn.prev() = header
    n.elem
  }
{% endhighlight %}

Wait for multiple events {#alternatives}
------------------------

Another way to proceed after an atomic block ends in `retry` is to
provide an alternative. You can chain atomic blocks using `orAtomic`;
the lower alternatives will be tried if the upper ones call `retry`.
This lets you compose functions that block using `retry`, or to convert
to and from blocking behavior.

For example, we can use the blocking version of `removeFirst` to
construct one that returns an `Option`, by providing an alternative.

{% highlight scala %}
  def maybeRemoveFirst(): Option[Int] = {
    atomic { implicit txn =>
      Some(removeFirst())
    } orAtomic { implicit txn =>
      None
    }
  }
{% endhighlight %}

It is also easy to switch from a function that returns a failure code to
one that blocks. The following `select` method waits until one of its
inputs is non-empty, then removes an element from that list.

{% highlight scala %}
object ConcurrentIntList {
  def select(stacks: ConcurrentIntList*): (ConcurrentIntList, Int) = {
    atomic { implicit txn =>
      for (s <- stacks) {
        s.maybeRemoveFirst() match {
          case Some(e) => return (s -> e)
          case None => _
        }
      }
      retry
    }
  }
{% endhighlight %}

Be careful about rollback {#rollback}
-------------------------

ScalaSTM might need to try an atomic block more than once before
optimistic concurrency can succeed. Any call into the STM might
potentially discover the failure and trigger the rollback and retry.
Local non-`Ref` variables that have a lifetime longer than the atomic
block won't be rolled back, and so they should be avoided. Local
variables used only inside or only outside the atomic block are fine,
though.

Below, `badToString` is incorrect because it uses a mutable
`StringBuilder` both outside and inside its atomic block. The return
value will definitely mention all of the elements of the list, but it
might include some of them two or more times. `toString` is correct
because it uses a new `StringBuilder` for each atomic attempt.

{% highlight scala %}
  def badToString: String = {
    val buf = new StringBuilder("ConcurrentIntList(")
    atomic { implicit txn =>
      var n = header.next()
      while (n != header) {
        buf ++= n.elem.toString
        n = n.next()
        if (n != header) buf ++= ","
      }
    }
    buf ++= ")" toString
  }

  override def toString: String = {
    atomic { implicit txn =>
      val buf = new StringBuilder("ConcurrentIntList(")
      var n = header.next()
      while (n != header) {
        buf ++= n.elem.toString
        n = n.next()
        if (n != header) buf ++= ","
      }
      buf ++= ")" toString
    }
  }
{% endhighlight %}

Look at the source {#source}
------------------

This list example is part of the source on GitHub:
[ConcurrentIntList.scala](https://github.com/nbronson/scala-stm/blob/master/src/test/scala/scala/concurrent/stm/examples/ConcurrentIntList.scala)