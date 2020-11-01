---
title: Timing code using computation expressions
date: 2020-09-28
tags:
  - F#
  - Computation expressions
---

While building a link checking tool, I wanted to log the time it took to check a link. The go-to class to measure elapsed time in .NET applications is the [`Stopwatch` class](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.stopwatch?view=netcore-3.1). The first implementation looked like this:

```fsharp
// Create the stopwatch and immediately start measuring time
let stopwatch = System.Diagnostics.Stopwatch.StartNew()

// Execute the function we want to time
let linkStatus = checkLink()

// Stop measuring time
stopwatch.stop()

// Output the measured (elapsed) time (e.g. 00:00:12.9415700)
printfn "%A" stopwatch.Elapsed
```

Nice and simple.

To allow re-using the link timing functionality, we can create a new function that returns the link check result and the elapsed time as a tuple:

```fsharp
let checkLinkWithTiming() =
  let stopwatch = System.Diagnostics.Stopwatch.StartNew()
  let linkStatus = checkLink()
  stopwatch.stop()

  linkStatus, stopwatch.Elapsed

let linkStatus, elapsedTime = checkLinkWithTiming()
printfn "%A" elapsedTime
```

## Generalizing

Some time later, I found that I wanted to time other functions too. We can generalize the `checkLinkWithTiming` function to work with other functions too by taking the function to time as a parameter:

```fsharp
let executeAndTime func =
  let stopwatch = System.Diagnostics.Stopwatch.StartNew()

  // Call the `func` parameter (which is a function)
  let funcResult = func()

  stopwatch.stop()

  funcResult, stopwatch.Elapsed
```

We can then pass in the `checkLink()` function as an argument to get its result and its timing information:

```fsharp
let linkStatus, elapsedTime = executeAndTime checkLink
printfn "%A" elapsedTime
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
  stopwatch.stop()

  Timed(Result = funcResult, Elapsed = stopwatch.Elapsed)

let timedLinkCheckResult = executeAndTime checkLink
printfn "%A" timedLinkCheckResult.Elapsed
```

TODO: explain why this is better

## Timing asynchronous functions

TODO: add asynchronous timing

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
