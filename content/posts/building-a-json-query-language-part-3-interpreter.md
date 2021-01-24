---
title: "Building a JSON query language - part 3: interpreter"
date: 2020-02-01
tags:
  - csharp
  - domain-specific-language
  - superpower
  - interpreter
---

## Implementation

```csharp
using System;
using System.Linq;
using Newtonsoft.Json.Linq;

namespace Spil.Language
{
    public static class Interpreter
    {
        public static string Evaluate(string sourceText, string json)
        {
            var result = Parser.ParseExpression(sourceText);
            if (!result.HasValue)
                throw new InvalidOperationException(result.FormatErrorMessageFragment());

            var loweredExpression = Lowering.Apply(result.Value);

            return Evaluate(loweredExpression, JToken.Parse(json)) switch
            {
                {Type: JTokenType.Null or JTokenType.None or JTokenType.Undefined} => "null",
                {Type: JTokenType.Boolean} jsonElement => jsonElement.ToString().ToLower(),
                { } jsonElement => jsonElement.ToString()
            };
        }

        private static JToken Evaluate(Expression expression, JToken jsonToken) =>
            (expression, jsonToken) switch
            {
                (PropertyExpression property, JObject jsonObject) => jsonObject[property.Name] ?? JValue.CreateNull(),
                (PropertyExpression, _) => throw new InvalidOperationException("Cannot use property expression on non-object element"),
                (ArrayExpression array, JArray jsonArray) => jsonArray.ElementAtOrDefault(array.Index) ?? JValue.CreateNull(),
                (ArrayExpression, _) => throw new InvalidOperationException("Cannot use array expression on non-array element"),
                (LengthExpression, JArray jsonArray) => new JValue(jsonArray.Count),
                (LengthExpression, JValue {Type: JTokenType.String} jsonValue) => new JValue(jsonValue.Value<string>().Length),
                (LengthExpression, _) => throw new InvalidOperationException("Cannot use length expression on non-array element"),
                (ExistsExpression, _) => new JValue(jsonToken.Type != JTokenType.None && jsonToken.Type != JTokenType.Null),
                (PipeExpression composite, _) => Evaluate(composite.Right, Evaluate(composite.Left, jsonToken)),
                _ => throw new ArgumentOutOfRangeException(nameof(expression), "Invalid expression type")
            };
    }
}
```

## Conclusion

TODO

In the [next post]({{< ref "/posts/building-a-json-query-language-part-4-lowering" >}}), we'll look at what lowering is and how we can use it to introduce new features seamlessly.
