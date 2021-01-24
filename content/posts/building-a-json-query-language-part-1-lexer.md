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

- Concise: the syntax should be short and simple.
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

Our next step is to define the syntax for our language. This step is usually a quite iterative one, where you keep trying different things to see if they work.

## Lexer

TODO: difference lexer/tokenizer

## Superpower

https://github.com/datalust/superpower
https://github.com/sprache/Sprache

## Conclusion

TODO

In the [next post]({{< ref "/posts/building-a-json-query-language-part-2-parser" >}}), we'll build on our lexer and parse the tokens into a structure that describes code in our language.
