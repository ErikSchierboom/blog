---
title: "Combining nested model lambda's in Razor"
date: 2016-11-04
tags:
  - csharp
  - dotnet
  - aspnetmvc
  - razor
url: /2016/11/04/razor-combining-lambdas/
---

In ASP.NET MVC, views are defined by Razor templates. One very useful feature of Razor is that it allows you to generate HTML using simple lambda expressions, strongly-typed to your view model. However, if you have a complex, nested model and want to combine lambda expressions, things stop working. In this post, we'll see how to solve this problem.

## Simple example

Consider the following class that models user-related permissions:

```csharp
public class UserPermissions
{
    public bool Add { get; set; }
    public bool Edit { get; set; }
    public bool Delete { get; set; }
}
```

If we set the `@model` directive of a view to our `UserPermissions` class, the view's HTML helper will be bound to this class. We can then specify the property to generate HTML for by passing a lamdba to the HTML helper's methods:

```html
@model UserPermissions

<dl class="dl-horizontal">
  <dt>@Html.DisplayNameFor(m => m.Add)</dt>
  <dd>@Html.DisplayFor(m => m.Add)</dd>

  <dt>@Html.DisplayNameFor(m => m.Edit)</dt>
  <dd>@Html.DisplayFor(m => m.Edit)</dd>

  <dt>@Html.DisplayNameFor(m => m.Delete)</dt>
  <dd>@Html.DisplayFor(m => m.Delete)</dd>
</dl>
```

If we assume that `Add` is `true` and `Edit` and `Delete` are `false`, the following HTML output is rendered:

```html
<dl class="dl-horizontal">
  <dt>Add</dt>
  <dd>
    <input
      checked="checked"
      class="check-box"
      disabled="disabled"
      type="checkbox"
    />
  </dd>

  <dt>Edit</dt>
  <dd>
    <input class="check-box" disabled="disabled" type="checkbox" />
  </dd>

  <dt>Delete</dt>
  <dd>
    <input class="check-box" disabled="disabled" type="checkbox" />
  </dd>
</dl>
```

Clean and simple. Let's move to a more complex model.

## More complex example

For our more complex example, we'll create a model that stores the user permissions of several types of users:

```csharp
public class RolePermissions
{
    public UserPermissions User { get; set; }
    public UserPermissions Moderator { get; set; }
    public UserPermissions Administrator { get; set; }
}
```

The following view allows us to edit these permissions:

```html
@model RolePermissions @using (Html.BeginForm()) { @Html.AntiForgeryToken()

<table>
  <thead>
    <tr>
      <th></th>
      <th>@Html.DisplayNameFor(m => m.User)</th>
      <th>@Html.DisplayNameFor(m => m.Moderator)</th>
      <th>@Html.DisplayNameFor(m => m.Administrator)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>@Html.DisplayNameFor(m => m.User.Add)</td>
      <td>@Html.EditorFor(m => m.User.Add)</td>
      <td>@Html.EditorFor(m => m.Moderator.Add)</td>
      <td>@Html.EditorFor(m => m.Administrator.Add)</td>
    </tr>
    <tr>
      <td>@Html.DisplayNameFor(m => m.User.Edit)</td>
      <td>@Html.EditorFor(m => m.User.Edit)</td>
      <td>@Html.EditorFor(m => m.Moderator.Edit)</td>
      <td>@Html.EditorFor(m => m.Administrator.Edit)</td>
    </tr>
    <tr>
      <td>@Html.DisplayNameFor(m => m.User.Delete)</td>
      <td>@Html.EditorFor(m => m.User.Delete)</td>
      <td>@Html.EditorFor(m => m.Moderator.Delete)</td>
      <td>@Html.EditorFor(m => m.Administrator.Delete)</td>
    </tr>
  </tbody>
</table>

<button type="submit">Save</button>
}
```

Although this view is more complex, it's still quite readable. The rendered view looks something like this:

![Edit model view rendered](/images/posts/razor-combining-lambdas/edit-model-view-rendered.png)

It's not the prettiest of GUI's, but it is fully functional. To verify that our model is bound properly, let's first examine the view's controller:

```csharp
using System.Web.Mvc;

public class PermissionsController : Controller
{
    [HttpGet]
    public ActionResult Index()
    {
        return View();
    }

    [HttpPost]
    [ValidateAntiForgeryToken]
    public ActionResult Index(RolePermissions model)
    {
        return View(model);
    }
}
```

Once again, nothing exciting happening here. If we submit our form and put a breakpoint on the `return View(model);` line, we can verify that model binding still works perfectly.

