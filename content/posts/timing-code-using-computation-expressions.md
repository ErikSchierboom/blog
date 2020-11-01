---
title: Timing code using computation expressions
date: 2020-09-28
tags:
  - F#
  - Computation expressions
---

While building a link checking tool, I wanted to log the time it took to check a link. To measure the execution time a code in a .NET application, we can use [`Stopwatch` class](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.stopwatch?view=netcore-3.1), which is designed specifically for this purpose.

For demonstration purposes, the link checking function is defined as follows:

```fsharp
let linkIsValid () =
    // Simulate link checking by sleeping for a second
    System.Threading.Thread.Sleep(TimeSpan.FromSeconds(1.))

    true
```

We can use the `Stopwatch` class to time how long this function takes to execute:

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

To allow re-using the link timing functionality, we can create a new function that returns the link check result _and_ the elapsed time as a tuple:

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

But what if we want to time other functions? Well, we can generalize the `linkIsValidWithTiming` function to work with other functions by taking the function to time as a parameter:

```fsharp
let executeAndTime func =
  let stopwatch = System.Diagnostics.Stopwatch.StartNew()

  // Call the `func` parameter (which is a function)
  let funcResult = func()

  stopwatch.Stop()

  funcResult, stopwatch.Elapsed
```

We can then pass in the `linkIsValid()` function as an argument to get its result and its timing information:

```fsharp
let isValid, elapsed = executeAndTime linkIsValid
printfn "%A" elapsed
```

## Timing type

To make our code more explicit, let's define a type to hold timing results:

```fsharp
type Timed<'T> = Timed of Result: 'T * Elapsed: TimeSpan
```

We can then return this type from our `executeAndTime()` function:

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

To make the link checking more efficient, let's convert the `linkIsValid()` function to be asynchronous:

```fsharp
let linkIsValid () = async {
    do! Async.Sleep(1000)
    return true
}
```

If we re-run our code, the code still works, but the elapsed time seems to be off:

```
00:00:00.0010091
```

As the function should run for _at least_ 1000 milliseconds, this can't be right. The problem is that we don't wait for the async function to complete. This means that we're not timing the execution of the function from begin to end, but only the time it takes to begin executing it.

## Computation Expression

TODO: add computation expression

```fsharp
type Timed<'T> = Timed of 'T * TimeSpan

type TimedBuilder() =
   member x.Return(value) = Timed(value, TimeSpan.Zero)

   member x.Delay(func) =
       let stopwatch = Stopwatch.StartNew()
       let delayedResult = func()
       stopwatch.Stop()

       match delayedResult with
       | Timed(value, elapsed) -> Timed(value, elapsed + stopwatch.Elapsed)

let timed = TimedBuilder()
```

# Conclusion

TODO

[^1]: [Stylish F# by Kit Eason](https://www.apress.com/gp/book/9781484239995).
