---
title: "Building a JSON query language - part 1: lexer"
date: 2020-02-01
tags:
  - csharp
  - domain-specific-language
  - superpower
  - lexer
---

Programming languages and their design have always been one of my favorite topics. In this series of blog posts, we'll build our own JSON query language.

## Goals

Before defining its syntax, let's set some goals for our language:

- Simple: the syntax should be easy to read.
- Concise: the syntax should be short
- Composable: queries can easily be composed into bigger queries.
- Pure: mutation or side-effects are not allowed.

We'll keep these goals in mind while designing our language.

## Features

So what features should our language have? Let's compile a list of the things we want our language to support:

- Select a property of a JSON object.
- Select an element of a JSON array.
- Return the length of a JSON string or array.
- Test if a property or element exists.
- Pipe the output of one query into another query.

## Syntax

Our next step is to define the syntax for our language. This step is usually very iterative, where one keeps trying different things to see if they work.

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
