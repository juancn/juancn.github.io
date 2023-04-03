---
layout: post
title:  "Performance Invariants"
date:   2011-02-22 12:30:00 -0300
categories: post
---

**UPDATE**: A newer post on this subject [can be found here](/post/2011/02/25/performance-invariants-part-ii)

Let's start with a problem: How do you make unit tests that test for performance?

It might seem simple, but consider that:

 - Test must be stable across hardware/software configurations
 - Machine workload should not affect results (at least on normal situations)

A friend of mine (Fernando Rodriguez-Olivera if you must know) thought of the following (among many other things):
For each test run, record interesting metrics, such as:

 - specific method calls
 - number of queries executed
 - number of I/O operations
 - etc.

And after the test run, assert that these values are under a certain threshold. If they're not, fail the test.
He even implemented a proof-of-concept using [BeanShell](https://beanshell.github.io/) to record these stats to a file during the test run, and it would check the constraints after the fact.

Yesterday I was going over these ideas while preparing a presentation on code quality and something just clicked: annotate methods with performance invariants.
The concept is similar to pre/post conditions. Each annotation is basically a post condition on the method call that states which performance "promises" the method makes.

For example you should be able to do something like:

```java
@Ensure("queryCount <= 1")
public CustomerInfo loadCustomerInfo() {...}
```

Or maybe something like this:

```java
@Ensure("count(java.lang.Comparable.compareTo) < ceil(log(this.customers.size()))")
public CustomerInfo findById(String id) {...}
```

These promises are enabled only during testing since checking for them might be a bit expensive for a production system.

As this is quite recent I don't have anything working (yet), but I think it's worth exploring.
If I manage to find some time to try and build it, I'll post some updates here.