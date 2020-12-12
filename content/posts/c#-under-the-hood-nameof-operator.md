---
title: "C# 6 under the hood: nameof operator"
date: 2015-12-31
tags:
  - csharp
  - nameof
  - cil
url: /2015/12/31/csharp6-under-the-hood-nameof/
---

In C# 6, the `nameof` operator allows you to retrieve the name of a variable, type or member.

## Example

The following example shows how you can use the `nameof` operator to retrieve the name of a namespace, class, method, parameter, property, field or variable:

```csharp
using System;

public class Program
{
    private static DateTime Today = DateTime.Now;

    public string Name { get; set; }

    public static void Main(string[] args)
    {
        var localTime = DateTime.Now.ToLocalTime();
        var åçéñøûß = true;

        Console.WriteLine(nameof(localTime));     // "localTime"
        Console.WriteLine(nameof(åçéñøûß));       // "åçéñøûß"
        Console.WriteLine(nameof(args));          // "args"
        Console.WriteLine(nameof(System.IO));     // "IO"
        Console.WriteLine(nameof(Main));          // "Main"
        Console.WriteLine(nameof(Program));       // "Program"
        Console.WriteLine(nameof(Program.Today)); // "Today"
        Console.WriteLine(nameof(Program.Name));  // "Name"
    }
}
```

## Restrictions

Although the `nameof` operator works with most language constructs, there are some restrictions. For example, you cannot use the `nameof` operator on open generic types or method return values:

```csharp
using System;
using System.Collections.Generic;

public class Program
{
    public static int Main()
    {
        Console.WriteLine(nameof(List<>)); // Compile-time error
        Console.WriteLine(nameof(Main())); // Compile-time error

        return 0;
    }
}
```

Furthermore, if you apply it to a generic type, the generic type parameter will be ignored:

```csharp
using System;
using System.Collections.Generic;

public class Program
{
    public static void Main()
    {
        Console.WriteLine(nameof(List<int>));  // "List"
        Console.WriteLine(nameof(List<bool>)); // "List"
    }
}
```

## When to use?

So when should you use the `nameof` operator? The prime example are exceptions that take a parameter name as an argument. Consider the following code:

```csharp
public static void DoSomething(string val)
{
    if (val == null)
    {
        throw new ArgumentNullException("val");
    }
}
```

If the `val` parameter is `null`, an `ArgumentNullException` is thrown with the parameter name (`"val"`) as a string argument. However, if we do a rename refactoring of the `val` parameter, the `"val"` string will not be modified. This results in the wrong parameter name being passed to the `ArgumentNullException`:

```csharp
public static void DoSomething(string input)
{
    if (input == null)
    {
        throw new ArgumentNullException("val");
    }
}
```

Instead of hard-coding the parameter name as a string, we should use the `nameof` operator :

```csharp
public static void DoSomething(string val)
{
    if (val == null)
    {
        throw new ArgumentNullException(nameof(val));
    }
}
```

As `nameof(val)` returns the string `"val"`, the functionality is unchanged. However, as the `nameof` operator references the `val` parameter, a rename refactoring of that parameter also changes the `nameof` operator's argument:

```csharp
public static void DoSomething(string input)
{
    if (input == null)
    {
        throw new ArgumentNullException(nameof(input));
    }
}
```

This time, the refactoring did not break anything - the correct parameter name is still passed to the `ArgumentNullException` constructor.

Now we know how to use the `nameof` operator, let's find out how it is implemented.

## Under the hood

One way to find out how a C# feature is implemented, is by looking at the [Common Intermediate Language](https://en.wikipedia.org/wiki/Common_Intermediate_Language) (CIL) code the compiler generates. As a refresher, the following diagram describes how C# code is compiled:

![Compilation of .NET code](/images/posts/csharp6-under-the-hood-nameof/dotnet-compilation.svg)

There are basically two steps:

1. Compiling C# code to CIL code. This is done by the C# compiler and happens at _compile time_.
2. Compiling CIL code to native code. This is done by the CLR and happens at _runtime_.

As C# code is compiled to CIL code, examining the generated CIL code gives us insight in how C# features are implemented by the compiler. As an example, C# properties are compiled to plain getter and setter methods at the CIL level.

Let's examine the CIL code that is generated for the aforementioned examples.

