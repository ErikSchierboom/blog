---
title: "Fluent interfaces"
date: 2014-10-08
tags:
  - csharp
  - dotnet
  - fluent-interfaces
---

Fluent interfaces are software API's designed to be readable and to _flow_. They are like a miniature [domain-specific language](http://en.wikipedia.org/wiki/Domain-specific_language), code structured for one specific purpose.

## Flow

Methods in fluent interfaces can often be chained, creating a _flow_ of method calls. To allow a method to be chained, there is but one rule:

> The method must return a non-void value.

As the return type of a method determines what method can be called next, it determines its flow.

Consider the following, non-flowing method:

```csharp
public class NonFlowing
{
    public void Log(string str)
    {
       ...
    }
}
```

As the `Log()` method returns `void`, there can be no chain of method calls:

```csharp
NonFlowing nonFlowing = new NonFlowing();

// Error: fails to compile
nonFlowing.Log("line 1")
          .Log("line 2")
          .Log("line 3");
```

We can remedy this by returning a non-void type:

```csharp
public class Flowing
{
    public Flowing Log(string str)
    {
       ...

       return this;
    }
}
```

This `Log()` method returns a `Flowing` instance, which methods we can call when `Log()` has finished. We can now chain calls to the `Log()` method:

```csharp
Flowing flowing = new Flowing();

flowing.Log("line 1")
       .Log("line 2")
       .Log("line 3");
```

If we change the `Log()` method's return type, the flow changes to that type:

```csharp
public class FlowingContext
{
    public string Log(string str)
    {
       ...

       return str;
    }
}

FlowingContext flowing = new FlowingContext();

// Returns "LINE"
flowing.Log("line 1")
       .Substring(0, 4)
       .ToUpper();
```

### More complex flows

Some API's require a more complex flow, for example to build-up a complex structure. An example of this is a SQL query. A minimal SQL query has two required parts:

- The `SELECT` statement, indicating which column values to return.
- The `FROM` statement, specifying the tables from which the results are returned.

However, an SQL query also has two other, optional parts:

- The `WHERE` statement, which determines the filtering to apply.
- The `ORDER BY` statement, which specifies how to order the results.

A basic fluent API designed to meet these criteria could look like this:

```csharp
public class BasicQuery
{
    public BasicQuery Select(string selectSql)
    {
        ...
    }

    public BasicQuery From(string fromSql)
    {
        ...
    }

    public BasicQuery Where(string whereSql)
    {
        ...
    }

    public BasicQuery OrderBy(string orderBySql)
    {
        ...
    }

    public dynamic Execute()
    {
        ...
    }
}
```

We can now use this API as follows:

```csharp
new BasicQuery().Select("*")
                .From("users")
                .Where("name = 'Shaq'")
                .OrderBy("age")
                .Execute();
```

Although this works fine, we can still use this API in strange and incorrect ways:

```csharp
// 1. Weird order
new BasicQuery().OrderBy("age")
                .From("users")
                .Select("*")
                .Execute();

// 2. Missing required SELECT and FROM parts
new BasicQuery().Where("name = 'Shaq'")
                .OrderBy("age")
                .Execute();
```

What we want is two things:

1. Force the method calls order to: `Select()`, `From()`, `Where()` and `OrderBy()`.
2. Only allow a query to be executed when, at least, `Select()` and `From()` were called.

We can visualize these requirements in a state diagram:

![Fluent interface state diagram](/images/posts/fluent-interfaces-states.png)

Let's start with the first requirement. We can force the order by creating an interface for each state, each of which has a method to move to the next state:

```csharp
public interface ISelectQuery
{
    IFromQuery From(string fromSql);
}

public interface IFromQuery
{
    IWhereQuery Where(string whereSql);
    IOrderByQuery OrderBy(string orderBySql);
}

public interface IWhereQuery
{
    IOrderByQuery OrderBy(string orderBySql);
}

public interface IOrderByQuery
{
}
```

If we start with an `ISelectQuery` instance, the method calls are always in our desired order:

```csharp
ISelectQuery selectQuery;
IFromQuery fromQuery = selectQuery.From("users");
IWhereQuery whereQuery = fromQuery.Where("name = 'Shaq'");
IOrderByQuery orderByQuery = whereQuery.OrderBy("age");
```

And only valid state transitions are allowed:

