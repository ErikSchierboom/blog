---
title: "Building a JSON query language - part 1: lexer"
date: 2020-02-01
tags:
  - csharp
  - domain-specific-language
  - superpower
  - lexer
---

Programming languages and their design have always been one of my favorite topics. In this series of blog posts, we'll build our own [JSON](https://www.json.org/json-en.html) query language from scratch (using some superpower(s)).

## Goals

Before defining the syntax of our language, let's set some goals:

- Simple: there should be little syntax
- Readable: code should be easy to understand
- Concise: the syntax should be compact
- Composable: queries can easily be composed into bigger queries
- Pure: side-effects are not allowed
- Immutable: mutation is forbidden

Keeping these goals in mind, we can move on to the next phase.

## Features

Next up is specifying what features we want our language to support:

- Select a member of a JSON object
- Select an element of a JSON array
- Return the length of a JSON string or array
- Test if a member or element exists
- Combine multiple queries into a single query

As can be seen, we'll keep it simple for now.

## Syntax

Our next step is to define the syntax for our language. This step is usually quite iterative, where one tries different things to see if they work. There is definitely a subjective aspect at work here too: what is a beautiful syntax to some, might be off-putting to others. Having tried out tons of different syntaxes, here's what syntax we've decided on for each of our language's features.

### Syntax for: select member of a JSON object

