---
layout: post
title:  "Pratt Parsers"
date:   2011-03-14 19:10:00 -0300
categories: post
---

Some time ago I came across [Pratt parsers](http://crockford.com/javascript/tdop/tdop.html). I had never seen them before, and I found them quite elegant.

They were first described by Vaughan Pratt in the 1973 paper "Top down operator precedence". From a theoretical perspective they are not particularly interesting, but from an engineering point of view they are fantastic.

Let's start with a real-world example. This is the grammar from the expression language for my [Performance Invariants agent](https://github.com/juancn/performance):

```java
/* omitted */
import static performance.compiler.TokenType.*;

public final class SimpleGrammar
    extends Grammar<TokenType> {
    private SimpleGrammar() {
        infix(LAND, 30);
        infix(LOR, 30);

        infix(LT, 40);
        infix(GT, 40);
        infix(LE, 40);
        infix(GE, 40);
        infix(EQ, 40);
        infix(NEQ, 40);

        infix(PLUS, 50);
        infix(MINUS, 50);

        infix(MUL, 60);
        infix(DIV, 60);

        unary(MINUS, 70);
        unary(NOT, 70);

        infix(DOT, 80);

        clarifying(LPAREN, RPAREN, 0);
        delimited(DOLLAR_LCURLY, RCURLY, 70);

        literal(INT_LITERAL);
        literal(LONG_LITERAL);
        literal(FLOAT_LITERAL);
        literal(DOUBLE_LITERAL);
        literal(ID);
        literal(THIS);
        literal(STATIC);
    }

    public static Expr<TokenType> parse(final String text) throws ParseException {
        final Lexer<TokenType> lexer = new JavaLexer(text, 0 , text.length());
        final PrattParser<TokenType> prattParser = new PrattParser<TokenType>(INSTANCE, lexer);
        final Expr<TokenType> expr = prattParser.parseExpression(0);
        if(prattParser.current().getType() != EOF) {
            throw new ParseException("Unexpected token: " + prattParser.current());
        }
        return expr;
    }

    private static final SimpleGrammar INSTANCE = new SimpleGrammar();
}
```

Pretty, isn't it?

The number represents a precedence, for infix operators is quite obvious (it's basically a precedence table), but for clarifying and delimited expressions it sets the lower bound for the subexpression. In the grammar above, the delimited expression only accepts dot expressions and literals, parenthesis on the other hand, accept anything.

So, how does the parser work? The PrattParser itself is rather elegant also:

```java
/* omitted */
public final class PrattParser<T> {
    private final Grammar<T> grammar;
    private final Lexer<T> lexer;
    private Token<T> current;

    public PrattParser(Grammar<T> grammar, Lexer<T> lexer)
            throws ParseException
    {
        this.grammar = grammar;
        this.lexer = lexer;
        current = lexer.next();
    }

    public Expr<T> parseExpression(int stickiness) throws ParseException {
        Token<T> token = consume();
        final PrefixParser<T> prefix = grammar.getPrefixParser(token);
        if(prefix == null) {
            throw new ParseException("Unexpected token: " + token);
        }
        Expr<T> left = prefix.parse(this, token);

        while (stickiness < grammar.getStickiness(current())) {
            token = consume();

            final InfixParser<T> infix = grammar.getInfixParser(token);
            left = infix.parse(this, left, token);
        }

        return left;
    }

    public Token<T> current() {
        return current;
    }

    public Token<T> consume() throws ParseException {
        Token<T> result = current;
        current = lexer.next();
        return result;
    }
}
```

All the magic happens in the parseExpression method.

Given the current token, it fetches an appropriate prefix parser. Prefix parsers recognize simple expressions (such as literals, unary operators, delimited expressions, etc.). Then it goes to process infix parsers according to precedence (stickiness).

Pratt parsers are a variation of [recursive descent parsers](http://en.wikipedia.org/wiki/Recursive_descent_parser). The parseExpression methods represents a generalized rule in the grammar.

At this point you're thinking there must be more to this. The trick must be in the Grammar class:

```java
/* omitted */
public class Grammar<T> {
    private Map<T, PrefixParser<T>> prefixParsers = new HashMap<T, PrefixParser<T>>();
    private Map<T, InfixParser<T>>  infixParsers = new HashMap<T, InfixParser<T>>();

    PrefixParser<T> getPrefixParser(Token<T> token) {
        return prefixParsers.get(token.getType());
    }

    int getStickiness(Token<T> token) {
        final InfixParser infixParser = getInfixParser(token);
        return infixParser == null?Integer.MIN_VALUE:infixParser.getStickiness();
    }

    InfixParser<T> getInfixParser(Token<T> token) {
        return infixParsers.get(token.getType());
    }

    protected void infix(T ttype, int stickiness)
    {
        infix(ttype, new InfixParser<T>(stickiness));
    }

    protected void infix(T ttype, InfixParser<T> value) {
        infixParsers.put(ttype, value);
    }

    protected void unary(T ttype, int stickiness)
    {
        prefixParsers.put(ttype, new UnaryParser<T>(stickiness));
    }
    protected void literal(T ttype)
    {
        prefix(ttype, new LiteralParser<T>());
    }

    protected void prefix(T ttype, PrefixParser<T> value) {
        prefixParsers.put(ttype, value);
    }

    protected void delimited(T left, T right, int subExpStickiness) {
        prefixParsers.put(left, new DelimitedParser<T>(right, subExpStickiness, true));
    }

    protected void clarifying(T left, T right, int subExpStickiness) {
        prefixParsers.put(left, new DelimitedParser<T>(right, subExpStickiness, false));
    }
}
```

Nope. Just a couple of maps and some factory methods.

Even the infix and prefix parsers are rather simple:

```java
public class InfixParser<T> {
    private final int stickiness;
    protected InfixParser(int stickiness) {
        this.stickiness = stickiness;
    }

    public Expr<T> parse(PrattParser<T> prattParser, Expr<T> left, Token<T> token)
            throws ParseException {
        return new BinaryExpr<T>(token, left, prattParser.parseExpression(getStickiness()));
    }

    protected int getStickiness() {
        return stickiness;
    }
}

class LiteralParser<T>
        extends PrefixParser<T> {
    public Expr<T> parse(PrattParser<T> prattParser, Token<T> token)
            throws ParseException {
        return new ConstantExpr<T>(token);
    }
}

class UnaryParser<T>
    extends PrefixParser<T> {
    private final int stickiness;

    public UnaryParser(int stickiness) {
        this.stickiness = stickiness;
    }

    public Expr<T> parse(PrattParser<T> prattParser, Token<T> token)
            throws ParseException {
        return new UnaryExpr<T>(token, prattParser.parseExpression(stickiness));
    }
}
```

The infix and prefix parsers, just build an AST node. They recursively parse sub-expressions if necessary. If you want to check how delimited expressions work, you can [browse the code in github](https://github.com/juancn/performance).

These parsers have several interesting characteristics. One one of them is that the grammar can be modified at runtime (even though it's not shown here) by adding/removing parsers, even while parsing. You can also easily add conditional grammars for sub-languages (think embedded SQL for example).

The code shown here only supports an LL(1) grammar (if I'm not mistaken), but adding additional lookahead should allow for LL(k) grammars.

Another interesting fact is that they way the parser is extended (by adding infix/prefix parsers) naturally yields grammars without left recursion.

One thing to note is that in my simple expression language, I'm not syntactically restricting the types of sub-expressions that infix operators receive, so that has to be checked in a later stage.

The only downside I can think of (besides the LL(k)-ness), is that these parsers are heavily geared towards expressions (everything is an expressions), but with some creativity statements could be added. For example, you could treat the semicolon in Java/C/C++/etc. as an infix operator.

Feel free to take all this code as yours for any purpose whatsoever. Happy hacking!