```csharp
// Compiler error
selectQuery.Where("name = 'Shaq'");
selectQuery.OrderBy("age");
```

We can solve our second requirement by adding an interface for an executable query. We add this interface to each executable state's interface:

```csharp
public interface IExecutableQuery
{
    dynamic Execute();
}

public interface ISelectQuery
{
    IFromQuery From(string fromSql);
}

public interface IFromQuery : IExecutableQuery
{
    IWhereQuery Where(string whereSql);
    IOrderByQuery OrderBy(string orderBySql);
}

public interface IWhereQuery : IExecutableQuery
{
    IOrderByQuery OrderBy(string orderBySql);
}

public interface IOrderByQuery : IExecutableQuery
{
}
```

We have now ensured that all states except the `SELECT` state can be executed:

```csharp
// Valid calls
fromQuery.Execute();
whereQuery.Execute();
orderByQuery.Execute();

// Compiler error
selectQuery.Execute();
```

Ignoring the implementation of the state interfaces, we can create a class `Query` to start the flow with:

```csharp
public static class Query
{
    public static ISelectQuery Select(string selectSql)
    {
        ...
    }
}
```

Our API is now done and can be called like this:

```csharp
Query.Select("*")
     .From("users")
     .Where("name = 'Shaq'")
     .OrderBy("age")
     .Execute();
```

## Readability

When designing fluent interfaces, it is very important to design for readability. Here are some examples.

### Getters and setters

Take the following class with two flowing _setter_ methods:

```csharp
public class Player
{
    public Player SetName(string name) { ... }
    public Player SetAge(int age) { ... }
}
```

If we want to create a new `Player` instance and set its name and age, we can do that as follows:

```csharp
Player player = new Player();
player.SetName("LeBron James")
      .SetAge(27);
```

The setters allow for a nice flow, but consider the following alternative:

```csharp
public class AlternativePlayer
{
    public AlternativePlayer WithName(string name) { ... }
    public AlternativePlayer WithAge(int age) { ... }
}
```

We can now create a new instance as follows:

```csharp
AlternativePlayer player = new AlternativePlayer().WithName("LeBron James")
                                                  .WithAge(27);
```

To me, this code has slightly better readability as its reads more like a sentence than the previous example. However, it comes at the cost of reduced discoverability as setter methods are usually prefixed with "Set".

Another alternative would be to use a factory method to create new instances:

```csharp
public class FactoryPlayer
{
    private FactoryPlayer() { }

    public FactoryPlayer WithName(string name) { ... }
    public FactoryPlayer WithAge(string age) { ... }

    public static FactoryPlayer Create() { return new FactoryPlayer(); }
}
```

We can now create a new instance as follows:

```csharp
FactoryPlayer player = FactoryPlayer.Create()
                                    .WithName("LeBron James")
                                    .WithAge(27);
```

In C#, we would probably use properties instead of setters and getters:

```csharp
public class PropertiesPlayer
{
    public string Name { get; set; }
    public int Age { get; set; }
}

PropertiesPlayer player = new PropertiesPlayer { Name = "LeBron James", Age = 27 };
```

This looks good from a readability viewpoint, but limits your flow options as the type returned from the getters and setters cannot be different from the property type.

### Higher level functions

Sometimes, a class has several lower level properties or functions that are often modified together. In those cases, readability can be improved by providing higher level abstractions. Take the following example:

```csharp
public class Window
{
    public bool IsTranslucent { get; set; }
    public float OpacityPercentage { get; set; }
}
```

Suppose we support two use cases: setting the window to be opaque (opacity percentage equal to 100) and setting the window to be translucent (opacity percentage less than 100). This can be done as follows:

```csharp
Window window = new Window();

// 1. opaque
window.IsTranslucent = false;
window.OpacityPercentage = 100;

// 2. translucent
window.IsTranslucent = true;
window.OpacityPercentage = 50;
```

Although this code seem pretty straightforward, it is easy to make the following mistake:

```csharp
Window window = new Window();
window.IsTranslucent = true;

// Forgot to set the opacity percentage!
```

The following example does a much better job of explaining what the class can do:

```csharp
public class Window
{
    public bool IsTransparent { get; private set; }
    public float Opacity { get; private set; }

    public void MakeOpaque()
    {
        this.IsTransparent = false;
        this.Opacity = 100;
    }

    public void MakeTransparent(float opacity)
    {
        this.IsTransparent = true;
        this.Opacity = opacity;
    }
}
```

