---
title: "Building a JavaScript library - part 6: TypeScript"
date: 2015-11-04
tags:
  - javascript
  - knockout
  - typescript
---

This is the sixth in a [series of posts]({{< ref "/posts/building-a-javascript-library" >}}) that discuss the steps taken to publish our library. In our [previous post]({{< ref "/posts/building-a-javascript-library-part-5-build-server" >}}), we used build servers to automatically build and test our software. This post will show how we added TypeScript support to our library.

## TypeScript

The [TypeScript](http://www.typescriptlang.org/) language is a typed superset of JavaScript that adds features like modules, classes and interfaces. The great thing of it being a JavaScript superset is that any JavaScript code is also valid TypeScript code! This allows you to gradually introduce TypeScript-specific features to your existing JavaScript code.

Let's look at some TypeScript code:

```javascript
class Greeter {
  greeting: string;
  constructor(message: string) {
    this.greeting = message;
  }
  greet() {
    return "Hello, " + this.greeting;
  }
}

var greeter = new Greeter("world");
alert(greeter.greet());
```

As can be seen, TypeScript allows us to use features like classes and constructors. However, this code is not valid JavaScript code (it might be in the future). Therefore, we'll use the TypeScript compiler to convert to plain JavaScript. Our example compiles to the following JavaScript code:

```javascript
var Greeter = (function () {
  function Greeter(message) {
    this.greeting = message;
  }
  Greeter.prototype.greet = function () {
    return "Hello, " + this.greeting;
  };
  return Greeter;
})();
var greeter = new Greeter("world");
alert(greeter.greet());
```

The compiled output is valid JavaScript code, which is quite similar to the TypeScript source. One thing that is lost completely in the translation though, are the type annotations. So what's that about?

## Static typing

One of TypeScript's best features is that it allows you to add type annotations to your code. This makes TypeScript statically typed, as opposed to JavaScript being dynamically typed. As JavaScript does not support type annotations, the TypeScript compiler does not include them in the compiled JavaScript. This is known as [type erasure](https://en.wikipedia.org/wiki/Type_erasure).

Regardless of your stance on dynamic vs. static typing, the latter has some benefits:

1. Bugs can be found at compile-time instead of runtime.
2. You can (more) safely refactor code.
3. Tooling can easily support code-completion.

You might not miss these features in small projects, but in large projects they can greatly enhance productivity. That is why the Angular team chose to write Angular 2 [completely in TypeScript](http://blogs.msdn.com/b/typescript/archive/2015/03/05/angular-2-0-built-on-typescript.aspx).

## Declaration files

So how does TypeScript interact with plain JavaScript code, which doesn't have type annotations or classes? Well, thanks to [declaration files](https://typescript.codeplex.com/wikipage?title=Writing%20Definition%20%28.d.ts%29%20Files), you can use them as if they were written in TypeScript.

A declaration file is a TypeScript file, but with only types and variable definitions, the actual implementation is done in another (JavaScript) file. Let's consider the following JavaScript code:

```javascript
var Rectangle = (function () {
  function Rectangle(width, height) {
    this.width = width;
    this.height = height;
  }
  Rectangle.prototype.createSquare = function (size) {
    return new Rectangle(size, size);
  };
  return Rectangle;
})();
```

We could use this code from TypeScript as is, but we wouldn't have any type information and could thus easily use it incorrectly. We can remedy this by specifying the types in a declaration file:

```javascript
class Rectangle {
  width: number;
  height: number;

  constructor(width: number, height: number) {
    this.width = width;
    this.height = height;
  }

  createSquare(size: number) {
    return new Rectangle(size, size);
  }
}
```

If we reference this declaration file in our TypeScript code, we can then safely use the JavaScript code.

## Creating a declaration file

Now that we know what declaration files are, let's create one for our library. Declaration files must have a `.d.ts` extension, so lets name our library's declaration file `knockout-paging.d.ts`. As our library extends the [Knockout](http://knockoutjs.com/) library, we start by downloading its [declaration file](https://raw.githubusercontent.com/borisyankov/DefinitelyTyped/2d4c4679bc2b509f27435a4f9da5e2de11258571/knockout/knockout.d.ts). We'll reference this file in our declaration file to import its types.

We are now ready to define our declaration file. First, we'll define an interface for our paged observable array:

```js
/// <reference path="knockout.d.ts" />

interface KnockoutPagedObservableArray<T> extends KnockoutObservableArray<T> {
  pageSize: KnockoutObservable<number>;
  pageNumber: KnockoutObservable<number>;

  pageItems: KnockoutComputed<T[]>;
  pageCount: KnockoutComputed<number>;
  itemCount: KnockoutComputed<number>;
  firstItemOnPage: KnockoutComputed<number>;
  lastItemOnPage: KnockoutComputed<number>;
  hasPreviousPage: KnockoutComputed<boolean>;
  hasNextPage: KnockoutComputed<boolean>;
  isFirstPage: KnockoutComputed<boolean>;
  isLastPage: KnockoutComputed<boolean>;
  pages: KnockoutComputed<number[]>;

  toNextPage(): void;
  toPreviousPage(): void;
  toLastPage(): void;
  toFirstPage(): void;
}
```

As our paged observable array is a regular observable array with added properties and functions, our interface extends Knockout's `KnockoutObservableArray<T>` type. This type is defined in the previously downloaded `knockout.d.ts` file. To use the types in this declaration file, we reference it in our own declaration using the `/// <reference path="..." />` syntax.

To define our `ko.pagedObservableArray()` function, we'll have to extend the existing `ko` instance's type, which is the `KnockoutStatic` interface. Luckily, extending an interface is as simple as defining a new interface with the same name:

```js
interface KnockoutStatic {
  pagedObservableArray<T>(
    value?: T[],
    options?: KnockoutPagedOptions
  ): KnockoutPagedObservableArray<T>;
}

interface KnockoutPagedOptions {
  pageSize?: number;
  pageNumber?: number;
  pageGenerator?: string;
}
```

Here, we specify that the `ko` instance has a `pagedObservableArray()` function that takes two optional parameters. As the second parameter is actually an object, we define its allowed properties in a separate interface.

### Testing the declaration file

To test our declaration file, we can create a new TypeScript files that references our declaration file. We should then use our library in every supported way, checking to see if our declaration file allows it. For our library, this looks something like this:

```js
/// <reference path="knockout-paging.d.ts" />

// Different option formats
var emptyOptions = {};
var allOptions = {
  pageNumber: 2,
  pageSize: 10,
  pageGenerator: "sliding",
};

function pagedObservableArray() {
  var simple = ko.pagedObservableArray();
  var emptyOptions = ko.pagedObservableArray([1, 2, 3], emptyOptions);
  var allOptions = ko.pagedObservableArray([1, 2, 3], allOptions);
}

function observables() {
  var paged = ko.pagedObservableArray([]);
  var pageSize = paged.pageSize();
  var pageNumber = paged.pageNumber();
}

function computed() {
  var paged = ko.pagedObservableArray([]);
  var firstItemOnPage = paged.firstItemOnPage();
  var hasPreviousPage = paged.hasPreviousPage();
  var pages = paged.pages();
}

function functions() {
  var paged = ko.pagedObservableArray([]);
  paged.toNextPage();
  paged.toLastPage();
}
```

We can then try to compile this test file using the TypeScript compiler. It should build without errors or warnings.

Note that for brevity, we left out some tests.

## Including the declaration file

To make our declaration file available, we simply add it to our repository. People can then use it by referencing it from their TypeScript code.

Starting from version 1.6, the TypeScript compiler can [automatically load declaration files](https://github.com/Microsoft/TypeScript/wiki/Typings-for-npm-packages) (without explicitly referencing them). To support this, we'll add a `"typings"` property to our `package.json` file:

```json
"typings": "./knockout-paging.d.ts",
```

The declaration file will now automatically be picked up by the TypeScript compiler.

## Publishing the declaration file

An alternative place where people look for declaration files is the [DefinitelyTyped repository](https://github.com/borisyankov/DefinitelyTyped). This repository contains many declaration files, but mostly for libraries that don't provide a declaration file themselves.

Although our library' _does_ provide a declaration file, it's not a bad idea to also submit it to DefinitelyTyped. To do so, we just follow the [contribution guidelines](http://definitelytyped.org/guides/contributing.html):

1. We [fork](https://help.github.com/articles/fork-a-repo/) the [DefinitelyTyped repository](https://github.com/borisyankov/DefinitelyTyped).
2. In the fork, we create a folder with our library's name.
3. We add the declaration- and tests file to that folder.
4. We compile our tests file to see if everything is valid.
5. We commit our changes and submit a [pull request](https://help.github.com/articles/using-pull-requests/).

Once the pull request has been accepted, our declaration file will have been added to the DefinitelyType repository.

Note that if we update our declaration file, we should also update it in the DefinitelyTyped repository.

### Installing declaration files

Although you could manually search and download declaration files from the DefinitelyTyped repository, you can also use the [TSD](http://definitelytyped.org/tsd/) tool. To install it, we use NPM:

```
npm install tsd -g
```

We can now use the `tsd` command to install declaration files. Here is how we'd install our library's declaration file:

```
tsd install knockout-paging --save
```

Once this command has completed, our library's declaration file will have been saved in `typings/knockout-paging/knockout-paging.d.ts`.

When TSD executes a command, it modifies the `tsd.json` file. This file contains metadata used by TSD:

```json
{
  "version": "v4",
  "repo": "borisyankov/DefinitelyTyped",
  "ref": "master",
  "path": "typings",
  "bundle": "typings/tsd.d.ts",
  "installed": {
    "knockout-paging/knockout-paging.d.ts": {
      "commit": "001ca36ba58cef903c4c063555afb07bbc36bb58"
    }
  }
}
```

The most important part is the `"installed"` section, which lists all installed declaration files. This allows TSD to install all typings file the project depends on just by examining the `tsd.json` file, similar to how the `dependencies` section in a `package.json` file is used by NPM to install any dependencies.

## Conclusion

As TypeScript is becoming more popular, we created a declaration file for our library. Creating this file was fairly straightforward and gives users a type-safe way to interact with our library. Besides adding the declaration file to our repository, we also added it to the DefinitelyTyped repository.

In the [next post]({{< ref "/posts/building-a-javascript-library-part-7-cdn" >}}) we'll add our library to a CDN.