### CIL code for string-based example

To view the CIL code generated for our string-based example, we first have to compile it. Just as a reminder, this is the C# code for our string-based example:

```csharp
public static void DoSomething(string val)
{
    if (val == null)
    {
        throw new ArgumentNullException("val");
    }
}
```

To examine the CIL code the compiler generates, we compile our code. As this code is part of a console application, the C# compiler writes the CIL code to an executable.

To view the CIL code in the compiled executable, we can use a disassembler. We'll use [ildasm](<https://msdn.microsoft.com/en-us/library/f7dy01k1(v=vs.110).aspx>), which is a disassembler that comes pre-installed with Visual Studio. `ildasm` can be used both as a command-line and GUI tool, but we'll use the command-line functionality.

To use `ildasm` on our executable, we open a [Visual Studio Command Prompt](<https://msdn.microsoft.com/en-us/library/ms229859(v=vs.110).aspx>). Then, we navigate to the directory that contains the executable we just compiled. Finally, we call `ildasm` with our executable's name as its argument: `ildasm nameof.exe /text`.

This command will output all CIL code in the `nameof.exe` executable to the console (note: omit the `/text` modifier to open the `ildasm` GUI). Amongst the CIL code written to the console is the CIL code for our `DoSomething` method:

```
.method public hidebysig static void  DoSomething(string val) cil managed
{
  // Code size       22 (0x16)
  .maxstack  2
  .locals init ([0] bool V_0)
  IL_0000:  nop
  IL_0001:  ldarg.0
  IL_0002:  ldnull
  IL_0003:  ceq
  IL_0005:  stloc.0
  IL_0006:  ldloc.0
  IL_0007:  brfalse.s  IL_0015
  IL_0009:  nop
  IL_000a:  ldstr      "val"
  IL_000f:  newobj     instance void [mscorlib]System.ArgumentNullException::.ctor(string)
  IL_0014:  throw
  IL_0015:  ret
} // end of method Program::DoSomething
```

These instructions are the CIL representation of our C# code. If you are not familiar with CIL code, this may be a bit daunting. However, you don't have to know [all CIL instructions](https://en.wikipedia.org/wiki/List_of_CIL_instructions) to understand what is happening. Let's walk through each instruction to see what is happening.

- `.maxstack`: set the maximum number of items on the stack to 2.
- `.locals`: create a local variable of type `bool` at index 0.
- `IL_0000`: do nothing. This instruction allows a breakpoint to be placed on a non-executable piece of code; in this case, the method's opening curly brace.
- `IL_0001`: push the `val` parameter value (the argument at index 0) on the stack.
- `IL_0002`: push the `null` value on the stack.
- `IL_0003`: pop the top two stack parameters from the stack, compare them for equality, and push the comparison result on the stack.
- `IL_0005`: pop the top stack value (the equality comparison result) and store it in the local variable with index 0.
- `IL_0006`: push the local variable at index 0 (the equality comparison result) on the stack.
- `IL_0007`: pop top stack value (the equality comparison result) from stack. If this value is equal to 0 (`false`), jump to statement `IL_0015` (end of method). If not, do nothing.
- `IL_0009`: do nothing (allows breakpoint at opening curly brace inside `if` statement).
- `IL_000a`: push the string `"val"` on the stack.
- `IL_000f`: pop the top stack value (the `"val"` string) from the stack, and push a new `ArgumentNullException` instance on the stack by calling its constructor that takes the popped `"val"` string as its argument.
- `IL_0014`: pop the `ArgumentNullException` from the stack and throw it.
- `IL_0015`: return from the method.

While there is certainly more CIL code than there was C# code, it is not that hard to grasp what is happening if you are familiar with [stacks](<https://en.wikipedia.org/wiki/Stack_(abstract_data_type)>).

#### Optimizing

There is something odd about the generated CIL instructions though. For example, instructions `IL_0005` and `IL_0006` seem to be redundant, as the first instruction pops a value from the stack which the latter instruction then immediately pushes back on the stack. Removing these instructions would not change the behavior of the code and would improve its performance. So how can we instruct the compiler to omit these redundant instructions?

Well, note that we built our code using the default `Debug` build configuration. If you build code using the default `Debug` build configuration, the C# compiler will **not** apply optimizations to the CIL being output. This leads to valid, but often inefficient CIL code. However, if we switch to the `Release` build configuration and compile our code again, the compiler _will_ optimize the CIL code. Let's see what the optimized CIL code for our example is.

### CIL code for string-based example (optimized)

To see the optimized CIL code for our example, we select the `Release` build configuration and rebuild our code. This time, the generated CIL code looks remarkably different:

```
.method public hidebysig static void  DoSomething(string val) cil managed
{
  // Code size       15 (0xf)
  .maxstack  8
  IL_0000:  ldarg.0
  IL_0001:  brtrue.s   IL_000e
  IL_0003:  ldstr      "val"
  IL_0008:  newobj     instance void [mscorlib]System.ArgumentNullException::.ctor(string)
  IL_000d:  throw
  IL_000e:  ret
} // end of method Example::DoSomething
```

This optimized CIL code does the following things:

- `.maxstack`: set the maximum number of items on the stack to 8.
- `IL_0000`: push the `val` parameter value (the argument at index 0) on the stack.
- `IL_0001`: pop the top stack value (the `val` parameter value) from the stack. If this value is equal to `true` (which is the same as: not `null`), jump to statement `IL_000e` (end of method).
- `IL_0003`: push the string `"val"` on the stack.
- `IL_0008`: pop the top stack value (the `"val"` string) from the stack, and push a new `ArgumentNullException` instance on the stack by calling its constructor that takes the popped `"val"` string as its argument.
- `IL_000d`: pop the `ArgumentNullException` from the stack and throw it.
- `IL_000e`: return from the method.

As you can see, the functionality is still the same, but the number of CIL instructions has been drastically reduced. This not only leads to improved performance, but also makes it easier to see what is happening in the CIL code.

### CIL code for nameof operator

Let's see what CIL code is generated for our example that uses the `nameof` operator. The C# code that uses the `nameof` operator looks like this:

```csharp
public static void DoSomething(string val)
{
    if (val == null)
    {
        throw new ArgumentNullException(nameof(val));
    }
}
```

If we compile this code (using a `Release` build) and run `ildasm` again, the following optimized CIL is generated for the `DoSomething` method:

```
.method public hidebysig static void  DoSomething(string val) cil managed
{
  // Code size       15 (0xf)
  .maxstack  8
  IL_0000:  ldarg.0
  IL_0001:  brtrue.s   IL_000e
  IL_0003:  ldstr      "val"
  IL_0008:  newobj     instance void [mscorlib]System.ArgumentNullException::.ctor(string)
  IL_000d:  throw
  IL_000e:  ret
} // end of method Example::DoSomething
```

Interestingly, the CIL generated for the `nameof` version is _exactly the same_ as the string-based version! The `nameof` operator is thus just [_syntactic sugar_](https://en.wikipedia.org/wiki/Syntactic_sugar) for a plain CIL string - there is no `nameof` operator at the CIL level. This means that there is no runtime overhead to using the `nameof` operator over plain strings.

### CIL string optimizations

As the `nameof` operator compiles to a plain string in CIL, do strings optimizations also apply when the `nameof` operator is used?

As an example of such an optimization, consider the following code:

```csharp
public static void DoSomething(string val)
{
    Console.WriteLine ("Parameter name: " + "val");
}
```

Can you guess what CIL code will be generated for this C# code? If you figured there would be string concatenation, you'd be wrong. Let's check the optimized CIL code the compiler outputs:

```
.method public static hidebysig default void DoSomething (string val)  cil managed
{
    // Method begins at RVA 0x2058
    // Code size 11 (0xb)
    .maxstack 8
    IL_0000:  ldstr "Parameter name: val"
    IL_0005:  call void class [mscorlib]System.Console::WriteLine(string)
    IL_000a:  ret
} //
```

You can clearly see that instead of using CIL instructions to concatenate the `"Parameter name: "` and `"val"` strings, the compiler just outputs the concatenated strings directly. Obviously, this compiler optimization saves both time and memory.

To find out if this optimization also applies when the `nameof` operator is used, we modify our example to use the `nameof` operator:

```csharp
public static void DoSomething(string val)
{
    Console.WriteLine ("Parameter name: " + nameof(val));
}
```

We then recompile and inspect the generated CIL code:

```
.method public static hidebysig default void DoSomething (string val)  cil managed
{
    // Method begins at RVA 0x2058
    // Code size 11 (0xb)
    .maxstack 8
    IL_0000:  ldstr "Parameter name: val"
    IL_0005:  call void class [mscorlib]System.Console::WriteLine(string)
    IL_000a:  ret
} //
```

As the generated CIL is the same as that of our string-based example, we have verified that existing string optimizations also apply when the `nameof` operator is used.

### Backwards compatibility

As we saw earlier, `nameof` operator calls are converted to strings in the generated CIL code; there is no `nameof` concept at the CIL level. This allows code that uses the `nameof` operator to run on older versions of the .NET framework, as far back as .NET framework 2.0 (and probably 1.0). The only restriction, of course, is that the compiler must be recent enough to know how to compile the `nameof` operator.

To verify this backwards compatibility, create a project that uses the `nameof` operator. Then, open the project properties page and change the `Target framework` to `.NET Framework 2.0`. At this point, you might get compile warnings for features not supported in .NET Framework 2.0. Remove all such features until the code compiles again. Note that the compiler did not complain about the `nameof` operator! Now recompile the application and run it. Everything should still work just fine.

### Compiler

Previously we used a disassembler to see what CIL code was generated by the C# compiler. However, as the C# compiler (nicknamed _Roslyn_) is open-source, we can also examine its source code to find out how the `nameof` operator is compiled.

#### Preparing

To explore Roslyn's source code, we first get a local copy of the [Roslyn repository](https://github.com/dotnet/roslyn). We then follow the [build instructions](https://github.com/dotnet/roslyn/wiki/Building%20Testing%20and%20Debugging) to build the `master` branch. Once the build script has finished (this can take a while), we open the `Compilers.sln` solution. In the `CSharp` solution folder, we then select the `csc` project as the startup project. This console application project builds `csc.exe`, the command-line version of the C# compiler. To see how `csc.exe` compiles C# code that uses the `nameof` operator, we specify the following command line arguments on the `Debug` tab of the `csc` project's properties page:

    "nameof.cs" /out:"nameof.exe" /optimize

The first argument is our C# source file that contains the `nameof` operator code. The compiler will compile this file and write the CIL output to the file specified in the `"/out"` argument. Finally, the `"/optimize"` flag will cause the compiler to emit optimized CIL code (note: this flag is set when doing a `Release` build).

Now, if we press F5 to debug the `csc` project, the C# compiler will compile the `nameof.cs` file and write the results to `nameof.exe`. At this point, we haven't set any breakpoints, we just let the compiler finish. After finishing, the compiler has created the `nameof.exe` executable. Using `ildasm`, we can see that `csc.exe` has generated the exact same CIL code as we saw in our previous examples.

#### Compilation pipeline

Before we dive head-first into Roslyn's source code to see how the `nameof` operator is compiled, let's see what the [Roslyn compilation pipeline](https://github.com/dotnet/roslyn/wiki/Roslyn%20Overview) looks like:

![Compilation of .NET code in Roslyn](/images/posts/csharp6-under-the-hood-nameof/roslyn.svg)

This diagram shows that Roslyn compiles C# source code in various stages. In the next sections, we'll look at what happens to our `nameof` source code in each phase.

#### First steps

We'll start exploring Roslyn's internals by debugging the `csc` project while it tries to compile our C# code. This time though, we set a breakpoint in the project's `Main` method (located in the `Program.cs` file). Once the breakpoint hits, we use `Step Into` (F11) to dig deeper and deeper into Roslyn's source code.

At first, the compiler does some pretty mundane stuff, like parsing command-line arguments. However, things start to get interesting when we arrive at the `RunCore()` method in the `CommonCompiler` class. This method implements the aforementioned compilation pipeline. Let's see how the `nameof` operator is processed in the various phases.

#### Parse phase

In this phase, the compiler uses the `ParseFile()` method to parse our source code into a [syntax tree](https://github.com/dotnet/roslyn/wiki/Roslyn%20Overview#syntax-trees). The following is a simplified version of the syntax tree the compiler creates for our C# code:

<pre>
NamespaceDeclaration   
└── ClassDeclaration   
    └── MethodDeclaration
        └── IfStatement
            └── ThrowStatement
                └── ObjectCreationExpression
                    └── ArgumentList
</pre>

Clearly, the syntax tree hierarchy directly corresponds to the hierarchy in our C# code. For example, the root node is the namespace declaration, which has a class declaration as its child.

Each syntax element in the syntax tree has a number of properties, such as a reference to its parent. Some syntax elements also have an identifier property, which is a string that contains the name of that syntax element. For example the `ClassDeclaration` describing the `Program` class has its identifier property set to `"Program"`.

Let's zoom in on the syntax tree for the C# code that uses the `nameof` operator: the `new ArgumentNullException(nameof(val))` expression. This time, we'll also show the syntax element's identifier in the tree (if it has one):

<pre>
ObjectCreationExpression
├── NewKeyword
├── IdentifierName ("ArgumentNullException")
└── ArgumentList
    └── Argument
        └── InvocationExpression
            ├── IdentifierName ("nameof")
            └── ArgumentList
                └── Argument
                    └── IdentifierName ("val")
</pre>

At the root of this part of the syntax tree is the `ObjectCreationExpression`. The object creation expression has three parts:

1. The `new` keyword.
2. The identifier name (`"ArgumentNullException"`).
3. The argument list.

The argument list subtree has a single child: an `Argument` that describes an `InvocationExpression`, which represents the `nameof(val)` argument. The syntax tree for this element has two parts:

1. The identifier of the method to be called (`"nameof"`).
2. The argument list, which contains a single argument that refers to the `"val"` identifier.

We can see that the the `nameof` operator is treated as a regular method call with a single argument. That argument is not a string though, but an `IdentifierName` element, which is a reference to another element with a matching identifier.

We know that in our code the `val` argument of the `nameof` call refers to the `DoSomething()` method's `val` parameter. Therefore, we expect the syntax tree of the `DoSomething()` method to have a parameter syntax element with its identifier set to `"val"`, which it does:

<pre>
MethodDeclaration ("DoSomething")
└── ParameterList
|   └── Parameter ("val")
└── Block
    └── IfStatement
        └── ...    
</pre>

In the next phase, the compiler will match these identifiers.

#### Declaration and bind phase

In the declaration and bind phase, identifiers are bound to symbols. In the parse phase, we saw that the `nameof` call was described by an `InvocationExpression`. The binding of this expression is done in the `BindInvocationExpression()` method.

The `BindInvocationExpression()` method starts by calling `TryBindNameofOperator()`. As our `InvocationExpression` indeed contains a `nameof` operator, the `BindNameofOperatorInternal()` method is called to handle the binding of the `nameof` operator.

The following, abbreviated code shows the `BindNameofOperatorInternal()` method:

```csharp
private BoundExpression BindNameofOperatorInternal(InvocationExpressionSyntax node, DiagnosticBag diagnostics)
{
    var argument = node.ArgumentList.Arguments[0].Expression;

    string name = "";
    CheckSyntaxForNameofArgument(argument, out name, diagnostics);

    ...

    return new BoundNameOfOperator(node, boundArgument, ConstantValue.Create(name), Compilation.GetSpecialType(SpecialType.System_String));
}
```

First, the `InvocationExpressionSyntax` element's single argument is retrieved, which is of type `IdentifierNameSyntax`. Then the `CheckSyntaxForNameofArgument()` method sets the `name` parameter to the identifier of the `IdentifierNameSyntax` element. In our example, the `name` variable is set to `"val"`.

Finally, the method returns a `BoundNameOfOperator` instance, which is the bound symbol representation of the `InvocationExpressionSyntax`. Note that the third constructor argument is a call to `ConstantValue.Create()`, with the identifier name (`"val"`) as its argument. This method is implemented as follows:

```csharp
public static ConstantValue Create(string value)
{
    if (value == null)
    {
        return Null;
    }

    return new ConstantValueString(value);
}
```

By passing a `ConstantValue` instance to the `BoundNameOfOperator` constructor, we indicate that the `BoundNameOfOperator` symbol can also be represented by a constant value. The importance of this will become clear later.

#### Lowering

After the binding has finished, there is one thing left to: lowering. In the process of lowering, complex semantic constructs are rewritten in terms of simpler ones. For example, each `lock` call is lowered to a `try/finally` block that uses `Monitor.Enter` and `Monitor.Exit`.

As it turns out, lowering also applies to the `nameof` operator. The `VisitExpressionImpl()` method (in the `LocalRewriter` class) is called for each `BoundExpression` instance, including derived classes such as the `BoundNameOfOperator` class. It looks like this:

```csharp
private BoundExpression VisitExpressionImpl(BoundExpression node)
{
    ConstantValue constantValue = node.ConstantValue;
    if (constantValue != null)
    {
        ...

        return MakeLiteral(node.Syntax, constantValue, type);
    }

    ...
}
```

This method takes a `BoundExpression` and also returns a `BoundExpression`. However, if the `ConstantValue` property is not `null`, the returned `BoundExpression` is of type `BoundLiteral`, as returned by the `MakeLiteral()` method:

```csharp
private BoundExpression MakeLiteral(CSharpSyntaxNode syntax, ConstantValue constantValue, TypeSymbol type, BoundLiteral oldNodeOpt = null)
{
    ...

    return new BoundLiteral(syntax, constantValue, type, hasErrors: constantValue.IsBad);
}
```

Therefore, when the `VisitExpressionImpl()` method is called to apply lowering to bound expressions, each `BoundNameOfOperator` instance is replaced by a `BoundLiteral` instance.

#### Emit phase

The last phase is the emit phase. Here, the bound symbols created in the previous phase are written to file as CIL code.

From our `nameof` viewpoint, things start getting interesting when the `EmitExpression()` method is called with the `BoundLiteral` instance (representing our `nameof` call) as its argument.

As we saw earlier, the `BoundLiteral` instance had its `ConstantValue` property set to an instance of `ConstantValueString` (containing the string `"val"`). This is important because normally, the compiler evaluates an expression to determine what code to emit. However, if the compiler find that an expression's `ConstantValue` property is not `null`, it will skip evaluating the expression but instead emit the constant value.

In the `EmitExpression()` method, you can clearly see this behavior:

```csharp
private void EmitExpression(BoundExpression expression, bool used)
{
    ...

    var constantValue = expression.ConstantValue;
    if (constantValue != null)
    {
        ...

        EmitConstantExpression(expression.Type, constantValue, used, expression.Syntax);
    }

    ...
}
```

As the `ConstantValue` property of our `BoundLiteral` instance is not null, the `EmitConstantExpression()` method is called, which in turn calls `EmitConstantValue()`:

```csharp
internal void EmitConstantValue(ConstantValue value)
{
    ConstantValueTypeDiscriminator discriminator = value.Discriminator;

    switch (discriminator)
    {
        ...

        case ConstantValueTypeDiscriminator.String:
            EmitStringConstant(value.StringValue);
            break;

        ...
    }
}
```

As the `ConstantValue` property of the `BoundLiteral` instance is of type `ConstantValueString`, the `EmitStringConstant()` method is called with the `StringValue` property as its argument. Note that for our `ConstantValueString` instance, the `StringValue` property returns the string `"val"`.

Having arrived at the `EmitStringConstant()` method, we are finally able to see how the CIL code for the `nameof` operator is emitted:

```csharp
internal void EmitStringConstant(string value)
{
    if (value == null)
    {
        EmitNullConstant();
    }
    else
    {
        EmitOpCode(ILOpCode.Ldstr);
        EmitToken(value);
    }
}
```

With `value` not being `null`, the `else` branch is executed. First, the CIL code for the `ldstr` opcode is emitted. Then, the `"val"` string token is emitted. This will output the CIL code we were expecting:

```
IL_0003:  ldstr      "val"
```

Our tour through Roslyn's internals has shown us how the `nameof` operator has been parsed to syntax elements, then bound to symbols and finally emitted as CIL code.

## Conclusion

The `nameof` operator is easy to understand and use. While its use is limited, it does make your code more robust. As the `nameof` operator is just _syntactic sugar_, there is no runtime performance impact. Furthermore, existing string optimizations also apply to the `nameof` operator and the compiled code runs on older versions of the .NET framework.

To find out how the `nameof` operator was implemented, we used `ildasm` to examine the generated CIL code and stepped through Roslyn's internals to see how the compiler generated the CIL code for the `nameof` operator.
