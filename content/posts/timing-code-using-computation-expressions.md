---
title: Timing code using computation expressions
date: 2020-12-10
tags:
  - fsharp
  - computation-expressions
---

While building a link checking tool, I wanted to log the time it took to check a link. For demonstration purposes, we'll define the link checking function as follows:

```fsharp
let linkIsValid () =
  // Simulate link checking by sleeping for a second
  System.Threading.Thread.Sleep(TimeSpan.FromSeconds(1.))

  true
```

To measure this function's execution time, we'll use the [`Stopwatch` class](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.stopwatch?view=netcore-3.1), which is designed specifically for this purpose:

```fsharp
// This namespace contains the Stopwatch class
open System.Diagnostics

// Create the stopwatch and immediately start measuring time
let stopwatch = Stopwatch.StartNew()

// Execute the function we want to time
let isValid = linkIsValid()

// Stop measuring time
stopwatch.Stop()

// Output the measured (elapsed) time
printfn "%A" stopwatch.Elapsed
```

If we run this code, the output will look like this:

```
00:00:01.0155117
```

To allow re-using the link timing functionality, we'll encapsulate the above code in a function which returns the link check result _and_ the elapsed time as a tuple:

```fsharp
let linkIsValidWithTiming() =
  let stopwatch = Stopwatch.StartNew()
  let linkStatus = linkIsValid()
  stopwatch.Stop()

  // Return the link status _and_ the elapsed time
  linkStatus, stopwatch.Elapsed

let isValid, elapsed = linkIsValidWithTiming()
printfn "%A" elapsed
```

## Generalizing

What if we want to mesure the execution time of other functions? Well, we can generalize the `linkIsValidWithTiming` function to work with other functions by taking the function to time as a parameter:

```fsharp
let executeAndTime func =
  let stopwatch = Stopwatch.StartNew()

  // Call the `func` parameter (which is a function)
  let funcResult = func()

  stopwatch.Stop()

  funcResult, stopwatch.Elapsed
```

We can then pass in the `linkIsValid` function as an argument to get its result _and_ its timing information:

```fsharp
let isValid, elapsed = executeAndTime linkIsValid
printfn "%A" elapsed
```

## Timing type

To make our code more explicit, let's define a type to hold timing results:

```fsharp
type Timed<'T> =
  { Result: 'T
    Elapsed: TimeSpan }
```

We can then return this type from our `executeAndTime` function:

```fsharp
let executeAndTime func =
  let stopwatch = Stopwatch.StartNew()
  let result = func()
  stopwatch.Stop()

  { Result = result; Elapsed = stopwatch.Elapsed }

let timedResult = executeAndTime linkIsValid
printfn "%A" timedResult.Elapsed
```

To me, this version has better _motivational transparency_ [^1] as it makes it more explicit what the code is doing.

## Timing asynchronous functions

To allow for non-blocking link checking, let's define an asynchronous version of the `linkIsValid` function:

```fsharp
let linkIsValidAsync () = async {
  do! Async.Sleep(1000)
  return true
}
```

If we time this function, the elapsed time seems to be off:

```
00:00:00.0010091
```

As the `linkIsValidAsync` function should run for _at least_ 1000 milliseconds, this can't be right. The problem is that we don't wait for the async function to complete. This means that we're not timing the execution of the function from begin to end, but only the time it takes to start executing. To ensure that the asynchronous function call blocks until it has completed, we can pipe it into `Async.RunSynchronously`:

```fsharp
let executeAndTime func =
  let stopwatch = Stopwatch.StartNew()
  let funcResult = func() |> Async.RunSynchronously
  stopwatch.Stop()

  { Result = funcResult; Elapsed = stopwatch.Elapsed }
```

Our asynchronous function is now timed correctly:

```
00:00:01.0390756
```

However, if we try to time the old, synchronous `linkIsValid` function, we get a compile error:

```
[FS0001] Type mismatch. Expecting a
  'unit -> Async<'a>'
but given a
  'unit -> bool'
The type 'Async<'a>' does not match the type 'bool'
```

The compiler informs us that `Async.RunSynchronously` expects an `Async<'a>` argument, but that it received a `bool` argument, which is indeed what the (synchronous) `linkIsValid` function returns.

To fix this, let's revert back to our previous, synchronous version of the `executeAndTime` function (by removing `Async.RunSynchronously`). We can then pass in a lambda as the argument where we pipe the `linkIsValidAsync` result to `Async.RunSynchronously`:

```fsharp
let timedResult = executeAndTime (fun() -> linkIsValidAsync() |> Async.RunSynchronously)
```

Using this approach, we can time both synchronous and asynchronous functions, without the `executeAndTime` function having to be aware of the difference. The output confirms that we correctly time our asynchronous function:

```
00:00:01.0390756
```

## Computation Expression

While the function-based approach works well, wouldn't it be great if we could this instead?

```fsharp
let timedResult = timed {
  return linkIsValid()
}
```

This code looks very similar to the `async` syntax we used before in our `linkIsValidAsync` function:

```fsharp
let linkIsValidAsync () = async {
  do! Async.Sleep(1000)
  return true
}
```

