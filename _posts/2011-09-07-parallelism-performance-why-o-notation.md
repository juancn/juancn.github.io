---
layout: post
title:  "Parallelism & Performance: Why O-notation matters"
date:   2011-09-07 18:28:00 -0300
categories: post
---

> **UPDATE:** There was a bug in the way the suffix map was built in the original post that [lolplz](http://www.reddit.com/user/lolplz) found. I updated this post with that bug fixed, however, the content of the post has not changed significantly.

Aleksandar Prokopec presented a [very interesting talk about Scala Parallel Collections](http://days2011.scala-lang.org/node/138/272) in [Scala Days 2011](http://days2011.scala-lang.org/). I saw it today and gave me some food for thought,

He opens his talk with an intriguing code snippet:
```scala
for {
  s <- surnames
  n <- names
  if s endsWith n
} yield (n , s)
```
What this code does, given what presumably are two sequences of strings, it builds all pairs (name, surname) where the surname ends with the name.
The algorithm used is very brute force, it's order is roughly O(N^2) (it's basically a [Cartesian Product](http://en.wikipedia.org/wiki/Cartesian_product) being filtered). He then goes on to show that by just using parallel collections:
```scala
for {
  s <- surnames.par
  n <- names.par
  if s endsWith n
} yield (n , s)
```
He can leverage all the cores and reduce the runtime by two or four, depending on the number of cores available in the machine where it's running. For example, the non-parallel version runs in **1040ms**, the parallel one runs in **575ms** with two cores, and in **305ms** with four. Which is indeed very impressive for such a minor change.

What concerns me is that, no matter how many cores you add, the problem as presented is still O(N^2). It is true that many useful problems can only be speeded up by throwing more hardware at it, but most of the times, using the right data representation can yield even bigger gains.

If we use this problem as an example, we can build a slightly more complex implementation, but that hopefully is a lot faster. The approach I'm taking is that of building a suffix map for surnames. [There are more efficient data structures (memory wise)](http://en.wikipedia.org/wiki/Suffix_tree) to do this, but for simplicity I'll use a Map:

```scala
val suffixMap = collection.mutable.Map[String, List[String]]().withDefaultValue(Nil)
for (s <- surnames; i <- 0 until s.length; suffix = s.substring(i)) 
    suffixMap(suffix) = s :: suffixMap(suffix)
```
Having built the prefix map, we can naively rewrite the loop to use it (instead of the Cartesian Product):
```scala
for {
  n <- names
  s <- suffixMap(n)
} yield (n , s)
```
In theory, this loop is roughly O(N) (assuming the map is a HashMap), since it now does a constant number of operations on each name it processes, rather than processing all the names for each surname (I'm ignoring the fact that the map returns lists).

> **Note:** The algorithmic order does not change much if we take into account the suffix map construction. Let's assume that we have S surnames and N names, the order of building the suffix map is O(S) and the order building the pairs is O(N), the total order of the algorithm is O(N+S). If we assume that N is approximately equal to S, then the order is O(N+N) which is the same than O(2N), which can be simplified to O(N).

So, let's see if this holds up in real life. For this I wrote the following scala script that runs a few passes of several different implementations:

```scala
#!/bin/sh
exec scala -deprecation -savecompiled "$0" "$@"
!#
def benchmark[T](name: String)(body: =>T) {
    val start = System.currentTimeMillis()
    val result = body
    val end = System.currentTimeMillis()
    println(name + ": " + (end - start))
    result
}

val surnames = (1 to 10000).map("Name" + _)
val names    = (1 to 10000).map("Name" + _)

val suffixMap = collection.mutable.Map[String, List[String]]().withDefaultValue(Nil)
for (s <- surnames; i <- 0 until s.length; suffix = s.substring(i)) 
    suffixMap(suffix) = s :: suffixMap(suffix)

for( i <- 1 to 5 ) {
    println("Run #" + i)
    benchmark("Brute force") {
        for {
            s <- surnames
            n <- names
            if s endsWith n
        } yield (n , s)
    }

    benchmark("Parallel") {
        for {
            s <- surnames.par
            n <- names.par
            if s endsWith n
        } yield (n , s)
    }

    benchmark("Smart") {
        val suffixMap = collection.mutable.Map[String, List[String]]().withDefaultValue(Nil)
        for (s <- surnames; i <- 0 until s.length; suffix = s.substring(i)) 
            suffixMap(suffix) = s :: suffixMap(suffix)
        for {
            n <- names
            s <- suffixMap(n)
        } yield (n , s)
    }

    benchmark("Smart (amortized)") {
        for {
            n <- names
            s <- suffixMap(n)
        } yield (n , s)
    }
}
```
There are four implementations:

 - Brute Force: the original implementation
 - Parallel: same as before, but using parallel collections.
 - Smart: Using the prefix map (and measuring the map construction)
 - Smart (amortized): same as before, but with the prefix map cost amortized.

# Benchmark Results

Running this script in a four core machine, I get the following results:
```
Run #1
Brute force: 2158
Parallel: 1355
Smart: 153
Smart (amortized): 27
Run #2
Brute force: 1985
Parallel: 899
Smart: 82
Smart (amortized): 7
Run #3
Brute force: 1947
Parallel: 716
Smart: 69
Smart (amortized): 5
Run #4
Brute force: 1932
Parallel: 714
Smart: 67
Smart (amortized): 6
Run #5
Brute force: 1933
Parallel: 713
Smart: 68
Smart (amortized): 5
```
As expected, the parallel version runs 3.5 times as fast as the naive one, but the implementation using the "Smart" approach runs more than **30 times faster** than the naive one. If we were able to amortize the cost of building the suffix map, the speed-up is even more staggering, at a whooping 380 times faster (although this is not always possible)!

What we can conclude from this, paraphrasing Fred Brooks[^1], is that regarding performance there is [no silver bullet](http://en.wikipedia.org/wiki/No_Silver_Bullet). The basics matter, maybe now they even matter more than ever.

Analyzing algorithmic order, in its most basic form (making huge approximations) is a very practical tool to solving hard performance problems. Still, the simple approach used here to optimize the problem is also parallelizable, which for a large enough problem, it might gain some speedup using parallel collections.

Don't get me wrong, I love Scala parallel collections and parallel algorithms are increasingly important, but they are by no means a magical solution that can be used for any problem (and I think Aleksandar is with me in this one).

# Footnotes

[^1]: misquoting is probably more accurate.