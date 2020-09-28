---
title: Timing code using computation expressions
date: 2020-09-28
tags:
  - F#
  - Computation expressions
---

# Introduction

While building a [Markdown link checker in F#](https://github.com/erikschierboom/markdownlinkchecker), I wanted to display the time it took to check a specific link. The go-to class to measure elapsed time in .NET applications is the [`Stopwatch` class](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.stopwatch?view=netcore-3.1).

We can time our `checkLink()` function as follows:

```fsharp
open System.Diagnostics

// Create the stopwatch and start measuring time
let stopwatch = Stopwatch.StartNew()

let checkedStatus = checkLinkStatus()

// Stop measuring time
stopwatch.stop()

// Output the measured (elapsed) time (e.g. 00:00:12.9415700)
printfn "%A" stopwatch.Elapsed
```

Nice and simple. At some point though, I wanted to output the timing information somewhere else in my application, which meant that the timing information had to be returned from the function. We could return a simple tuple of type `'T * TimeSpan`

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
