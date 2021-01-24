---
title: "Building a JSON query language - part 4: lowering"
date: 2020-02-01
tags:
  - csharp
  - domain-specific-language
  - superpower
  - lowering
---

## Why lowering?

## Implementation

```csharp
using System;

namespace Spil.Language
{
    public abstract class ExpressionRewriter
    {
        public virtual Expression VisitExpression(Expression expression) => expression switch
        {
            ArrayExpression arrayExpression => VisitArrayExpression(arrayExpression),
            ExistsExpression existsExpression => VisitExistsExpression(existsExpression),
            LengthExpression lengthExpression => VisitLengthExpression(lengthExpression),
            FirstExpression firstExpression => VisitFirstExpression(firstExpression),
            PipeExpression pipeExpression => VisitPipeExpression(pipeExpression),
            PropertyExpression propertyExpression => VisitPropertyExpression(propertyExpression),
            _ => throw new ArgumentOutOfRangeException(nameof(expression))
        };

        public virtual Expression VisitPropertyExpression(PropertyExpression expression) => expression;

        public virtual Expression VisitArrayExpression(ArrayExpression expression) => expression;

        public virtual Expression VisitLengthExpression(LengthExpression expression) => expression;

        public virtual Expression VisitExistsExpression(ExistsExpression expression) => expression;

        public virtual Expression VisitFirstExpression(FirstExpression expression) => expression;

        public virtual Expression VisitPipeExpression(PipeExpression expression) =>
            new PipeExpression(VisitExpression(expression.Left), VisitExpression(expression.Right));
    }
}
```

```csharp
namespace Spil.Language
{
    public static class Lowering
    {
        public static Expression Apply(Expression expression) =>
            new FirstRewriter().VisitExpression(expression);

        private class FirstRewriter : ExpressionRewriter
        {
            public override Expression VisitFirstExpression(FirstExpression expression) =>
                new ArrayExpression(0);
        }
    }
}
```

## Conclusion

TODO
