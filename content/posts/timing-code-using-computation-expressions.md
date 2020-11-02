---
title: Timing code using computation expressions
date: 2020-09-28
tags:
  - F#
  - Computation expressions
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
// Create the stopwatch and immediately start measuring time
let stopwatch = System.Diagnostics.Stopwatch.StartNew()

// Execute the function we want to time
let isValid = linkIsValid()

// Stop measuring time
stopwatch.Stop()

// Output the measured (elapsed) time (e.g. 00:00:01.0053087)
printfn "%A" stopwatch.Elapsed
```

Nice and simple.

To allow re-using the link timing functionality, let's create a function that returns the link check result _and_ the elapsed time as a tuple:

```fsharp
let linkIsValidWithTiming() =
  let stopwatch = System.Diagnostics.Stopwatch.StartNew()
  let linkStatus = linkIsValid()
  stopwatch.Stop()

  linkStatus, stopwatch.Elapsed

let isValid, elapsed = linkIsValidWithTiming()
printfn "%A" elapsed
```

## Generalizing

But what if we want to time other functions too? Well, we can generalize the `linkIsValidWithTiming` function to work with other functions by taking the function to time as a parameter:

```fsharp
let executeAndTime func =
  let stopwatch = System.Diagnostics.Stopwatch.StartNew()

  // Call the `func` parameter (which is a function)
  let funcResult = func()

  stopwatch.Stop()

  funcResult, stopwatch.Elapsed
```

We can then pass in the `linkIsValid` function as an argument to get its result and its timing information:

```fsharp
let isValid, elapsed = executeAndTime linkIsValid
printfn "%A" elapsed
```

## Timing type

To make our code more explicit, let's define a type to hold timing results:

```fsharp
type Timed<'T> = Timed of Result: 'T * Elapsed: TimeSpan
```

We can then return this type from our `executeAndTime` function:

```fsharp
let executeAndTime func =
  let stopwatch = System.Diagnostics.Stopwatch.StartNew()
  let funcResult = func()
  stopwatch.Stop()

  Timed(Result = funcResult, Elapsed = stopwatch.Elapsed)

let (Timed(Elapsed = elapsed)) = executeAndTime linkIsValid
printfn "%A" elapsed
```

To me, this version has better _motivational transparency_ [^1] as it makes it more explicit what the code is doing.

## Timing asynchronous functions

To make the link checking more efficient, let's create an asynchronous version of the `linkIsValid` function:

```fsharp
let linkIsValidAsync () = async {
    do! Async.Sleep(1000)
    return true
}
```

If we time this function though, the elapsed time seems to be off:

```
00:00:00.0010091
```

As the `linkIsValidAsync` function should run for _at least_ 1000 milliseconds, this can't be right. The problem is that we don't wait for the async function to complete. This means that we're not timing the execution of the function from begin to end, but only the time it takes to start executing. To ensure that the asynchronous function call blocks until it has completed, we can pipe it into `Async.RunSynchronously`:

```fsharp
let executeAndTime func =
    let stopwatch = System.Diagnostics.Stopwatch.StartNew()
    let funcResult = func() |> Async.RunSynchronously
    stopwatch.Stop()

    Timed(Result = funcResult, Elapsed = stopwatch.Elapsed)
```

Our asynchronous function is now timed correctly:

```
00:00:01.0390756
```

However, if we try to time the old, synchronous `linkIsValid` function, we get a compile error:

```
Program.fs(29, 53): [FS0001] Type mismatch. Expecting a
  'unit -> Async<'a>'
but given a
  'unit -> bool'
The type 'Async<'a>' does not match the type 'bool'
```

The compiler informs us that `Async.RunSynchronously` expects an `Async<'a>` argument, but that it received a `bool` argument, which is indeed what the (synchronous) `linkIsValid` function returns.

To fix this, let's revert back to our previous, synchronous version of the `executeAndTime` function (by removing `Async.RunSynchronously`). We can then pass in a lambda as the argument where we pipe the `linkIsValidAsync` result to `Async.RunSynchronously`:

