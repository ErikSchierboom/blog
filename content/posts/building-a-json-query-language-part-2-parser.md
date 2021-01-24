---
title: "Building a JSON query language - part 2: parser"
date: 2020-02-01
tags:
  - csharp
  - domain-specific-language
  - superpower
  - parser
---

## Implementation

```csharp
namespace Spil.Language
{
    public abstract record Expression;
    public record PropertyExpression(string Name): Expression;
    public record ArrayExpression(int Index): Expression;
    public record LengthExpression: Expression;
    public record ExistsExpression: Expression;
    public record PipeExpression(Expression Left, Expression Right) : Expression;
}
```

```csharp
using Superpower;
using Superpower.Model;
using Superpower.Parsers;

namespace Spil.Language
{
    public static class Parser
    {
        public static TokenListParserResult<TokenKind, Expression> ParseExpression(string sourceText)
        {
            var tokenList = Lexer.Tokenize(sourceText);
            if (tokenList.HasValue)
                return PExpression.TryParse(tokenList.Value);

            return TokenListParserResult.Empty<TokenKind, Expression>(tokenList.Value, tokenList.ErrorPosition, tokenList.ErrorMessage);
        }

        private static TokenListParser<TokenKind, Expression> PPropertyExpression =>
            from dot in Token.EqualTo(TokenKind.Dot)
            from identifier in Token.EqualTo(TokenKind.Identifier).Apply(Character.Letter.AtLeastOnce())
            select (Expression)new PropertyExpression(new string(identifier));

        private static TokenListParser<TokenKind, Expression> PArrayExpression =>
            from dot in Token.EqualTo(TokenKind.Dot)
            from openBracket in Token.EqualTo(TokenKind.OpenBracket)
            from index in Token.EqualTo(TokenKind.Number).Apply(Numerics.IntegerInt32)
            from closeBracket in Token.EqualTo(TokenKind.CloseBracket)
            select (Expression)new ArrayExpression(index);

        private static TokenListParser<TokenKind, Expression> PLengthExpression =>
            Token.EqualTo(TokenKind.LengthKeyword).Value((Expression)new LengthExpression());

        private static TokenListParser<TokenKind, Expression> PExistsExpressions =>
            Token.EqualTo(TokenKind.ExistsKeyword).Value((Expression)new ExistsExpression());

        private static TokenListParser<TokenKind, Expression> PFirstExpressions =>
            Token.EqualTo(TokenKind.FirstKeyword).Value((Expression)new FirstExpression());

        private static TokenListParser<TokenKind, Expression> PTermExpression =>
            PArrayExpression.Try().Or(PPropertyExpression).Or(PLengthExpression).Or(PExistsExpressions).Or(PFirstExpressions);

        private static TokenListParser<TokenKind, Expression> PPipeExpression =>
             Parse.Chain(
                 Token.EqualTo(TokenKind.Pipe),
                 PTermExpression,
                 (_, left, right) => (Expression) new PipeExpression(left, right));

         private static TokenListParser<TokenKind, Expression> PExpression =>
             PPipeExpression.Or(PTermExpression);
    }
}
```

## Conclusion

TODO

TODO

In the [next post]({{< ref "/posts/building-a-json-query-language-part-3-interpreter" >}}), we'll build an interpreter that will allow us to execute programs written in our language.
