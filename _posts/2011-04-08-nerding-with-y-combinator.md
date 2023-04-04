---
layout: post
title:  "Nerding with the Y-combinator"
date:   2011-04-08 16:51:00 -0300
categories: post
---

What follows is a pointless exercise. Hereby I present you the [Y-combinator](http://kestas.kuliukas.com/YCombinatorExplained/) in Java with generics:

```java
public class Combinators {
    interface F<A,B> {
        B apply(A x);
    }
    //Used for proper type checking
    private static interface FF<A, B> extends F<FF<A, B>, F<A, B>> {}

    //The Y-combinator
    public static <A, B> F<A, B> Y(final F<F<A, B>,F<A, B>> f) {
        return U(new FF<A, B>() {
            public F<A, B> apply(final FF<A, B> x) {
                return f.apply(new F<A, B>() {
                    public B apply(A y) {
                        return U(x).apply(y);
                    }
                });
            }
        });
    }

    //The U-combinator
    private static <A,B> F<A, B> U(FF<A, B> a) {
        return a.apply(a);
    }

    static F<F<Integer, Integer>, F<Integer, Integer>> factorialGenerator() {
        return new F<F<Integer, Integer>, F<Integer, Integer>>() {
            public F<Integer, Integer> apply(final F<Integer, Integer> fact) {
                return new F<Integer, Integer>() {
                    public Integer apply(Integer n) {
                        return n == 0 ? 1 : n * fact.apply(n-1);
                    }
                };
            }
        };
    }

    public static void main(String[] args) {
        F<Integer, Integer> fact = Y(factorialGenerator());
        System.out.println(fact.apply(6));
    }
}
```
Having the Y-combinator implemented in Java, actually serves no purpose (Java supports recursion) but it was interesting to see if it could be done with proper generics.