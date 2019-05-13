---
title: "Extension methods - advanced"
date: 2014-07-27
tags: 
  - C#
  - .NET
  - Extension methods
url: /2014/07/07/extension-methods-advanced/
---

Previously, the [Extension methods - An introduction]({{< ref "/posts/extension-methods-an-introduction" >}}) post showed the basics of extension methods. This post will show more advanced usages of extension methods.

## Gracefully handle null values
Regular instance methods throw a `NullReferenceException` when called on a `null` instance. However, extension methods are static methods and thus can choose how to handle `null` values. We can use this to create methods that appear to be instance methods, but don't throw a `NullReferenceException` when invoked on a `null` instance.

As an example, this extension method defines a null-safe wrapper for the existing [`GetHashCode()`](http://msdn.microsoft.com/en-us/library/system.object.gethashcode\(v=vs.110\).aspx) method:

```csharp
public static class ObjectExtensions
{
    public static int GetHashCodeNullSafe(this object obj)
    {
        return obj == null ? 0 : obj.GetHashCode();
    }
}

3.GetHashCodeNullSafe()); // Returns 3
3.GetHashCode());         // Returns 3

string nullString = null;
nullString.GetHashCodeNullSafe() // Returns 0
nullString.GetHashCode()         // Throws NullReferenceException
```

Note: be careful when using this technique, as people expect a `NullReferenceException` to be thrown when an instance method is called on a `null` instance.

## Hiding functionality
Sometimes, you want to hide advanced functionality of a class by default. You could do this by creating two classes: a `Basic` class with only basic methods and an `Advanced` class that extends the `Basic` class and adds the advanced methods.

Extension methods allow you to do this without inheritance. You define the `Basic` class as before, but the advanced methods are now defined as extension methods in a *different namespace* from the `Basic` class. This means that for users to be able to access the advanced methods, they have to explicitly include the namespace in which the advanced methods are defined.

As an example, say that we have the following code:

```csharp
namespace Core
{
    public class Basic
    {
        public void BasicMethod() {}
    }
}

namespace Core.Advanced
{   
    public static class BasicExtensions
    {
        public static void AdvancedMethod(this Basic instance) {}
    }
}
```

This setup means that by default, only `BasicMethod()` can be called on `Basic` class instances. However, when the `Core.Advanced` namespace is included, `AdvancedMethod()` will also be available. 

This approach has one clear disadvantage: the advanced methods can only work on public members of the class they are extending.

## Creating fluent interfaces
[Fluent interfaces](http://en.wikipedia.org/wiki/Fluent_interface) allow chaining of methods. They do this by returning a value from the method (usually the instance on which the method was called), which can then be used to call members on. 

The extension methods provided by [LINQ](http://msdn.microsoft.com/en-us/library/bb397926.aspx) form a fluent interface:

```csharp
var numbers = new [] { 1, 2, 3, 4, 5 };

// Will return [5, 3, 1]
var orderedNumbers = numbers.Where(n => n % 2 == 1)
                            .OrderByDescending(n => n)
                            .ToArray();
```

However, sometimes you might want methods to be chainable but they are not:

```csharp
var list = new List<int>();

list.Add(1)
    .Add(2)
    .Add(3);
```

This will lead to a compile-time error:

`Operator '.' cannot be applied to operand of type 'void'`

If we look at the [definition](http://msdn.microsoft.com/en-us/library/3wcytfd1\(v=vs.110\).aspx) of the `Add()` method, we can see that it returns `void` and thus is not chainable:

```csharp
public void Add(T item);
```

The following extension method *does* allow fluently adding items:

```csharp
public static class ListExtensions
{
    public static List<T> AddFluent<T>(this List<T> list, T item)
    {
        list.Add(item);

        return list;       
    }
}
```

We can use this extension method to fluently add items:

```csharp
var list = new List<int>();

list.AddFluent(1)
    .AddFluent(2)
    .AddFluent(3);
```

## Use in older versions of the .NET framework
As extension methods are compiled to plain static method calls, they can be used in .NET 2.0. However, if you try to compile an extension method in a .NET 2.0 project, you'll get the following error:

`Cannot define a new extension method because the compiler required type 'System.Runtime.CompilerServices.ExtensionAttribute' cannot be found. Are you missing a reference to System.Core.dll?`

It turns out that the compiler tries to add the `ExtensionAttribute` type to compiled extension methods, but can't find that type (it expects to find it in `System.Core.Dll`). 

The solution is simple: define the `ExtensionAttribute` yourself:

```csharp
namespace System.Runtime.CompilerServices
{
    [AttributeUsageAttribute(AttributeTargets.Assembly | 
                             AttributeTargets.Class | 
                             AttributeTargets.Method)] 
    public class ExtensionAttribute : Attribute {}
}
```

Now your code should compile and you can use extension methods on .NET 2.0.

## Find extensions methods at runtime 
As discussed in the previous section, extension methods are automatically decorated with the `ExtensionAttribute` class. This means that at runtime, you can find extension methods using the following code:

```csharp
Assembly.GetExecutingAssembly()
    .GetTypes()
    .Where(t => Attribute.IsDefined(t, typeof(ExtensionAttribute)))
```

## Conclusion
Of these advanced techniques, the one that is probably most useful is the graceful null-handling. It has the potential of making your code less prone to the infamous `NullReferenceException` without much effort.