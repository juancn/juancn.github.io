---
layout: post
title:  "Performance Invariants - Part II"
date:   2011-02-25 20:04:00 -0300
categories: post
---

A few days ago I wrote [a post about performance invariants](/post/2011/02/22/performance-invariants). The basic idea behind them, is that there should be an easy way to declare performance constraints at the source code level, and that you should be able to check them every time you run your unit tests. To make a long story short, I have been a busy little bee for the last few days and managed to build [a reasonable proof-of-concept](http://github.com/juancn/performance).

Let's start with simple example:

```java
import performance.annotation.Expect;
...
class Test {
    @Expect("InputStream.read == 0")
    static void process(List<String> list) {
        //...
    }
}
```

What we're asserting here is that we want to make sure that the number of calls to methods called read defined in classes named InputStream should be exactly zero.

If we want to exclude basically all IO, we can change the expectation to:

```java

import performance.annotation.Expect;
...
class Test {
    @Expect("InputStream.read == 0 && OutputStream.write == 0")
    static void process(List<String> list) {
        //...
    }
}
```

Note that these are checked even for code that is called indirectly by the method process.

If we add an innocent looking `println`:

```java
    @Expect("InputStream.read == 0 && OutputStream.write == 0")
    static void process(List<String> list) {
        System.out.println("Hi!");
        //...
    }
```

And run it with the agent enabled by using:

```
~>java -javaagent:./performance-1.0-SNAPSHOT-jar-with-dependencies.jar \
       -Xbootclasspath/a:./performance-1.0-SNAPSHOT-jar-with-dependencies.jar Test
You should get something like the following output:
Hi!
Exception in thread "main" java.lang.AssertionError: Method 'Test.process' did not fulfil: InputStream.read == 0 && OutputStream.write == 0
         Matched: [#OutputStream.write=7, #InputStream.read=0]
         Dynamic: []
        at performance.runtime.PerformanceExpectation.validate(PerformanceExpectation.java:69)
        at performance.runtime.ThreadHelper.endExpectation(ThreadHelper.java:52)
        at performance.runtime.Helper.endExpectation(Helper.java:61)
        at Test.process(Test.java:17)
        at Test.main(Test.java:39)
```

This is witchraft, I say! ... well kind of.

Let's stop a moment and consider what's going on here. Notice the first line of the output. It contains the text "Hi!" that we printed. This happens because the check is performed after the method process finishes. In the fourth line, you can see how many times each method matched during the execution of the process method. Ignore the "Dynamic" list for just a second.

Let's try something a bit more interesting:
```java
    class Customer { /*... */}
    //...
    @Expect("Statement.executeUpdate < ${customers.size}")
    void storeCustomers(List<Customer> customers) {
        //...
    }
```

Note the `${customers.size}` in the expression, what this intuitively mean is that we want to take the size of the list as an upper bound. It's like the poor programmer's big-O notation. If we were to run this, but assuming that we execute two updates for each customer (instead of one as asserted), we would get:

```
Exception in thread "main" java.lang.AssertionError: Method 'Test.storeCustomers' did not fulfil: Statement.executeUpdate < ${customers.size}
         Matched: [#Statement.executeUpdate=50]
         Dynamic: [customers.size=25.0]
        at performance.runtime.PerformanceExpectation.validate(PerformanceExpectation.java:69)
        at performance.runtime.ThreadHelper.endExpectation(ThreadHelper.java:52)
        at performance.runtime.Helper.endExpectation(Helper.java:61)
        at Test.storeCustomers(Test.java:19)
        at Test.main(Test.java:42)
```

Check the third line, this time, the "Dynamic" list contains the length of the list. In general, expressions of the form `${a.b.c.d}` are called dynamic values. They refer to arguments, instance variables or static variables. For example:

 - `${static.CONSTANT}` refers to a variable named `CONSTANT` in the current class.
 - `${this.instance}` refers to a variable named `instance` in the current object (only valid for instance methods).
 - `${n}` refers to an argument named `n` (this only works if the class has debug information)
 - `${3}` refers to the fourth argument from the left (zero based indexing)

All dynamic values MUST yield a numeric value, otherwise a failure will be reported at runtime. Currently the library will complain if any dynamic value is null.

Although this is an early implementation, it is enough to start implementing performance invariants that can be checked every time you run your unit tests.
Enough for today, in a followup post I'll go into the internals of the agent. If you want to browse the source code or try it out, [go and grab a copy from github](http://github.com/juancn/performance).