## Removing duplication

Although our view looks nice, it does have a fair amount of duplication. In particular, the rows in the `<tbody>` section all have the exact same structure:

- The first column contains the name of the permission.
- The second column allows editing of that permission for the `User` role.
- The third column allows editing of that permission for the `Moderator` role.
- The fourth column allows editing of that permission for the `Administrator` role.

If one or more user types are added, or new types of permissions introduced, we have to edit our template in multiple places. This is tedious and error-prone, so let's try to remove this repetition.

In Razor, you can extract shared functionality into reusable components using [Razor helpers](https://www.asp.net/web-pages/overview/ui-layouts-and-themes/creating-and-using-a-helper-in-an-aspnet-web-pages-site), which are functions that return HTML. What's great though, is that you use a Razor template to define the HTML it returns. You can think of Razor helpers as a parameterized Razor view.

A first, naive attempt to create such a helper for our `<tbody>` rows would look something like this:

```csharp
@helper PermissionRow(Func<UserPermissions, bool> permission)
{
    <tr>
        <td>@Html.DisplayNameFor(m => permission(m.User))</td>
        <td>@Html.EditorFor(m => permission(m.User))</td>
        <td>@Html.EditorFor(m => permission(m.Moderator))</td>
        <td>@Html.EditorFor(m => permission(m.Administrator))</td>
    </tr>
}
```

We can then eliminate much of the duplication as follows:

```html
<tbody>
  @PermissionRow(permissions => permissions.Add) @PermissionRow(permissions =>
  permissions.Edit) @PermissionRow(permissions => permissions.Delete)
</tbody>
```

Unfortunately, if we execute this code, we get the following runtime exception:

<pre>
Additional information: Templates can be used only with field access, property access, single-dimension array index, or single-parameter custom indexer expressions.
</pre>

What this error message tells us, is that the HTML helper methods only accept a very restricted set of expressions, namely ones that directly reference a property. In our example, we used a function to reference the property, which is not supported.

In our previous example, we did directly reference the property: `@Html.EditorFor(m => m.User.Delete)`, which worked fine. That lambda's expression is of type `Expression<RolePermissions, bool>`, so how can we convert the `Func<UserPermissions, bool>` parameter of our Razor helper to `Expression<RolePermissions, bool>`? Let's find out!

## Solution

To convert our `Func<UserPermissions, bool>` expression to `Expression<RolePermissions, bool>`, we'll start by wrapping it in an `Expression`:

```csharp
@helper PermissionRow(Expression<Func<UserPermissions, bool>> permission)
{
    <tr>
        <td>@Html.DisplayNameFor(m => permission(m.User))</td>
        <td>@Html.EditorFor(m => permission(m.User))</td>
        <td>@Html.EditorFor(m => permission(m.Moderator))</td>
        <td>@Html.EditorFor(m => permission(m.Administrator))</td>
    </tr>
}
```

The reason we use an `Expression<Func<T>>` instead of a `Func<T>`, is that expressions can be transformed into other expressions. We'll see how we can use this feature shortly.

The next step is to extract the column rendering from our `PermissionRow` HTML helper to a separate HTML helper: `PermissionColumn`. This helper will receive two parameters:

1. An expression describing a function from `UserPermission` to `bool`. This expression points to the permission property: `Add`, `Edit` or `Delete`.
2. An expression describing a function from `RolePermissions` to `UserPermission`. This expression points to the user permission property: `User`, `Moderator` or `Administrator`.

The basic structure of this helper looks like this:

```csharp
@helper PermissionColumn(
    Expression<Func<UserPermissions, bool>> userPermission,
    Expression<Func<RolePermissions, UserPermissions>> rolePermission)
{
    <td>@Html.EditorFor("TODO: combine the two expressions")</td>
}
```

At this point, we are almost there. If we could combine the two expressions into one expression, in which we pipe the output of `Func<RolePermissions, UserPermissions>` into `Func<UserPermissions, bool>`, we would end up with an expression of type `Expression<Func<RolePermissions, bool>>`, precisely what we need!

So how can we combine these two expressions into a single expression? Let's start with the basic skeleton of the combine method:

```csharp
public static Expression<Func<RolePermissions, UserPermissions>>
    Combine<RolePermissions, UserPermissions, bool>
        (Expression<Func<RolePermissions, UserPermissions>> outer,
         Expression<Func<UserPermissions, bool>> inner)
{
    return Expression.Lambda<Func<RolePermissions, bool>>(TODO, TODO);
}
```

In this method, we use the `Expression.Lambda` method to create a new expression. It takes two parameters:

1. The body of the expression.
2. The parameters passed to the expression in the body.

If you think about it, the second parameter must be equal to the outer expression's parameters. That's one step completed:

```csharp
Expression.Lambda<Func<RolePermissions, bool>>(body, outer.Parameters);
```

For the first parameter, we need to construct an expression that represents the body of the combined expression. What we want is to pass the output of the `Func<RolePermissions, UserPermissions>` expression to the input parameter of the `Func<UserPermissions, bool>` expression. If you think about this backwards, this is the same as replacing the input parameter of the `Func<UserPermissions, bool>` expression with the output of the body of the `Func<RolePermissions, UserPermissions>` expression.

To transform/rewrite an expression, the `ExpressionVisitor` class can be used. Surprisingly, we need very little code to create a visitor that replaces one expression with another:

```csharp
private class SwapVisitor : ExpressionVisitor
{
    private readonly Expression _from;
    private readonly Expression _to;

    public SwapVisitor(Expression from, Expression to)
    {
        _from = from;
        _to = to;
    }

    public override Expression Visit(Expression node) => node == _from ? _to : base.Visit(node);
}
```

The constructor takes two `Expression` parameters:

1. The `from` parameter is the expression we want to replace.
2. The `to` parameter is the expression we want to replace it with.

The actual rewriting happens in the `Visit()` method. There, we check if the node parameter equals the `from` expression. If so, we return the `to` expression; otherwise, we just continue visiting the node.

We can now use this class to create an expression visitor in which we replace the inner expression's parameter (which has one parameter) with the outer expression's body:

```csharp
var swap = new SwapVisitor(inner.Parameters[0], outer.Body);
```

The final step is to use this swap visitor to create our target expression, which we do by calling its `Visit()` method with the inner expression's body:

```csharp
using System;
using System.Linq.Expressions;

public static class Expressions
{
    public static Expression<Func<RolePermissions, bool>>
        Combine<RolePermissions, UserPermissions, bool>
            (Expression<Func<RolePermissions, UserPermissions>> outer,
             Expression<Func<UserPermissions, bool>> inner)
    {
        var swap = new SwapVisitor(inner.Parameters[0], outer.Body);
        return Expression.Lambda<Func<TOuter, TProperty>>(
            swap.Visit(inner.Body), outer.Parameters);
    }
}
```

When the `Visit` method is called, the following happens:

1. The `node` parameter of the `Visit()` method will equal the inner expression's body. As this expression does not equal the `inner.Parameters[0]` expression, the expression is passed to `base.Visit()`.
2. Within `base.Visit()`, the expression's children are visited. As the expression is a function, at some point it will visit the expression representing the function's parameters.
3. The `Visit()` method is now called with the `inner.Parameters[0]` expression, which it recognizes to be equal to the `from` expression, so the `to` expression (the outer expression's body) is returned.

Finally, the `Visit` method will return the transformed expression, which has the type we were looking for: `Expression<Func<RolePermissions, bool>>`.

With our `PermissionColumn` helper is finished, we can use it in our `PermissionRow` helper:

```csharp
@helper PermissionRow(Expression<Func<UserPermissions, bool>> permission)
{
    var displayName = Expressions.Combine((RolePermissions model) => model.User, permission);
    <tr>
        <td>@Html.DisplayNameFor(displayName)</td>
        <td>@PermissionColumn(permission, m => m.User)</td>
        <td>@PermissionColumn(permission, m => m.Moderator)</td>
        <td>@PermissionColumn(permission, m => m.Administrator)</td>
    </tr>
}
```

We can now run our application to verify that everything still works, which it does!

### Making the extension generic

The final step is to make the `Combine` method generic, which is just a matter of replacing the concrete types with type parameters:

```csharp
public static Expression<Func<TOuter, TProperty>> Combine<TOuter, TInner, TProperty>(
    Expression<Func<TOuter, TInner>> outer,
    Expression<Func<TInner, TProperty>> inner)
{
    var swap = new SwapVisitor(inner.Parameters[0], outer.Body);
    return Expression.Lambda<Func<TOuter, TProperty>>(swap.Visit(inner.Body), outer.Parameters);
}
```

And there we have it: a fully generic method that can combine `Func<TOuter, TInner>` and `Func<TInner, TProperty>` expressions into an `Expression<Func<TOuter, TProperty>>` expression.

## Conclusion

HTML helpers in Razor views are a very convenient feature. In this post we showed how to use the `ExpressionVisitor` class to combine two expressions into a single expression. We used this feature to allow us to write small Razor helpers with which we were able to greatly reduce duplication in our view.