```fsharp
let (Timed(Elapsed = elapsed)) = executeAndTime (fun() -> linkIsValidAsync() |> Async.RunSynchronously)
```

Using this approach, we can time both synchronous and asynchronous functions, without the `executeAndTime` function having to be aware of the difference. The output confirms that we correctly time our asynchronous function:

```
00:00:01.0390756
```

## Computation Expression

While the function-based approach works well, wouldn't it be great if we could this instead?

```fsharp
let (Timed(Elapsed = elapsed)) = timed {
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

As it turns out, the `async` syntax is provided by an F# feature called [computation expressions](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions), which provide a convenient syntax for computations that can be sequenced and combined. Besides the built-in computation expressions (like `async`), one can also define custom computation expressions, so let's try and build a `timed` computation expression.

Computation expressions are implemented as classes, which are known as _builder_ types. To get the nice syntax we saw above, one has to bind an instance of the builder type to an identifier. It is the name of the identifier that determines how to call the computation expression in your code.

Let's start building our computation expression:

```fsharp
// Empty build class
type TimedBuilder() = class end

// Builder class instance
let timed = TimedBuilder()
```

This will allow us to do:

```fsharp
let _ = timed {}
```

If we try to compile this code, we get can error:

```
Program.fs(45, 23): [FS0003] This value is not a function and cannot be applied.
```

While the message is slightly cryptic, it tries to tell us that we're not doing anything in our computation expression. This starts to make sense if you realize that computation expressions are translated to method invocations on the builder type instance (in our case: `timed`). Let's try to mimic the `async` computation expression and return the result of calling the `linkIsValid` function:

```fsharp
let _ = timed {
    return linkIsValid()
}
```

This time, we get the following compile error:

```
Program.fs(46, 9): [FS0708] This control construct may only be used if the computation expression builder defines a 'Return' method
```

This is actually very descriptive! It tells us that to be able to use the `return` keyword inside our custom computation expression, its builder (type) needs to define a `Return` method. If we [check the documentation](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions#creating-a-new-type-of-computation-expression), the `Return` method should be of type `'T -> M<'T>`. You should read this as: it takes a "regular" type and returns a computation expression-specific type that "wraps" the "regular" type. For our computation expression, our "wrapped" type is the `Timed<'T>` type. Let's add the `Return` member to the `TimedBuilder` class:

```fsharp
type TimedBuilder() =
   member x.Return(value) = Timed(Result = value, Elapsed = TimeSpan.Zero)
```

We can then use this as follows:

```fsharp
let (Timed(Elapsed = elapsed)) = timed {
    return linkIsValid()
}
printfn "%A" elapsed
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
       Timed(Result = result, Elapsed = stopwatch.Elapsed)
```

As with our `executeAndTime` function, we expect the value that is passed in to be a function, which means that we can use our `timed` computation expression as follows:

```fsharp
let (Timed(Elapsed = elapsed)) = timed {
    return linkIsValid
}
printfn "%A" elapsed
```

And this works!

```
let (Timed(Elapsed = elapsed)) =
        timed {
            return linkIsValid()
        }
    printfn "%A" elapsed

    let (Timed(Elapsed = elapsed)) =
        timed {
            return async {
                return! linkIsValidAsync()
            } |> Async.RunSynchronously
        }
    printfn "%A" elapsed
```

```fsharp
type TimedBuilder() =
   member x.Return(value) = Timed(Result = value, Elapsed = TimeSpan.Zero)

   member x.Delay(func) =
       let stopwatch = Stopwatch.StartNew()
       let delayedResult = func()
       stopwatch.Stop()

       match delayedResult with
       | Timed(Result = result; Elapsed = elapsed) -> Timed(Result = result, Elapsed = elapsed + stopwatch.Elapsed)
```

# Conclusion

TODO

[^1]: [Stylish F# by Kit Eason](https://www.apress.com/gp/book/9781484239995).