This class shows its capabilities better and prevents the mistake mentioned earlier:

```csharp
Window window = new Window();

// 1. opaque
window.MakeOpaque();

// 2. translucent
window.MakeTransparent(50);
```

### Pre-populated fields

In some cases, you can expect your code to be often called with a specific set of parameters. In this case, it can be convenient to provide a pre-populated instance with that set of parameters.

Consider the following class to filter a population:

```csharp
public class PopulationFilter
{
    public int MinimumAge { get; set; }
    public int MaximumAge { get; set; }
}
```

If you know that users of your API will often filter out [toddlers](http://en.wikipedia.org/wiki/Toddler), you could add a pre-populated field:

```csharp
public class PopulationFilter
{
    public static PopulationFilter Toddlers = new PopulationFilter
                                                    {
                                                        MinimumAge = 1,
                                                        MaximumAge = 2
                                                    };

    public int MinimumAge { get; set; }
    public int MaximumAge { get; set; }
}
```

Users can now easily filter on toddlers using `PopulationFilter.Toddlers`.

These examples show that simple refactorings can greatly influence readability. When developing a fluent API, one should try different API designs to find the one with the best readability.

## Creating a fluent API

With all that we now know about designing fluent API's, let's try to create one from scratch. Our goal is to create an image processor that can do some basic image transformations and save the result to a file.

Often, it is best to start with writing code how you'd _want_ to be calling the fluent API. Experiment with different options to see what works best.

Say we came up with this design:

```csharp
ImageProcessor.FromFile(@"c:\input.png")
              .Rotate(Degrees.FromAngle(90))
              .Save(@"c:\output1.png");

ImageProcessor.FromFile(@"c:\input.png")
              .Scale(Scale.Half)
              .Save(@"c:\output2.png");

ImageProcessor.FromImage(Image.FromFile(@"c:\input.png"))
              .Scale(Scale.FromPercentage(75))
              .Rotate(Degrees.FromAngle(45))
              .Save(@"c:\output3.png");
```

Even if you would not know the API that is used, you'd probably have a good idea of what this code does. Let start building it!

In our API design, we opted for factory methods instead of constructors, resulting in this class:

```csharp
public class ImageProcessor
{
    private readonly Image image;

    private ImageProcessor(Image image)
    {
        this.image = image;
    }

    public static ImageProcessor FromFile(string fileName)
    {
        return new ImageProcessor(Image.FromFile(fileName));
    }

    public static ImageProcessor FromImage(Image image)
    {
        return new ImageProcessor(image);
    }
}
```

This allows us to do:

```csharp
var processorFromFile = ImageProcessor.FromFile(@"c:\input.png");
var processorFromImage = ImageProcessor.FromImage(new Bitmap(100, 100));
```

The next step is to add the transformation methods:

```csharp
public class ImageProcessor
{
    ...

    public ImageProcessor Rotate(Degrees degrees)
    {
        using (var graphics = Graphics.FromImage(this.image))
        {
            graphics.RotateTransform(degrees.Angle);
        }

        return this;
    }

    public ImageProcessor Scale(Scale scale)
    {
        using (var graphics = Graphics.FromImage(this.image))
        {
            graphics.ScaleTransform(scale.ScaleFactorX, scale.ScaleFactorY);
        }

        return this;
    }
}
```

The implementation of these methods is straightforward, but now we need to define the `Degrees` and `Scale` classes. If we look back at our API design, the `Degrees` instance is created as follows:

```csharp
Degrees.FromAngle(90)
```

Once again, can define a factory method to create new instances:

```csharp
public class Degrees
{
    private Degrees(float angle)
    {
        this.Angle = angle;
    }

    public float Angle { get; private set; }

    public static Degrees FromAngle(float angle)
    {
        return new Degrees(angle);
    }
}
```

The `Scale` class is similar:

```csharp
public class Scale
{
    private Scale(float scaleFactorX, float scaleFactorY)
    {
        this.ScaleFactorX = scaleFactorX;
        this.ScaleFactorY = scaleFactorY;
    }

    public float ScaleFactorX { get; private set; }
    public float ScaleFactorY { get; private set; }

    public static Scale FromPercentage(float percentage)
    {
        return new Scale(percentage / 100.0f,  percentage / 100.0f);
    }
}
```

In our API design, we used `Scale.Half` as a shortcut for scaling images to half their size. For this, we add a public (readonly) instance of the `Scale` class:

```csharp
public class Scale
{
    public static readonly Scale Half = Scale.FromPercentage(50.0f);

    ...
}
```

The final step is to add the `Save()` method, which is the logical end of the image processing:

```csharp
public class ImageProcessor
{
    ...

    public void Save(string fileName)
    {
        this.image.Save(fileName);
    }
}
```

As this method should be the last method called in a method chain, it returns `void`.

The final fluent API implementation is as follows:

```csharp
public class ImageProcessor
{
    private readonly Image image;

    private ImageProcessor(Image image)
    {
        this.image = image;
    }

    public static ImageProcessor FromFile(string fileName)
    {
        return new ImageProcessor(Image.FromFile(fileName));
    }

    public static ImageProcessor FromImage(Image image)
    {
        return new ImageProcessor(image);
    }

    public ImageProcessor Rotate(Degrees degrees)
    {
        using (var graphics = Graphics.FromImage(this.image))
        {
            graphics.RotateTransform(degrees.Angle);
        }

        return this;
    }

    public ImageProcessor Scale(Scale scale)
    {
        using (var graphics = Graphics.FromImage(this.image))
        {
            graphics.ScaleTransform(scale.ScaleFactorX, scale.ScaleFactorY);
        }

        return this;
    }

    public void Save(string fileName)
    {
        this.image.Save(fileName);
    }
}

public class Degrees
{
    private Degrees(float angle)
    {
        this.Angle = angle;
    }

    public float Angle { get; private set; }

    public static Degrees FromAngle(float angle)
    {
        return new Degrees(angle);
    }
}

public class Scale
{
    public static readonly Scale Half = Scale.FromPercentage(50.0f);

    private Scale(float scaleFactorX, float scaleFactorY)
    {
        this.ScaleFactorX = scaleFactorX;
        this.ScaleFactorY = scaleFactorY;
    }

    public float ScaleFactorX { get; private set; }
    public float ScaleFactorY { get; private set; }

    public static Scale FromPercentage(float percentage)
    {
        return new Scale(percentage / 100.0f,  percentage / 100.0f);
    }
}
```

### Code complexity

The amount of code in this example shows that fluent API's often need quite a lot of code. This is in part due to us implementing the API in C#. The Objective-C language, for example, requires parameter names to always be explicitly passed. This is ideal for fluent API's, as function calls are thus more explicit and you have less need of factory methods to make things explicit.

In C#, you can also pass parameters by name. If we were to take advantage of this feature, we could rewrite our `Rotate()` method to:

```csharp
public ImageProcessor Rotate(float angle)
{
    using (var graphics = Graphics.FromImage(this.image))
    {
        graphics.RotateTransform(angle);
    }

    return this;
}
```

We can then call this method as follows:

```csharp
ImageProcessor.FromFile(@"c:\input.png")
              .Rotate(angle: 90)
              .Save(@"c:\output1.png");
```

This example is just as clear as the one with the `Degrees` class, but with less code. Unfortunately, it is also possible to call `Rotate()` without its parameter name, which greatly lessens its readability:

```csharp
ImageProcessor.FromFile(@"c:\input.png")
              .Rotate(90)
              .Save(@"c:\output1.png");
```

The design of a fluent API is thus also influenced by the language used.

## Examples

If you are designing a fluent API, it is useful to look at existing fluent API designs. Here are some good examples:

#### jQuery

```csharp
$("#p1").css("color", "red")
        .slideUp(2000)
        .slideDown(2000);
```

#### LINQ (.NET)

```csharp
Enumerable.Range(1, 5)
          .Where(n => n % 2 == 1)
          .OrderByDescending(n => n)
          .ToArray();
```

#### EasyMock (Java):

```java
expect(mock.voteForRemoval("Document"))
  .andReturn(42).times(3)
  .andThrow(new RuntimeException(), 4)
  .andReturn(-42);
```

## Conclusion

Designing a good fluent API is not as easy as simply returning `this` from your methods. Readability is a key aspect of fluent API's and that requires you to pay attention to how you name and structure your code. Implementing a fluent API will also most likely increase the complexity of your code. However, when properly designed, fluent API's have great discoverability and improve the readability of the code.
