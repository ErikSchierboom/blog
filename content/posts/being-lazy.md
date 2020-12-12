---
title: "Being lazy"
date: 2014-08-16
tags:
  - csharp
  - dotnet
  - laziness
---

When optimizing software, code that is never executed can of course be removed. But how to optimize code that is used sometimes, but not all the time? Ideally, you'd only run that code when you need it; this is called [_lazy-loading_](http://en.wikipedia.org/wiki/Lazy_loading) or _lazy-initialization_.

Here is a sample that shows a naive loading strategy:

```csharp
public class NaiveLoading
{
    private int naiveResult = Calculate();

    public int Result
    {
        get { return naiveResult; }
    }
}

// The result will be calculated when the instance is created,
// even though we might never access the Result property
var naiveLoading = new NaiveLoading();
```

A less naive solution would be to move the calculation to the `Result` property:

```csharp
public class LessNaiveLoading
{
    public int Result
    {
        get { return Calculate(); }
    }
}

// This will NOT calculate the result
var lessNaiveLoading = new LessNaiveLoading();

// First time calling will calculate result
var result1 = lessNaiveLoading.Result;

// Second time calling will re-calculate result
var result2 = lessNaiveLoading.Result;
```

We now only calculate the result when needed, but we do it every single time `Result` is accessed, which our naive example did not. The following example combines the best of both methods:

```csharp
public class LazyLoading
{
    private int? lazyResult = null;

    public int Result
    {
        get
        {
            if (lazyResult == null)
            {
                lazyResult = Calculate();
            }

            return lazyResult.Value;
        }
    }
}

// This will NOT calculate the result
var lazyLoading = new LazyLoading();

// First time calling will calculate result
var result1 = lazyLoading.Result;

// Second time calling will return the previously calculated result
var result2 = lazyLoading.Result;
```

Through some simple boilerplate code, we managed to optimize our class: it will only calculate the result when needed and will only do so once.

## Lazy&lt;T&gt;

The .NET framework version 4.0 introduced the `Lazy<T>` class to easily enable lazy-loading in your code. Let's convert our previous example to a `Lazy<T>` version:

```csharp
public class LazyTypeLoading
{
    private Lazy<int> lazyResult = new Lazy<int>(() => Calculate());

    public int Result
    {
        get { return lazyResult.Value; }
    }
}

// This will NOT calculate the result
var lazyTypeLoading = new LazyTypeLoading();

// First time calling will calculate result
var result1 = lazyTypeLoading.Result;

// Second time calling will return the previously calculated result
var result2 = lazyTypeLoading.Result;
```

We replaced our boilerplate code with a `Lazy<int>` instance. When creating this instance, we provide it with a lambda expression that will be used to compute the return value. Note that the lambda expression is **not** executed when the instance is created, it is only stored for later use.

When the `Value` property of a `Lazy<T>` instance is accessed, it checks if the value has already been calculated; if so, it returns that value and if not, it calculates the value, stores it and then returns it.

You can use the `IsValueCreated` property to check if the `Value` has already been calculated:

```csharp
Lazy<int> lazy = new Lazy<int>(() => 3);

lazy.IsValueCreated; // Returns false
lazy.Value;          // Will execute the lambda, returns 3
lazy.IsValueCreated; // Returns true
```

With lazy-loading, keep in mind that the value is calculated only once:

```csharp
Lazy<DateTime> lazy = new Lazy<DateTime>(() => DateTime.Now);

var value1 = lazy.Value; // First time, returns current time

Thread.Sleep(2000);

var value2 = lazy.Value; // Second time, returns previous value

value1 == value2; // Returns true!
```

Note that by default, `Lazy<T>` instances are thread safe. If you don't want it to be thread-safe, you can use one of the [constructor overloads](<http://msdn.microsoft.com/en-us/library/dd642318(v=vs.110).aspx>) that let you specify the thread safety mode:

```csharp
var nonThreadSafeLazy = new Lazy<int>(LazyThreadSafetyMode.None);
```

## Conclusion

Lazy loading can be a good optimization strategy, but you should only use if when performance is an issue. When you do have a use case for lazy-loading, the `Lazy<T>` class makes it easy to implement.
