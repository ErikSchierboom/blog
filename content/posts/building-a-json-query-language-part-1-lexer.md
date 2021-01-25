---
title: "Building a JSON query language - part 1: lexer"
date: 2020-02-01
tags:
  - csharp
  - domain-specific-language
  - superpower
  - lexer
---

Programming languages and their design have always been one of my favorite topics. In this series of blog posts, we'll build our own [JSON](https://www.json.org/json-en.html) query language from scratch (with a little help).

## Goals

Before defining its syntax, let's set some goals for our language:

- Simple: there should be little syntax
- Readable: code should be easy to understand
- Concise: the syntax should be compact
- Composable: queries can easily be composed into bigger queries
- Pure: side-effects are not allowed
- Immutable: mutation is forbidden

Keeping these goals in mind, we can move on to the next phase.

## Features

A language without any features is quite boring, so let's define what we want to do with our language:

- Select a member of a JSON object
- Select an element of a JSON array
- Return the length of a JSON string or array
- Test if a member or element exists
- Combine multiple queries into a single query

As can be seen, we'll keep it simple for demonstration purposes.

## Syntax

Our next step is to define the syntax for our language. This step is usually quite iterative, where one keeps trying different things to see if they work. There is definitely a subjective aspect at work here: what is a beautiful syntax to some, might be off-putting to others. Having tried out tons of different syntaxes, here's what syntax we've decided on for our language's features.

### Select member of a JSON object

Members of a JSON object are accessed using the dot (`.`) character followed by the member's name. Using the same syntax in our language allows users to re-use their existing understanding of how to work with JSON objects. This makes the language easier to learn and applies the [principle of least surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment), a very important design principle.

Syntax: `.<field>`

Examples: `.title`, `.year`, `.awards`

### Select an element of a JSON array

Once again, we'll re-use JSON's syntax, this time for accessing array elements:

Syntax: `.[<index>]`

Examples: `.[0]`, `.[1]`, `.[239]`

### Return the length of a JSON string or array

This is the first features for which there is no built-in JSON function. To keep with our simple, concise and readable goals, we'll introduce a `length` keyword.

Syntax: `length`

Example: `length`

### Test if a member or element exists

Once again, we have feature for which there is no built-in JSON function. Using the same argumentation as for the length feature, we'll introduce an `exists` keyword.

Syntax: `exists`

Example: `exists`

### Combine multiple queries into a single query

To combine multiple queries into a single query, we'll need to pass the output of one invocation as input of the next invocation. As this is exactly what UNIX piping does, we'll use the same character: the pipe (`|`). for this, as this both neatly fits with the UNIX syntax that most people will be familiar with _and_ has the added benefit of being very concise.

Syntax: `<expr1> | <expr2>`

Examples: `.title | length`, `.genres | .[0]`, `.directors | .[1] | .name`

## Specification

Now that we have an idea what the syntax for our language looks like, it is time for a formal specification. We need a more formal specification as there are still open questions, like what characters are allowed in member names or whether negative indexes are allowed.

There are several formats to define a language's syntax, but one of the most used is [Extended Backus-Naur Form](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form). We'll define our syntax in the [W3C EBNF dialect](https://www.w3.org/TR/REC-xml/#sec-notation), which adds support for ranges and regular-expression like multiplicity modifiers.

```bnf
letter ::= [a-zA-Z]
digit  ::= [0-9]

field  ::= '.' letter+;
index  ::= '.[' digit+ ']'
length ::= 'length'
exists ::= 'exists'

expr   ::= field | index | exists
query  ::= expr ('|' expr)*
```

## Lexer

TODO: difference lexer/tokenizer

## Superpower

https://github.com/datalust/superpower
https://github.com/sprache/Sprache

## Implementation

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
