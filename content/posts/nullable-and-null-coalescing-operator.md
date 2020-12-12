---
title: "Nullable<T> and the ?? operator"
date: 2014-08-04
tags:
  - csharp
  - dotnet
  - nullable
  - null-coalescing-operator
url: /2014/08/04/nullable-and-null-coalescing-operator/
---

## Nullable value types

The concept of a `null` value is simple: it denotes the _absence_ of a value. In the first version of C#, you could not have `null` value types. However, C# 2.0 introduced the `Nullable<T>` type to remedy this:

```csharp
Nullable<int> nullInt = new Nullable<int>();
nullInt.HasValue;    // Returns false
nullInt.Value;       // Throws exception as no value has been set

Nullable<int> nonNullInt = new Nullable<int>(2);
nonNullInt.HasValue; // Returns true
nonNullInt.Value;    // Returns 2
```

You can also directly assign or retrieve the value:

```csharp
Nullable<int> nullInt = null;
Nullable<double> nonNullDouble = 2.0;
```

Even better, appending the **?** keyword to a type makes it a `Nullable<T>`:

```csharp
int? nullInt = 2;         // Type is Nullable<int>
bool? nonNullBool = true; // Type is Nullable<bool>
```

Comparing nullable types also works as expected:

```csharp
int? nullInt = null;
int? nonNullInt = 2;

nullInt == null;    // Returns true
nonNullInt == 2;    // Returns true
nonNullInt == null; // Returns false
```

The `Nullable<T>` class itself is implemented as a value type, so can we create a nullable `Nullable<T>`? Let's try:

```csharp
Nullable<Nullable<int>> nullableInception = null;
```

It turns out we can't; the above code does _not_ compile. This is due to the fact that the `Nullable<T>` struct does not allow nullable types to be specified as its generic type parameter (see [MSDN](http://msdn.microsoft.com/en-us/library/d5x73970.aspx)).

### The **??** operator

Besides adding `Nullable<T>`, C# 2.0 introduced another feature that deals with `null` values: the **??** operator (also known as the _null-coalescing_ operator). It returns the left-hand operand if that is not null; otherwise it returns the right hand operand. This simple operator can greatly simplify your `null` checks:

```csharp
public static string NullCheckWithIfStatement()
{
    if (str == null)
    {
        return string.Empty;
    }

    return str;
}

public static string NullCheckWithTernaryOperator()
{
    return str == null ? string.Empty : str;
}

public static string NullCheckWithNullCoalescingOperator()
{
    return str ?? string.Empty;
}
```

The **??** operator manages to be both concise _and_ very readable.

With more complex statements, the difference becomes even more striking:

```csharp
public static string ComplexNullCheckWithIfStatement()
{
    string result1 = GetPotentiallyNullString1();

    if (result1 != null)
    {
        return result1;
    }

    string result2 = GetPotentiallyNullString2();

    if (result2 != null)
    {
        return result2;
    }

    return string.Empty;
}

public static string ComplexNullCheckWithNullCoalescingOperator()
{
    return GetPotentiallyNullString1() ??
           GetPotentiallyNullString2() ??
           string.Empty;
}
```

Of course, you can also use it on `Nullable<T>` values:

```csharp
int? nullInt = null;
int? nonNullInt = 2;

nullInt ?? 8;    // Returns 8
nonNullInt ?? 5; // Returns 2
```

If you use the **??** operator to return a type's default value, you can also use the `GetValueOrDefault()` method:

```csharp
int? nullInt = null;
int? nonNullInt = 2;

nullInt.GetValueOrDefault();    // Returns 0
nullInt ?? default(int);        // Returns 0

nonNullInt.GetValueOrDefault(); // Returns 2
nonNullInt ?? default(int);     // Returns 2
```

## Conclusion

Adding support for nullable value types was a very useful addition to C#. I use it often when dealing with databases, where nullable value types are common.

The **??** operator is a personal favorite of mine due to its conciseness and usefulness.
