---
title: "Property-based testing"
date: 2016-02-22
tags: 
  - C#
  - Testing
  - Property-based testing
---

Writing unit tests helps verify the correctness of code. However, most unit tests only test a limited set of pre-defined input values, often just one. Testing with fixed input values is known as example-based tests.

The problem with example-based tests is that they only verify correctness for the pre-defined input values. This can easily lead to an implementation that passes the test for the pre-defined values, but fails for any other value. The implementation could thus be (largely) incorrect, even though the test passes. 

Let's demonstrate this through an example. Note that we'll use [xUnit.net](https://xunit.github.io/) as the test framework in our examples.

## Single input value

Suppose we want to write a method that calculates the [MD5 hash](https://en.wikipedia.org/wiki/MD5) of a given string. Let's start by writing a unit test:

```csharp
[Fact]
public void MD5ReturnsCorrectHash()
{
    var input = "foo";
    var md5 = MD5(input);
    var expected = "acbd18db4cc2f85cedef654fccc4a4d8";
    Assert.Equal(expected, md5);
}
```

This test method use a single input value (`"foo"`) to verify the correctness of the `MD5()` method. Therefore, this is an example-based test. To make this test pass, we can implement the `MD5()` method as follows:

```csharp
public static string MD5(string input)
{
    return "acbd18db4cc2f85cedef654fccc4a4d8";
}
```

Clearly, this implementation does **not** correctly implement the MD5 algorithm, but it passes our test! The danger of using one input value to verify correctness is that you can make the test pass by just hard-coding the expected result. 

Note that in TDD, you actually *should* write the minimal amount of code to make the test pass, so the above implementation is perfectly reasonable when doing TDD. 

In the next section, we'll see how to strengthen our test by using multiple input values.

## Multiple input values

The obvious solution to strengten tests that use one input value, is to use several input values. One way to do this is to create copies of the existing test method, but with different input values:

```csharp
[Fact]
public void MD5ForFooInputReturnsCorrectHash()
{
    var md5 = MD5("foo");
    Assert.Equal("acbd18db4cc2f85cedef654fccc4a4d8", md5);
}

[Fact]
public void MD5ForBarInputReturnsCorrectHash()
{
    var md5 = MD5("bar");
    Assert.Equal("37b51d194a7513e45b56f6524f2d51f2", md5);
}

[Fact]
public void MD5ForBazInputReturnsCorrectHash()
{
    var md5 = MD5("baz");
    Assert.Equal("73feffa4b7f6bb68e44cf984c85f6e88", md5);
}
```

Although there is nothing wrong with these three tests, we have some code duplication. Luckily, xUnit has the concept of *parameterized tests*, which allows us to define a single test with more than one input value. 

Here is the parameterized test equivalent of our previous three tests:

```csharp
[Theory]
[InlineData("foo", "acbd18db4cc2f85cedef654fccc4a4d8")]
[InlineData("bar", "37b51d194a7513e45b56f6524f2d51f2")]
[InlineData("baz", "73feffa4b7f6bb68e44cf984c85f6e88")]
public void MD5ReturnsCorrectHash(string input, string expected)
{
    var md5 = MD5(input);
    Assert.Equal(expected, md5);
}
```

This test differs from our previous tests in several ways:

- The `[Fact]` attribute is replaced with the `[Theory]` attribute. This marks the test as a parameterized test.
- The test method has two parameters: `input` and `expected`, which replace the hard-coded values in our test.
- Three `[InlineData]` attributes have been added, one for each input value/expected hash combination.

When xUnit runs this test, it will actually run it three times, with the `[InlineData]` attributes' parameters used as the `input` and `expected` parameters. Therefore, running our parameterized test results in our test being called three times with the following arguments:

```csharp
MD5ReturnsCorrectHash("foo", "acbd18db4cc2f85cedef654fccc4a4d8");
MD5ReturnsCorrectHash("bar", "37b51d194a7513e45b56f6524f2d51f2");
MD5ReturnsCorrectHash("baz", "73feffa4b7f6bb68e44cf984c85f6e88");
```

If we run our parameterized test, it will fail for the `"bar"` and `"baz"` input values. To make our test pass, we could again hard-code the expected values:

```csharp
public static string MD5(string input)
{
    if (input == "foo")
    {
        return "acbd18db4cc2f85cedef654fccc4a4d8";
    }
    if (input == "bar")
    {
        return "37b51d194a7513e45b56f6524f2d51f2";
    }
    if (input == "baz")
    {
        return "73feffa4b7f6bb68e44cf984c85f6e88";
    }
    
    return input;
}
```

With this modified implementation, the test passes for all three input values. Unfortunately, having multiple tests still allowed us to easily hard-code the implementation; it did not prevent us from incorrectly implementing the MD5 algorithm. 

So although having multiple input values leads to stronger tests, it still remains a limited set of input values. Wouldn't it be great if we could run our tests using *all* possible input values? Enter property-based testing.

## Property-based testing

In property-based testing, we take a different approach to writing our tests. Instead of testing for specific input and output values, we test the *properties* of the code we are testing. You can think of properties as rules, invariants or requirements. For example, these are some essential properties of the MD5 algorithm:

1. The hash is 32 characters long.
2. The hash only contains hexadecimal characters.
3. Equal inputs have the same hash.
4. The hash is different from its input.
5. Similar inputs have significantly different hashes.  

So how do we write tests for these properties? First, notice that these five properties are generic: they must be true for *all* possible input values. Unfortunately, writing tests that actually check all input values is not feasible, as running them would take ages. But if we can't test all input values, how can we write tests for our properties? Well, we use the next best thing: random input values. 

If you test using random input values, hard-coding the expected values is no longer possible as you don't know beforehand which input values will be tested. This forces you to write a generic implementation that works for any (valid) input value.

Let's see how that works by writing property-based tests for all five properties we just defined.

### Property 1: hash is 32 characters long

Our first property states that an MD5 hash is a string of length 32. If we would write this as a regular, example-based test, it would look like this:

```csharp
[Fact]
public void MD5ReturnsStringOfCorrectLength()
{
    var input = "foo";
    var hash = MD5(input);
    Assert.Equal(32, hash.Length);
}
```

We could strengthen our test by converting it to a parameterized test with several input values: 

```csharp
[Theory]
[InlineData("foo")]
[InlineData("bar")]
[InlineData("baz")]
public void MD5ReturnsStringOfCorrectLength(string input)
{
    var hash = MD5(input);
    Assert.Equal(32, hash.Length);
}
```

However, as said, property-based tests should work with *any* input value. A property-based test is thus a parameterized test, but with the pre-defined input values replaced by random input values. Our initial attempt at writing a property-based test might look like this:

```csharp
[Theory]
public void MD5ReturnsStringOfCorrectLength(string input)
{
    var hash = MD5(input);
    Assert.Equal(32, hash.Length);
}
```

Unfortunately, if we run this test, we'll find that xUnit reports an error for parameterized tests without input. 

To define a test that works with randomly generated input, we'll use the [FsCheck](https://fscheck.github.io/FsCheck/) property-based testing framework. We'll also install the [FsCheck.Xunit](https://www.nuget.org/packages/FsCheck.Xunit/) package, for easy integration of FsCheck with xUnit.

Having installed these libraries, we can convert our test to a property-based test by decorating it with the `[Property]` attribute:

```csharp
[Property]
public void MD5ReturnsStringOfCorrectLength(string input)
{
    var hash = MD5(input);
    Assert.Equal(32, hash.Length);
}
```

Note that we don't explicitly specify the input value(s) to use, FsCheck will (randomly) generate those. Let's run this test to see what happens:  

```
MD5ReturnsStringOfCorrectLength [FAIL]

    FsCheck.Xunit.PropertyFailedException : 
    Falsifiable, after 1 test (0 shrinks) (StdGen (766423555,296119444)):
    Original:
    ""

    ---- Assert.Equal() Failure
    Expected: 32
    Actual:   0
```

The test report indicates that our property-based test failed after one test, for the randomly generated empty string (`""`) input value. To make our empty string pass the test, we'll pad unknown inputs to a length of 32:

```csharp
public static string MD5(string input)
{
    if (input == "foo")
    {
        return "acbd18db4cc2f85cedef654fccc4a4d8";
    }
    if (input == "bar")
    {
        return "37b51d194a7513e45b56f6524f2d51f2";
    }
    if (input == "baz")
    {
        return "73feffa4b7f6bb68e44cf984c85f6e88";
    }
    
    return input.PadRight(32);
}
```

This should fix the empty string problem, so let's run the test again:

```
MD5ReturnsStringOfCorrectLength [FAIL]

    FsCheck.Xunit.PropertyFailedException : 
    Falsifiable, after 12 tests (0 shrinks) (StdGen (1087984417,296119448)):
    Original:
    <null>
        
    ---- System.NullReferenceException : Object reference not set to an instance of an object
```

Hmmm, our test still fails. This time though, the first 11 tests passed, so we made some progress. The test now failed when FsCheck generated the `null` input value. Let's fix this:

```csharp
public static string MD5(string input)
{
    if (input == "foo")
    {
        return "acbd18db4cc2f85cedef654fccc4a4d8";
    }
    if (input == "bar")
    {
        return "37b51d194a7513e45b56f6524f2d51f2";
    }
    if (input == "baz")
    {
        return "73feffa4b7f6bb68e44cf984c85f6e88";
    }
    
    return (input ?? string.Empty).PadRight(32);
}
```

And we run our test again:

```bash
MD5ReturnsStringOfCorrectLength [FAIL]

    FsCheck.Xunit.PropertyFailedException : 
    Falsifiable, after 43 tests (34 shrinks) (StdGen (964736467,296119643)):
    Original:
    "#e3n+[TC9[Jlbs,x=3U!f\~J'i u+)-y>4VLg]uA("
                
    Shrunk:
    "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"

    ---- Assert.Equal() Failure
    Expected: 32
    Actual:   33
```

Again, more progress, as it now took 43 tests before the test failed for the input value `"#e3n+[TC9[Jlbs,x=3U!f\~J'i u+)-y>4VLg]uA("`. There is something different about the test report this time though, as it mentions "34 shrinks" and a "shrunk" value, what is that about?

#### Shrinking 

In property-based testing, shrinking is used to find a minimal counter-example that proves the property does **not** hold for all possible input values. This minimal counter example is listed as the "shrunk" in our test report. Before we examine how shrinking works, try to figure out for yourself what's so special about the "shrunk" `"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"`  input value mentioned in the test report.

If you guessed that the specified "shrunk" input was special for its length, you'd be correct! The `"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"` string contains 33 characters, which significance becomes apparent if we look at the code in our implementation that processes this input value:

```csharp
return (input ?? string.Empty).PadRight(32);
```

If we use `"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"` as our input, the `PadRight(32)` call doesn't do anything as our string is already greater than or equal to 32. Therefore, the input value is returned unmodified, which means that the string that is returned also has length 33, which fails our test. 

The interesting thing about an input value of length 33 is that 33 is the *minimum* length for which the test fails; strings of length 32 or less all pass the test. As such, the listed "shrunk" value of length 33 is a minimal counter-example to our property. 

The main benefit of having a minimal counter-example is that is helps you locate precisely where your implementation breaks down (in our case for strings of length 33 and greater). You thus spend less time debugging and more writing code.

##### Finding the "shrunk" value 

So how does FsCheck find this shrunk value? To find out, we'll have FsCheck output the values it generates. Doing that is easy, we just set the `Verbose` property of our `[Property]` attribute to `true`: 

```csharp
[Property(Verbose = true)]
public void MD5ReturnsStringOfCorrectLength(string input)
{
    var hash = MD5(input);
    Assert.Equal(32, hash.Length);
}
```

Now if we run our test, we'll see exactly which values FsCheck generated as test input:

```bash
MD5ReturnsStringOfCorrectLength [FAIL]

    FsCheck.Xunit.PropertyFailedException : 
    Falsifiable, after 50 tests (38 shrinks) (StdGen (1153044621,296119646)):
    Original:
    "XqyW\O!Lr%ce3]4=H~=6lG, 5lT\aDz%n9"
    Shrunk:
    "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"

    ---- Assert.Equal() Failure
    Expected: 32
    Actual:   33

    Output:
    0: "5z"
    1: "Y"
    2: "r"
    3: ""
    4: "t9[Q"
    ...
    45: <null>
    46: "Qlbz?|perK"
    47: "XP3vO$-`l"
    48: Y{q6oevZA7"0R
    49: "XqyW\O!Lr%ce3]4=H~=6lG, 5lT\aDz%n9"

    shrink: "qyW\O!Lr%ce3]4=H~=6lG, 5lT\aDz%n9"
    shrink: "yW\O!Lr%ce3]4=H~=6lG, 5lT\aDz%n9"
    shrink: "yW\O!Lr%ce3]4=H~=6lG,5lT\aDz%n9"
    shrink: "W\O!Lr%ce3]4=H~=6lG,5lT\aDz%n9"
    shrink: "\O!Lr%ce3]4=H~=6lG,5lT\aDz%n9"
    shrink: "O!Lr%ce3]4=H~=6lG,5lT\aDz%n9"
    shrink: "O!Lr%ce3]4=H~=6lG,5lT\aDz%n9a"
    shrink: "O!Lr%ce3]4=H~=6lG,5lT\aDz%naa"
    shrink: "O!Lr%ce3]4=H~=6lG,5lT\aDz%aaa"
    shrink: "O!Lr%ce3]4=H~=6lG,5lT\aDzaaaa"
    shrink: "O!Lr%ce3]4=H~=6lG,5lT\aDaaaaa"
    shrink: "O!Lr%ce3]4=H~=6lG,5lT\aaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6lG,5lTaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6lG,5lTaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6lG,5laaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6lG,5laaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6lG,5aaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6lG,aaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6lG,aaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6lGaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6laaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=6aaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~=aaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=H~aaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=Haaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4=aaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]4aaaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3]aaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ce3aaaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%ceaaaaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%caaaaaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%caaaaaaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lr%aaaaaaaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Lraaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!Laaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "O!aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "Oaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
    shrink: "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
```        

The test report starts with the 50 randomly generated input values (note: we omitted most for brevity). The last input value (`"XqyW\O!Lr%ce3]4=H~=6lG, 5lT\aDz%n9"`), listed as input #49, is the first for which the test failed. Having found an input value that fails the test, FsCheck then starts the shrinking process.

To see how shrinking works, let's list the failing input value and the first six subsequent shrinks:

```markdown
| # | Input                                | Length | Passes test
| - | -----------------------------------: | ------ | -----------
| 0 | "XqyW\O!Lr%ce3]4=H~=6lG, 5lT\aDz%n9" | 33     | false
| 1 | "qyW\O!Lr%ce3]4=H~=6lG, 5lT\aDz%n9"  | 32     | true
| 2 | "yW\O!Lr%ce3]4=H~=6lG, 5lT\aDz%n9"   | 31     | true
| 3 | "yW\O!Lr%ce3]4=H~=6lG,5lT\aDz%n9"    | 30     | true
| 4 | "W\O!Lr%ce3]4=H~=6lG,5lT\aDz%n9"     | 29     | true
| 5 | "\O!Lr%ce3]4=H~=6lG,5lT\aDz%n9"      | 28     | true
| 6 | "O!Lr%ce3]4=H~=6lG,5lT\aDz%n9"       | 27     | true
```

The pattern here is quite obvious: with each shrinking step, the previous value is stripped of its first character, decreasing the length by one. Note that all strings of length 32 or less pass the test. FsCheck will use this fact later on.

The next six shrinks follow a different pattern:

```markdown
| #  | Input                                | Length | Passes test
| -- | -----------------------------------: | ------ | -----------
| 7  | "O!Lr%ce3]4=H~=6lG,5lT\aDz%n9a"      | 28     | true
| 8  | "O!Lr%ce3]4=H~=6lG,5lT\aDz%naa"      | 28     | true
| 9  | "O!Lr%ce3]4=H~=6lG,5lT\aDz%aaa"      | 28     | true
| 10 | "O!Lr%ce3]4=H~=6lG,5lT\aDzaaaa"      | 28     | true
| 11 | "O!Lr%ce3]4=H~=6lG,5lT\aDaaaaa"      | 28     | true
| 12 | "O!Lr%ce3]4=H~=6lG,5lT\aaaaaaa"      | 28     | true
```

This time, each shrink step replaces one character with the `'a'` character. FsCheck uses this shrink strategy to check if replacing specific characters in the input string can make the test fail.  

As changing characters in the input values did not make the test fail, FsCheck then uses the fact that the only input value that failed the test had a length of 33. It therefore modifies its shrinking strategy and starts generating longer input values, which will lead to generating a 33 length input. Besides generating longer input values, it will additionaly also use the one character modification shrinking strategy as an extra check. This leads to the following sequence of shrinks:

```markdown
| #  | Input                                | Length | Passes test
| -- | :----------------------------------- | ------ | -----------
| 13 | "O!Lr%ce3]4=H~=6lG,5lTaaaaaaaa"      | 29     | true
| 14 | "O!Lr%ce3]4=H~=6lG,5lTaaaaaaaaa"     | 30     | true
| 15 | "O!Lr%ce3]4=H~=6lG,5laaaaaaaaaa"     | 30     | true
| 16 | "O!Lr%ce3]4=H~=6lG,5laaaaaaaaaaa"    | 31     | true
| 17 | "O!Lr%ce3]4=H~=6lG,5aaaaaaaaaaaa"    | 31     | true
| 18 | "O!Lr%ce3]4=H~=6lG,aaaaaaaaaaaaa"    | 31     | true
| 19 | "O!Lr%ce3]4=H~=6lG,aaaaaaaaaaaaaa"   | 32     | true
| 20 | "O!Lr%ce3]4=H~=6lGaaaaaaaaaaaaaaa"   | 32     | true
```

The 11 shrinks that follow these shrinks are all 32 character strings with one character replaced, which all pass the test. Things start getting interesting from shrink #32, which is a string of length 33 and thus fails the test:

```markdown
| #  | Input                               | Length | Passes test
| -- | ----------------------------------- | ------ | -----------
| 32 | "O!Lr%caaaaaaaaaaaaaaaaaaaaaaaaaaa" | 33     | false
```

At this point, FsCheck correctly infers that strings of length 33 form the minimal set of counter-examples to our property. The final shrinking steps again use the single character replacement shrinking strategy, to see if changing a single character in an input value of length 33 can make the test pass:

```markdown
| #  | Input                               | Length | Passes test
| -- | ----------------------------------- | ------ | -----------
| 33 | "O!Lr%aaaaaaaaaaaaaaaaaaaaaaaaaaaa" | 33     | false
| 34 | "O!Lraaaaaaaaaaaaaaaaaaaaaaaaaaaaa" | 33     | false
| 35 | "O!Laaaaaaaaaaaaaaaaaaaaaaaaaaaaaa" | 33     | false
| 36 | "O!aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa" | 33     | false
| 37 | "Oaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa" | 33     | false
| 38 | "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa" | 33     | false
```

Even with all characters changed, the input still fails the test. At this point, FsCheck considers the input value `"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"` to be the minimal counter-example (or "shrunk") to our property-based test.

Now that we know our test fails for strings of length 33 (and greater), we can fix our implementation as follows:

```csharp
public static string MD5(string input)
{
    if (input == "foo")
    {
        return "acbd18db4cc2f85cedef654fccc4a4d8";
    }
    if (input == "bar")
    {
        return "37b51d194a7513e45b56f6524f2d51f2";
    }
    if (input == "baz")
    {
        return "73feffa4b7f6bb68e44cf984c85f6e88";
    }
    
    return (input ?? string.Empty).PadRight(32).Substring(0, 32);
}
```

Now, our test passes all generated inputs, and we have our first, working, property-based test! 

Clearly, this single property test did not force us to write a correct implementation, but that is perfectly normal. With property-based testing, you usually need to write several property-based tests before you are finally forced to write a correct implementation, so let's move on to property two.

### Property 2: hash contains only hexadecimal characters

Our second property states that the MD5 hash consists of only hexadecimal characters:

```csharp
[Property]
public void MD5ReturnsStringWithOnlyAlphaNumericCharacters(string input)
{
    var hash = MD5(input);
    var allowed = "0123456789abcdef".ToCharArray();
    Assert.All(hash, c => Assert.Contains(c, allowed));
}
```

This test should fail for any input other than the `"foo"`, `"bar"` or `"baz"` strings, which indeed it does:

```bash
MD5ReturnsStringWithOnlyAlphaNumericCharacters [FAIL]

    FsCheck.Xunit.PropertyFailedException : 
    Falsifiable, after 1 test (0 shrinks) (StdGen (961984695,296121694)):
    Original:
    ""

    ---- Assert.All() Failure: 32 out of 32 items in the collection did not pass.
    [31]: Xunit.Sdk.ContainsException: Assert.Contains() Failure
        Not found: ' '
        In value:  Char[] ['0', '1', '2', '3', '4', ...]
```

 Let's fix this:

```csharp
public static string MD5(string input)
{
    if (input == "foo")
    {
        return "acbd18db4cc2f85cedef654fccc4a4d8";
    }
    if (input == "bar")
    {
        return "37b51d194a7513e45b56f6524f2d51f2";
    }
    if (input == "baz")
    {
        return "73feffa4b7f6bb68e44cf984c85f6e88";
    }
    
    return "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
}
```

This implementation not only passes this test, but also all other tests. On to property three.

### Property 3: same hash for same input

This test verifies that when presented with the same input, the same hash is returned:

```csharp
[Property]
public void MD5ReturnsSameHashForSameInput(string input)
{
    var hash1 = MD5(input);
    var hash2 = MD5(input);
    Assert.Equal(hash1, hash2);
}
```

Simple enough, and the current implementation passes this test.

### Property 4: hash is different from input

Our next property verifies that the hash is different from the input value (which is an essential property of any hashing algorithm):

```csharp
[Property]
public void MD5ReturnsStringDifferentFromInput(string input)
{
    var hash = MD5(input);
    Assert.NotEqual(input, hash);
}
```

At the moment, this test will pass for every input string except `"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"`. That means that unless FsCheck randomly generates the `"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"` input value, the property-based test will pass and we would think our implementation was correct. Let's demonstrate this by adding an example-based test: 

```csharp
[Fact]
public void MD5ReturnsStringDifferentFromManyAsInput()
{
    var input = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
    var hash = MD5(input);
    Assert.NotEqual(input, hash);
}
```

If we run both the property-based and the example-based tests, only the example-based test fails:

```bash
MD5ReturnsStringDifferentFromManyAsInput [FAIL]

    Assert.NotEqual() Failure
    Expected: Not "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
    Actual:   "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
```

This is one of the drawbacks of testing with random data: you can have tests that *sometimes* fail. In such cases, augmenting a property-based test with an example-based test can be quite useful. 

The fix is simple of course:

```csharp
public static string MD5(string input)
{
    if (input == "foo")
    {
        return "acbd18db4cc2f85cedef654fccc4a4d8";
    }
    if (input == "bar")
    {
        return "37b51d194a7513e45b56f6524f2d51f2";
    }
    if (input == "baz")
    {
        return "73feffa4b7f6bb68e44cf984c85f6e88";
    }
    if (input == "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa")
    {
        return "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb";
    }
    
    return "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
}
```

Let's move on to our last property.  

### Property 5: similar inputs have non-similar hashes

Our last property states a very important property of MD5 hashes: similar inputs should not return similar hashes. For example, the hash for the string `"hello"` should be completely different from the hash for the string `"hello1"`. 

To write a test for this property, we have to define how we calculate the "difference" between two strings. In this case, we'll just use a simple algorithm that counts the number of positions where the strings have different characters. Obviously, this simple algorithm would normally not suffice, but it works for our purposes. Note that for previty we'll omit the actual implementation.

The property test looks like this:

```csharp
[Property]
public void MD5WithSimilarInputsInputsDoesNotReturnSimilarHashes(string input, char addition)
{
    var similar = input + addition;
    var hash1 = MD5(input);
    var hash2 = MD5(similar);
    var difference = Difference(hash1, hash2);
    Assert.InRange(difference, 5, 32);
}
```

In this test, we let FsCheck generate two input values: the input value and a character to append to the input value. We then calculate the hashes for the original and modified input, calculate the difference and verify that the difference is at least 5 characters (which is was chosen arbitrarily). 

In we run our test again, it fails:

```bash
MD5WithSimilarInputsInputsDoesNotReturnSimilarHashes [FAIL]

    FsCheck.Xunit.PropertyFailedException : 
    Falsifiable, after 1 test (2 shrinks) (StdGen (1928700473,296121700)):
    Original:
    ("D", 'F')
    Shrunk:
    ("", 'a')

    ---- Assert.InRange() Failure
    Range:  (5 - 32)
    Actual: 0
```

Modifying our implementation is still possible, but it becomes increasingly harder. At this point, we'll give in and correctly implement the MD5 algorithm:

```csharp
public static string MD5(string input)
{
    using (var md5 = System.Security.Cryptography.MD5.Create()) 
    {
        var inputBytes = System.Text.Encoding.ASCII.GetBytes(input);
        var hash = md5.ComputeHash(inputBytes);

        var sb = new StringBuilder();

        for (var i = 0; i < hash.Length; i++)
        {
            sb.Append(hash[i].ToString("x2"));
        }

        return sb.ToString();    
    }
}
```

Let's run all our test to verify they all pass:

```bash
MD5WithSimilarInputsInputsDoesNotReturnSimilarHashes [FAIL]
      FsCheck.Xunit.PropertyFailedException : 
      Falsifiable, after 13 tests (1 shrink) (StdGen (1520259769,296121702)):
      Original:
      (null, 'm')
      Shrunk:
      (null, 'a')
      
    ---- System.ArgumentNullException : String reference not set to an instance of a String.

MD5ReturnsStringWithOnlyAlphaNumericCharacters [FAIL]
      FsCheck.Xunit.PropertyFailedException : 
      Falsifiable, after 8 tests (0 shrinks) (StdGen (1521993499,296121702)):
      Original:
      <null>
      
    ---- System.ArgumentNullException : String reference not set to an instance of a String.

MD5ReturnsStringDifferentFromInput [FAIL]
      FsCheck.Xunit.PropertyFailedException : 
      Falsifiable, after 4 tests (0 shrinks) (StdGen (1522075879,296121702)):
      Original:
      <null>
      
    ---- System.ArgumentNullException : String reference not set to an instance of a String.

MD5ReturnsSameHashForSameInput [FAIL]
    FsCheck.Xunit.PropertyFailedException : 
    Falsifiable, after 2 tests (0 shrinks) (StdGen (1522087989,296121702)):
    Original:
    <null>
      
    ---- System.ArgumentNullException : String reference not set to an instance of a String.

MD5ReturnsStringOfCorrectLength [FAIL]
    FsCheck.Xunit.PropertyFailedException : 
    Falsifiable, after 28 tests (0 shrinks) (StdGen (1522097819,296121702)):
    Original:
    <null>
      
    ---- System.ArgumentNullException : String reference not set to an instance of a String.
```

Oh no, by correctly implementing the MD5 method, we made all our property-based tests fail! What happened? Well, previously, we correctly handled `null` input values in our implementation, but we don't anymore. If we think about it, `null` can be consider an invalid input value and throwing an `ArgumentNullException` is thus perfectly reasonable. We could write an example-based test to verify this behavior:

```csharp
[Fact]
public void MD5WithNullInputThrowsArgumentNullException()
{
    Assert.Throws<ArgumentNullException>(() => MD5(null));
}
```

To fix our failing property-based tests, we should rephrase our properties to state that they only hold for valid (*non-null*) input values. Our next step is to make FsCheck only generate valid, non-null input values. 

#### Customizing input value generation

FsCheck generates values through a concept known as *arbitraries*. To fix our failing property tests, we'll define a function that returns a `Arbitrary<string>`, which FsCheck will use to generate strings with. Our custom `Arbitrary<string>` instance can generate *any* string, expect the `null` string. 

To define our custom arbitrary, we create a class with a static method that returns an `Arbitrary<string>` instance: 

```csharp
public static class NonNullStringArbitrary
{
    public static Arbitrary<string> Strings()
    {
        return Arb.Default.String().Filter(x => x != null);
    }
} 
```

The `Strings()` method returns a modified `Arbitrary<string>` instance that filters out the `null` string. Note that the name of the method is arbitrary (pun intended), FsCheck will ignore the name and only look at the return type.

We can then use our custom arbitrary in our tests through the `Arbitrary` property of the `[Property]` attribute:

```csharp
[Property(Arbitrary = new[] { typeof(NonNullStringArbitrary) })]
public void MD5ReturnsStringOfCorrectLength(string input)
{
    var hash = MD5(input);
    Assert.Equal(32, hash.Length);
}
```

If we now run our test, our custom arbitrary is used, which means the `null` input will not be generated and all our implementation passes all property-based tests!

To make using our custom arbitrary easier, we can create an attribute that derives from `PropertyAttribute` that automatically sets the `Arbitrary` property to our custom arbitrary:

```csharp
public class MD5PropertyAttribute : PropertyAttribute
{
    public MD5PropertyAttribute()
    {
        Arbitrary = new[] { typeof(NonNullStringArbitrary) };
    }
}
```

We can now use this attribute instead of using a `[Property]` attribute:

```csharp
[MD5Property]
public void MD5ReturnsStringOfCorrectLength(string input)
{
    var hash = MD5(input);
    Assert.Equal(32, hash.Length);
}
```

Much better. The final step is to use this custom attribute on all our property-based tests. To verify we didn't break anything, we run all tests one more time and thankfully, they all still pass!

## Regular tests or property tests?

Now that we know how to write property-based tests, should we stop writing example-based tests? Well, probably not. 

One problem with property-based tests is that it can be hard to identify a set of properties that completely cover the desired functionality. In those cases, it can be very useful to also define some example-based tests. This is what we did in our example. The five properties for which we wrote tests did not completely describe the MD5 algorithm though, which is why we also needed an additional parameterized example-based test to ensure the correctness of our implementation.

Furthermore, as example-based tests are less abstract, they are usually easier to understand. Defining some example-based tests in addition to your property-based tests, can thus help explain what you are testing.

## Practicing

Propery-based testing is not easy to start with. Especially in the beginning, you'll probably struggle to identify properties. A good way to practice your property-based testing skills, is to solve the [Diamond kata](http://claysnow.co.uk/recycling-tests-in-tdd/) using property-based testing. 

If you'd like to learn more about FsCheck and property-based testing, check out the following links:

- [An introduction to property-based testing](http://fsharpforfunandprofit.com/posts/property-based-testing/) (F#)
- [Diamond kata with FsCheck](http://blog.ploeh.dk/2015/01/10/diamond-kata-with-fscheck/) (F#)
- [A (Gentle) Introduction to Property-based Testing](https://blog.entelect.co.za/view/9742/a-gentle-introduction-to-property-based-testing) (C#)

FsCheck is the most popular property-based framework for the .NET platform, but other platforms have their own implementation. The most well-known are [QuickCheck](https://wiki.haskell.org/Introduction_to_QuickCheck1) (Haskell), and [ScalaCheck](https://www.scalacheck.org/) (JVM), which both inspired FsCheck.   

## Conclusion

Although writing tests is important, the way you write your tests is equally important. To strengthen example-based tests, you should use as many inputs as possible. While multiple inputs are better than one input, property-based testing is even better as it works with *any* input value. It achieves this by randomly generating a large number of input values. 

The random nature of property-based testing forces you to approach your tests differently. You don't focus on specific use cases, but on the general properties of the code you want to test. This helps you better understand the requirements of the code you want to test. Testing with random inputs also makes hard-coding an implementation using pre-defined input values extremely hard. Furthermore, the clever way in which property-based testing frameworks can provide minimal counter-examples, really helps identifying and fixing issues in your implementation.

The main problem with property-based testing is that it can be hard to get started with. In particular, figuring out what properties to test can be quite hard. Furthermore, property-based tests can fail to completely describe all aspects of the implementation. In those cases, you should augment property-based tests with regular example-based tests.