As it turns out, the `async` syntax is provided by an F# feature called [computation expressions](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions), which provides a convenient syntax for computations that can be sequenced and combined. Besides the [built-in computation expressions](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions#built-in-computation-expressions) (like `async`), one can also define custom computation expressions, so let's try and build a `timed` computation expression.

Computation expressions are implemented as classes, which are known as _builder_ types. To get the nice syntax we saw above, one has to bind an instance of the builder type to an identifier. The name of the identifier determines how one can use the computation expression in code.

```fsharp
// Empty build class
type TimedBuilder() = class end

// Builder class instance. The name determines how to use the computation expression
let timed = TimedBuilder()
```

This will allow us to do:

```fsharp
let timedResult = timed {}
```

If we try to compile this code, we get can error:

```
[FS0003] This value is not a function and cannot be applied.
```

While the message is slightly cryptic, it tries to tell us that we're not doing anything in our computation expression. This starts to make sense if you realize that computation expressions are translated to method invocations on the builder type instance (in our case: `timed`). Let's try to mimic the `async` computation expression and return the result of calling the `linkIsValid` function:

```fsharp
let timedResult = timed {
  return linkIsValid()
}
```

This time, we get the following compile error:

```
[FS0708] This control construct may only be used if the computation expression builder defines a 'Return' method
```

This is actually very descriptive! It tells us that to be able to use the `return` keyword inside our custom computation expression, its builder (type) needs to define a `Return` method. If we [check the documentation](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions#creating-a-new-type-of-computation-expression), the `Return` method should be of type `'T -> M<'T>`. You should read this as: it takes a "regular" type and returns a computation expression-specific type that "wraps" the "regular" type. For our computation expression, our "wrapped" type is the `Timed<'T>` type. Let's add the `Return` member to the `TimedBuilder` class:

```fsharp
type TimedBuilder() =
  member x.Return(value) = { Result = value; Elapsed = TimeSpan.Zero }
```

We can then use this as follows:

```fsharp
let timedResult = timed {
  return linkIsValid()
}
printfn "%A" timedResult.Elapsed
```

This time, the code runs successfully and outputs:

```
00:00:00
```

Let's try and time the function execution:

```fsharp
type TimedBuilder() =
  member x.Return(value) =
    let stopwatch = Stopwatch.StartNew()
    let result = value()
    stopwatch.Stop()

    { Result = result; Elapsed = stopwatch.Elapsed }
```

As with our `executeAndTime` function, the `Return` method now expects its parameter to be a function, which means that the computation expression should not return the result of invoking the `linkIsValid` function, but return the function itself:

```fsharp
let timedResult = timed {
  return linkIsValid
}
```

And this works! Nice and clean, right?

### Improved function handling

Our current implementation is not idiomatic though. When executing functions in a computation expression, the `Delay` member should be implemented. Its signature is `(unit -> M<'T>) -> M<'T>`, which means that it takes a function without parameters that returns the "wrapped" type and also returns the "wrapped" type. Let's implement this method:

```fsharp
type TimedBuilder() =
  member x.Delay(func) =
    let stopwatch = Stopwatch.StartNew()
    let timedResult = func()
    stopwatch.Stop()

    { timedResult with Elapsed = timedResult.Elapsed + stopwatch.Elapsed }
```

The time measuring code is the same, the biggest change is that when we invoke the `func` parameter, we receive a `Timed<'T>` value. We then deconstruct this value and return a new `Timed<'T>` instance, but with the elapsed time as measured by the stopwatch added to the elapsed time returned by the function. If we try to compile this, we get:

```
[FS0708] This control construct may only be used if the computation expression builder defines a 'Return' method
```

Whoops. We're still using the `return` keyword in our computation expression invocation, which means that we still have to implement the `Return` method. Note though that any value passed to the `Return` method is automatically wrapped in a "delay" function and passed to the `Delay` method to execute. We can use our original `Return` method implementation:

```fsharp
type TimedBuilder() =
  member x.Return(value) = { Result = value; Elapsed = TimeSpan.Zero }

  member x.Delay(func) =
    let stopwatch = Stopwatch.StartNew()
    let timedResult = func()
    stopwatch.Stop()

    { timedResult with Elapsed = timedResult.Elapsed + stopwatch.Elapsed }
```

We'll have to change our computation expression to actually invoke the `linkIsValid` function again:

```fsharp
let timedResult =
  timed {
    return linkIsValid()
  }
```

And we're back to things working. Note that we can now also return a constant value from our computation expression (although that admittedly doesn't make much sense):

```fsharp
let timedResult =
  timed {
    return 2
  }
```

And with that, we have created a computation expression that elegantly allows us to measure function execution time using a nice and concise syntax.

# Conclusion

Computation expressions are a great way to help make your code more expressive. While writing a computation expression takes a little getting used to, they are not that hard once you're familiar with their implementation rules. The compiler is also quite helpful, outputting detailed error message that help pinpoint what methods to implement.

[^1]: [Stylish F# by Kit Eason](https://www.apress.com/gp/book/9781484239995).