The [JSON spec](https://www.json.org/json-en.html) specifies that members of a JSON object are accessed using the dot (`.`) character followed by the member's name. Using the same syntax in our language allows users to re-use their existing understanding of how to work with JSON objects. This makes the language easier to learn and applies the [principle of least surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment), a very important design principle.

Syntax: `.<field>`

Examples: `.title`, `.year`, `.awards`

### Syntax for: select an element of a JSON array

Once again, we'll re-use JSON's syntax, this time for accessing array elements:

Syntax: `.[<index>]`

Examples: `.[0]`, `.[1]`, `.[239]`

### Syntax for: return the length of a JSON string or array

This is the first features for which there is no built-in JSON function. To keep with our simple, concise and readable goals, we'll introduce a `length` keyword.

Syntax: `length`

Example: `length`

### Test if a member or element exists

Once again, we have feature for which there is no built-in JSON function. Using the same argumentation as for the length feature, we'll introduce an `exists` keyword.

Syntax: `exists`

Example: `exists`

### Syntax for: combine multiple queries into a single query

To combine multiple queries into a single query, we'll need to pass the output of one invocation as input to the next invocation. As this is exactly what UNIX piping does, we'll use the same character: the pipe (`|`). Anyone familiar with UNIX systems will understand what this character will do. It also has the added benefit of being very concise.

Syntax: `<expr1> | <expr2>`

Examples: `.title | length`, `.genres | .[0]`, `.directors | .[1] | .name`

## Specification

Now that we have a rough idea what the syntax for our language looks like, it is time for a more formal specification. The formal specification will answer any open questions, such as whether negative indexes are supported or what characters can be used in field names.

There are several formats to formally define a language's syntax (also known as its _grammar_), but one of the most used is [Extended Backus-Naur Form](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form). We'll define our syntax in the [W3C EBNF dialect](https://www.w3.org/TR/REC-xml/#sec-notation), which adds support for ranges and regular-expression like multiplicity modifiers.

EBNF grammars are defined using _rules_ which usually build upon each other. Let's start by defining rules for letters and digits:

```bnf
letter ::= [a-zA-Z]
digit  ::= [0-9]
```

Anyone familiar with [regular expressions](https://en.wikipedia.org/wiki/Regular_expression) will recognize these as ranges. The `letter` rule will match any lowercase or uppercase letter and the `digit` rule will match any digit.

Using these base rules, we can then define the rules for identifiers and numbers, which are respectively one or more letters and one or more digits:

```bnf
identifier ::= letter+
number     ::= digit+
```

The uses for the select field and select element expressions use these two rules, but combined with string literals:

```bnf
field ::= '.' identifier
index ::= '.[' number ']'
```

Here, you can see that the `field` rule first specifies that the `.` string literal must occur, followed by an identifier (one or more letters). Similarly, the `index` rule starts with the `.[` string literal, followed by a number (one or more digits) and ending with the `]` string literal. The definitions answer our previously open questions: fields can only use letters (to simplify things) and indices cannot use negative numbers.

The rules for the length and exists queries are simple:

```bnf
length ::= 'length'
exists ::= 'exists'
```

We can now define a rule to match when one of the four expression rules is used:

```bnf
expr ::= field | index | length | exists
```

With this rule being defined, we support single expression queries. The final step is to allow chaining queries:

```bnf
query ::= expr ('|' expr)*
```

Here we specify that a query consists of at least one expression, followed by zero or more expressions that are preceded by the `|` character.

And with that we now a formal specification for our language defined in EBNF.

```bnf
letter     ::= [a-zA-Z]
digit      ::= [0-9]
identifier ::= letter+
number     ::= digit+

field      ::= '.' letter+
index      ::= '.[' digit+ ']'
length     ::= 'length'
exists     ::= 'exists'

expr       ::= field | index | length | exists
query      ::= expr ('|' expr)*
```

## Lexer

We are now ready to start coding! The first step in implementing a programming language is to implement a lexer (sometimes referred to as a tokenizer). A lexer converts source code (text) to a sequence of tokens (syntax). This process is called _tokenization_.

Tokens are the most basic constructs of a language. By themselves, they don't have any meaning (syntax), only when they are combined in a certain order do they gain meaning (semantics). Tokenization is _only_ concerned with syntax though, so we can ignore the semantics for now.

There are many ways to implement a lexer. One approach is to use a tool like [ANTLR](https://www.antlr.org/), which can generate a lexer from an (E)BNF specification. Another commonly used alternative is [Lex and Yacc](http://dinosaur.compilertools.net/). Hand-crafting a tokenizer is of course also possible, but we'll use a fourth option: using a [parser combinator](https://en.wikipedia.org/wiki/Parser_combinator).

## Parser combinators

A parser combinator is a library that allows parsing text by defining mini-parsers, which are defined as functions. What makes parser combinators great is that they offer a suite of functions to combine simple parsers into more complex parsers. This makes it a perfect fit for implementing our lexer.

The library we'll be using is called [Superpower](https://github.com/datalust/superpower), which is itself the successor of [Sprache](https://github.com/sprache/Sprache).

## Implementation

Superpower allows us to define parsers in two ways:

1. Text-based parsers, which are fairly high-level
1. Token-based parsers, which are more low-level

The benefit of token-based parsers is that they can provide better error messages than text-based parsers. As (good) error messages are very important to a compiler, we'll go with the token-based parser approach.

### Tokens

Let's recall what our EBNF grammar looks like:

```bnf
letter     ::= [a-zA-Z]
digit      ::= [0-9]
identifier ::= letter+
number     ::= digit+

field      ::= '.' letter+
index      ::= '.[' digit+ ']'
length     ::= 'length'
exists     ::= 'exists'

expr       ::= field | index | length | exists
query      ::= expr ('|' expr)*
```

To convert this grammar into a format that Superpower can work with, we need to define an enum with its members representing each type of token. Each token type member must also be annotated with the `[Token]` attribute.

The first token type we'll define members for are _keywords_, which are words that have a special meaning in a language. Our language has two keywords: `length` and `exists`, which we'll define as follows:

```csharp
// TODO: add using statement

public enum TokenKind
{
    [Token]
    LengthKeyword,

    [Token]
    ExistsKeyword
}
```

Note that these tokens directly correspond to the `length` and `exists` rules from our grammar. We can also define token types for our `identifier` and `number` rules:

```csharp
public enum TokenKind
{
    // Keyword members ...

    [Token]
    Identifier,

    [Token]
    Number,
}
```

The final token types we need to define are for the characters used in the string literals (`.`, `[`, `]` and `|`):

```csharp
public enum TokenKind
{
    // Keyword, identifier and number members ...

    [Token]
    Dot,

    [Token]
    OpenBracket,

    [Token]
    CloseBracket,

    [Token]
    Pipe
}
```

Note that the names of the tokens reflect the character they represent, _not_ how we interpret them later on. This makes sense as the same token can have a different meaning depending on the context.

### Tokenizer

The next step is to implement a _tokenizer_, which will take a string (the source code) and parses it to a sequence of tokens. In Superpower, a tokenizer is represented by the generic `Tokenizer<T>` type, where `T` is the type of the `enum` defining the tokens.

To create a tokenizer, the `TokenizerBuilder<T>` type is used. This builder class has a nice fluent interface for mapping source code to tokens. Let start with building a minimal tokenizer that can parse the `.` character to `TokenKind.Dot`:

```csharp
Tokenizer<TokenKind> tokenizer =
    new TokenizerBuilder<TokenKind>()
        .Match(Span.EqualTo("."), TokenKind.Dot)
        .Build();
```

We can then run

```csharp
tokenizer.TryTokenize(source.Trim());
```

TODO: check if there is Tokenize available (no Try)`

```csharp
using Superpower;
using Superpower.Model;
using Superpower.Parsers;
using Superpower.Tokenizers;

public static class Lexer
{
    public static Result<TokenList<TokenKind>> Tokenize(string source) =>
        CreateTokenizer().TryTokenize(source.Trim());

    private static Tokenizer<TokenKind> CreateTokenizer() =>
        new TokenizerBuilder<TokenKind>()
            .Ignore(Span.WhiteSpace)
            .Match(Numerics.IntegerInt32, TokenKind.Number)
            .Match(Span.EqualTo("."), TokenKind.Dot)
            .Match(Span.EqualTo("["), TokenKind.OpenBracket)
            .Match(Span.EqualTo("]"), TokenKind.CloseBracket)
            .Match(Span.EqualTo("|"), TokenKind.Pipe)
            .Match(Span.EqualTo("length"), TokenKind.LengthKeyword)
            .Match(Span.EqualTo("exists"), TokenKind.ExistsKeyword)
            .Match(Span.EqualTo("first"), TokenKind.FirstKeyword)
            .Match(Character.Letter.AtLeastOnce(), TokenKind.Identifier)
            .Build();
}
```

```csharp
using Superpower.Display;

namespace Spil.Language
{
    public enum TokenKind
    {
        [Token(Example = ".", Category = "token")]
        Dot,

        [Token(Example = "[", Category = "token")]
        OpenBracket,

        [Token(Example = "]", Category = "token")]
        CloseBracket,

        [Token(Example = "|", Category = "token")]
        Pipe,

        [Token(Example = "name", Category = "identifier")]
        Identifier,

        [Token(Example = "1", Category = "literal")]
        Number,

        [Token(Example = "length", Category = "keyword")]
        LengthKeyword,

        [Token(Example = "exists", Category = "keyword")]
        ExistsKeyword
    }
}
```

```csharp
using Superpower;
using Superpower.Model;
using Superpower.Parsers;
using Superpower.Tokenizers;

namespace Spil.Language
{
    public static class Lexer
    {
        private static readonly Tokenizer<TokenKind> TokenizerImpl = CreateTokenizer();

        private static Tokenizer<TokenKind> CreateTokenizer() =>
            new TokenizerBuilder<TokenKind>()
                .Ignore(Span.WhiteSpace)
                .Match(Numerics.IntegerInt32, TokenKind.Number)
                .Match(Span.EqualTo("."), TokenKind.Dot)
                .Match(Span.EqualTo("["), TokenKind.OpenBracket)
                .Match(Span.EqualTo("]"), TokenKind.CloseBracket)
                .Match(Span.EqualTo("|"), TokenKind.Pipe)
                .Match(Span.EqualTo("length"), TokenKind.LengthKeyword)
                .Match(Span.EqualTo("exists"), TokenKind.ExistsKeyword)
                .Match(Span.EqualTo("first"), TokenKind.FirstKeyword)
                .Match(Character.Letter.AtLeastOnce(), TokenKind.Identifier)
                .Build();

        public static Result<TokenList<TokenKind>> Tokenize(string source) =>
            TokenizerImpl.TryTokenize(source.Trim());
    }
}
```

## Conclusion

TODO

In the [next post]({{< ref "/posts/building-a-json-query-language-part-2-parser" >}}), we'll build on our lexer and parse the tokens into a structure that describes code in our language.
