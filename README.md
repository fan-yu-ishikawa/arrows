# Arrows - High-performance Arrow and Task in Scala
----------------------------------------------------

This library provides `Arrow` and `Task` implementations in two flavors:

### `arrows-stdlib`: built on top of the Scala Future, without external dependencies. This module also provides ScalaJS artifacts.

```
libraryDependencies ++= Seq(
  "io.trane" %%% "arrows-stdlib" % "0.1.0-SNAPSHOT"
)
```

### `arrows-twitter`: built on top of the Twitter Future with the `twitter-util` dependency.

```
libraryDependencies ++= Seq(
  "io.trane" %% "arrows-twitter" % "0.1.0-SNAPSHOT"
)
```

Both implementations have similar behavior, but they mirror the interface of the underlying Future to make the migration easier.

The library is inspired by the paper [Generalizing Monads to Arrows](http://www.cse.chalmers.se/~rjmh/Papers/arrows.pdf). It introduces Arrows as a way to express computations statically. For instance, this monadic computation:

```scala
callServiceA.flatMap { r =>
  callServiceB(r)
}
````

can't be fully inspected statically. It's possible to determine that `callServiceA` will be invoked, but only after running `callServiceA` it's possible to determine that `callServiceB` will be invoked since there's a data dependency.

Arrows are functions that can be composed and reused for multiple executions:

```scala
val myArrow: Arrow[Int, Int] = callServiceA.andThen(callServiceB)
val result: Future[Int] = myArrow.run(1)
```

The paper also introduces the `first` operator to deal with scenarios like branching where expressing the computation using `Arrow`s is challenging. It's an interesting technique that simulates a `flatMap` using tuples and joins/zips, but introduces considerable code complexity.

This library takes a different approach. Instead of avoiding `flatMap`s, it introduces a special case of `Arrow` called `Task`, which is a regular monad. `Task` is based on `ArrowApply` also introduced by the paper.

Users can easily express computations that are static using arrows and apply them within `flatMap` functions to produce `Task`s:

```scala
val callServiceA: Arrow[Int, Int] = ???
val callServiceB: Arrow[Int, Int] = ???
callServiceA.flatMap {
   case 0 => Task.value(0)
   case i => callServiceB(i)
}
```

It's a mixed approach: computations that can be easily expressed statically don't need `flatMap` and `Task`, which are normally most of the cases. Only the portion of the computations that require some dynamicity needs to incur the price of using `flatMap` and `Task`.

`Task` is an `Arrow` without inputs. It's declared as a type alias:

```scala
type Task[+T] = Arrow[Unit, T]
```

Additionally, it has a companion object that has methods similar to the ones provided by the `Future` companion object.

Static computations expressed using `Arrow`s have better performance since they avoid many allocations at runtime, but `Task` can also be as a standalone solution without using `Arrow` directly. It's equivalent to the `IO` and `Task` implementations provided by libraries like Monix, Scalaz 8, and Cats Effect.

The `Task` interface mirrors the methods provided by `Future`. For more details on the available operations, please see the scaladocs. 

**Note that, even though `Task` has an API very similar to `Future`, it has a different execution mechanism. `Future`s are strict and start executing once created. `Task`s only describe computations without running them, and are executed only when the user calls `run`.**