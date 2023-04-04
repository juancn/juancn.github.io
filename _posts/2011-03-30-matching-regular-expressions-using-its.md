---
layout: post
title:  "Matching Regular Expressions using its Derivatives"
date:   2011-03-30 20:42:00 -0300
categories: post
---

## Introduction

Regular expressions are expressions that describe a set of strings over a particular alphabet. We will begin with a crash course on simple regular expressions. You can assume that we're talking about text and characters but in fact this can be generalized to any (finite) alphabet.

The definition of regular expressions is quite simple[^1], there are three basic (i.e. terminal) regular expressions:
 - The null expression (denoted as: ∅) which never matches anything
 - The empty expression, that only matches the empty string (I will use ε to represent this expression since it's customary[^2])
 - The character literal expression (usually called 'c'), that matches a single character

These three basic building blocks can be combined using some operators to form more complex expressions:
 - sequencing of regular expressions matches two regular expressions in sequence
 - alternation matches one of the two sub-expressions (usually represented by a '\|' symbol)
 - the repetition operator (aka. Kleene's star) matches zero or more repetitions of the specified subexpression

Some examples will make this clearer:
 - The expression 'a' will only match the character 'a'. Similarly 'b' will only match 'b'. If we combine them by sequencing 'ab' will match 'ab'.
 - The expression 'a\|b' will match either 'a' or 'b'.
 - If we combine sequencing with alternation as in '(a\|b)(a\|b)' (the parenthesis are clarifying), it will match: 'aa', 'ab', 'ba' or 'bb'.
 - Kleene's star as mentioned before matches zero or more of the preceding subexpression. So the expression 'a*' will match: '', 'a', 'aa', 'aaa', 'aaaa', ...
 - We can do more complex combinations, such as 'ab*(c\|ε)' that will match things like: 'a', 'ab', 'ac', 'abc', 'abb', 'abbc', ... that is any string starting with an 'a' followed by zero or more 'b''s and optionally ending in a 'c'.

Typical implementations of regular expression matchers convert the regular expression to an NFA or a DFA (which are a kind of [finite state machine](http://en.wikipedia.org/wiki/Finite_state_machine)).

Anyway, a few weeks ago I ran into [a post about using the derivative of a regular expression for matching](http://matt.might.net/articles/implementation-of-regular-expression-matching-in-scheme-with-derivatives/).

It is a quite intriguing concept and worth exploring. The original post gives an implementation in Scheme[^3]. But leaves out some details that make it a bit tricky to implement. I'll try to walk you through the concept, up to a working implementation in Java.

## Derivative of a Regular Expression

So, first question: What's the derivative of a regular expression?

> The derivative of a regular expression with respect to a character 'c' computes a new regular expression that matches what the original expression would match, assuming it had just matched the character 'c'.

As usual, some examples will (hopefully) help clarify things:
 - The expression 'foo' derived with respect to 'f' yields the expression: 'oo' (which is what's left to match).
 - The expression 'ab\|ba' derived with respect to 'a', yields the expression: 'b'
    Similarly, the expression 'ab\|ba' derived with respect to 'b', yields the expression: 'a'
 - The expression '(ab\|ba)*' derived with respect to 'a', yields the expression: 'b(ab\|ba)*'

As we explore this notion, we will work a RegEx class. The skeleton of this class looks like this:
```java
public abstract class RegEx {
    public abstract RegEx derive(char c);
    public abstract RegEx simplify();
//...
    public static final RegEx unmatchable = new RegEx() { /* ... */ }
    public static final RegEx empty = new RegEx() { /* ... */ }
}
```

It includes constants for the unmatchable (null) and empty expressions, and a derive and simplify methods wich we will cover in detail (but not just now).

Before we go in detail about the rules of regular expression derivation, let's take a small -but necessary- detour and cover some details that will help us get a working implementation.

The formalization of the derivative of a regular expression depends on a set of simplifying constructors that are necessary for a correct implementation. These will be defined a bit more formally and we will build the skeleton of its implementation at this point.

Let's begin with the sequencing operation, we define the following constructor (ignore spaces):

1. `seq( ∅, _ ) = ∅`
2. `seq( _, ∅ ) = ∅`
3. `seq( ε, r2 ) = r2`
4. `seq( r1, ε ) = r1`
5. `seq( r1, r2 ) = r1 r2`

The first two definitions state that if you have a sequence with the null expression (∅, which is unmatchable) and any other expression, it's the same than having the null expression (i.e. it will not match anything).

The third and fourth definitions state that if you have a sequence of the empty expression (ε, matches only the empty string) and any other expression, is the same than just having the other expression (the empty expression is the identity with respect to the sequence operator).

The fifth and last definition just builds a regular sequence.

With this, we can draft a first implementation of a sequence constructor (in the gang-of-four's parlance it's a factory method):
```java
    public RegEx seq(final RegEx r2) {
        final RegEx r1 = this;
        if(r1 == unmatchable || r2 == unmatchable) return unmatchable;
        if(r1 == empty) return r2;
        if(r2 == empty) return r1;
        return new RegEx() {
             // ....
        };
    }
```

I'm leaving out the details of the RegEx for the time being, we will come back to them soon enough.

The alternation operator also has simplifying constructor that is analogous to the sequence operator:

```
alt( ε, _  ) = ε
alt(  _, ε ) = ε
alt( ∅, r2 ) = r2
alt( r1, ∅ ) = r1
alt( r1, r2 ) = r1 | r2
```

If you look closely, the first two definitions are rather odd. They basically reduce an alternation with the empty expression to the empty expression (ε). This is because the simplifying constructors are used as part of a simplification function that reduces a regular expression to the empty expression if it matches the empty expression. We'll see how this works with the rest of it in a while.

The third and fourth definitions are fairly logical, an alternation with an unmatchable expression is the same than the alternative (the unmatchable expression is the identity with respect to the alternation operator).

The last one is the constructor.

Taking these details into account, we can build two factory methods, one internal and one external:

```java
    private RegEx alt0(final RegEx r2) {
        final RegEx r1 = this;
        if(r1 == empty || r2 == empty) return empty;
        return alt(r2);
    }

    public RegEx alt(final RegEx r2) {
        final RegEx r1 = this;
        if(r1 == unmatchable) return r2;
        if(r2 == unmatchable) return r1;
        return new RegEx() {
             //.....
        };
    }
```
The internal one alt0 includes the first two simplification rules, the public one is user-facing. That is, it has to let you build something like: 'ab*(c|ε)'.

Finally, the repetition operator (Kleene's star) has the following simplification rules:

```
rep( ∅ ) = ε
rep( ε ) = ε
rep( re ) = re*
```

The first definition states that a repetition of the unmatchable expression, matches at least the empty string.

The second definition states that a repetition of the empty expression is the same than matching the empty expression.

And as usual, the last one is the constructor for all other cases.

A skeleton for the rep constructor is rather simple:

```java
    public RegEx rep() {
        final RegEx re = this;
        if(re == unmatchable || re == empty) return empty;
        return new RegEx() {
             // ....
        };
    }
```
## Simplify & Derive

As hinted earlier on, derivation is based on a simplification function. This simplification function reduces a regular expression to the empty regular expression (ε epsilon) if it matches the empty string or the unmatchable expression (∅) if it does not.
The simplification function is defined as follows:

```
s(∅) = ∅
s(ε) = ε
s(c) = ∅
s(re1 re2) = seq(s(re1), s(re2))
s(re1 | re2) = alt(s(re1), s(re2))
s(re*) = ε
```

Note that this function depends on the simplifying constructors we described earlier on.
Suppose that we want to check if the expression 'ab\*(c\|ε)' matches the empty expression, if we do all the substitutions:


1. `seq(s(ab*),s(c|ε))`
2. `seq(s(seq(s(a), s(b*))),s(alt(s(c), s(ε))))`
3. `seq(s(seq(∅, s(ε))),s(alt(∅, ε)))`
4. `seq(s(seq(∅, ε)),s(ε))`
5. `seq(s(∅),ε)`
6. `seq(∅,ε)`
7. `∅`

We get the null/unmatchable expression as a result. This means that the expression 'ab*(c\|ε)' does not match the empty string.
If on the other hand we apply the reduction on **'a\*\|b'**:


1. `alt(s(a*), s(b))`
2. `alt(ε, ∅)`
3. `ε`

We get the empty expression, hence the regular expression 'a\*\|b' will match the empty string.
The derivation function given a regular expression and a character 'x' derives a new regular expression as if having matched 'x'.

Derivation is defined by the following set of rules:
```
D( ∅, _ ) = ∅
D( ε, _ ) = ∅
D( c, x ) = if c == x then ε else ∅
D(re1 re2, x) = alt(
                     seq( s(re1) , D(re2, x) ),
                     seq( D(re1, x), re2 )
                )
D(re1 | re2, x)  = alt( D(re1, x) , D(re2, x) )
D(re*, x)        = seq( D(re, x)  , rep(re) )
```

The first two definitions define the derivative of the unmatchable and empty expressions regarding any character, wich yields the unmatchable expression.
The third definition states that if a character matcher (for example 'a') is derived with respect to the same character yields the empty expression otherwise yields the unmatchable expression.

The fourth rule is a bit more involved, but trust me, it works.

The fifth rule states that the derivative of an alternation is the alternation of the derivatives (suitably simplified).

And the last one, describes how to derive a repetition. For example D('(ba)*', 'b') yields 'a(ba)*'.

We now have enough information to implement the simplify and derive.

## Matching

If you haven't figured it out by now, matching works by walking the string we're checking character by character and successively deriving the regular expression until we either run out of characters, at wich point we simplify the derived expression and see if it matches the empty string. Or we end up getting the unmatchable expression, at wich point it is impossible that the rest of the string will match.
A iterative implementation of a match method is as follows:

```java
    public boolean matches(final String text) {
        RegEx d = this;
        String s = text;
        //The 'unmatchable' test is not strictly necessary, but avoids unnecessary derivations
        while(!s.isEmpty() && d != unmatchable) {
            d = d.derive(s.charAt(0));
            s = s.substring(1);
        }
        return d.simplify() == empty;
    }
```
If we match **'ab\*(c\|ε)'** against the text "abbc", we get the following derivatives:

1. `D(re, a) = ab*(c|ε) , rest: "bbc"`
2. `D(re, b) = b*(c|ε) , rest: "bc"`
3. `D(re, b) = b*(c|ε) , rest: "c"`
4. `D(re, c) = b*(c|ε) , rest: ""`

And if we simplify the last derivative we get the empty expression, therefore we have a match.
One interesting fact of this matching strategy is that it is fairly easy to implement a non-blocking matcher. That is, doing incremental matching as we receive characters.

## Implementation

The following is the complete class with all methods implemented. I provide a basic implementation of the toString method (which is nice for debugging), and a helper text method which is a shortcut to build an expression for a sequence of characters. This class is fairly easy to modify to match over a different alphabet, such as arbitrary objects and Iterables instead of Strings (it can be easily generified).

```java
public abstract class RegEx {
    public abstract RegEx derive(char c);
    public abstract RegEx simplify();

    public RegEx seq(final RegEx r2) {
        final RegEx r1 = this;
        if(r1 == unmatchable || r2 == unmatchable) return unmatchable;
        if(r1 == empty) return r2;
        if(r2 == empty) return r1;
        return new RegEx() {
            @Override
            public RegEx derive(char c) {
                return r1.simplify().seq(r2.derive(c))
                        .alt0(r1.derive(c).seq(r2));
            }

            @Override
            public RegEx simplify() {
                return r1.simplify().seq(r2.simplify());
            }

            @Override
            public String toString() {
                return r1 + "" + r2;
            }
        };
    }

    private RegEx alt0(final RegEx r2) {
        final RegEx r1 = this;
        if(r1 == empty || r2 == empty) return empty;
        return alt(r2);
    }

    public RegEx alt(final RegEx r2) {
        final RegEx r1 = this;
        if(r1 == unmatchable) return r2;
        if(r2 == unmatchable) return r1;
        return new RegEx() {
            @Override
            public RegEx derive(char c) {
                return r1.derive(c).alt0(r2.derive(c));
            }

            @Override
            public RegEx simplify() {
                return r1.simplify().alt0(r2.simplify());
            }

            @Override
            public String toString() {
                return "(" + r1 + "|" + r2 + ")";
            }
        };
    }

    public RegEx rep() {
        final RegEx re = this;
        if(re == unmatchable || re == empty) return empty;
        return new RegEx() {
            @Override
            public RegEx derive(char c) {
                return re.derive(c).seq(re.rep());
            }

            @Override
            public RegEx simplify() {
                return empty;
            }

            @Override
            public String toString() {
                String s = re.toString();
                return s.startsWith("(")
                        ? s + "*"
                        :"(" + s + ")*";
            }

        };
    }
    
    public static RegEx character(final char exp) {
        return new RegEx() {
            @Override
            public RegEx derive(char c) {
                return exp == c?empty:unmatchable;
            }

            @Override
            public RegEx simplify() {
                return unmatchable;
            }

            @Override
            public String toString() {
                return ""+ exp;
            }
        };
    }

    public static RegEx text(final String text) {
        RegEx result;
        if(text.isEmpty()) {
            result = empty;
        } else {
            result = character(text.charAt(0));
            for (int i = 1; i < text.length(); i++) {
                result = result.seq(character(text.charAt(i)));
            }
        }
        return result;
    }


    public boolean matches(final String text) {
        RegEx d = this;
        String s = text;
        //The 'unmatchable' test is not strictly necessary, but avoids unnecessary derivations
        while(!s.isEmpty() && d != unmatchable) {
            d = d.derive(s.charAt(0));
            s = s.substring(1);
        }
        return d.simplify() == empty;
    }

    private static class ConstantRegEx extends RegEx {
        private final String name;
        ConstantRegEx(String name) {
            this.name = name;
        }

        @Override
        public RegEx derive(char c) {
            return unmatchable;
        }

        @Override
        public RegEx simplify() {
            return this;
        }

        @Override
        public String toString() {
            return name;
        }
    }

    public static final RegEx unmatchable = new ConstantRegEx("<null>");
    public static final RegEx empty = new ConstantRegEx("<empty>");

    public static void main(String[] args) {
        final RegEx regEx = character('a')
                             .seq(character('b').rep())
                             .seq(character('c').alt(empty));
        if(regEx.matches("abbc")) {
            System.out.println("Matches!!!");
        }
    }
}
```

Disclaimer: Any bugs/misconceptions regarding this are my errors, so take everything with a grain of salt. Feel free to use the code portrayed here for any purpose whatsoever, if you do something cool with it I'd like to know, but no pressure.

Footnotes

[^1]: Sometimes the simpler something is, the harder it is to understand. See lambda calculus for example.
[^2]: I will not use ε (epsilon) to also represent the empty string since I think it is confusing, even though it is also customary.
[^3]: I think that the Scheme implementation in that article won't work if you use the repetition operator, but I haven't tested it. It might just as well be that my Scheme-foo is a bit rusty.
