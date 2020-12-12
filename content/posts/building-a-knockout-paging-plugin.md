---
title: "Building a Knockout paging plugin"
date: 2015-06-10
tags:
  - javascript
  - knockout
  - paging
---

If you want to add paging to a [Knockout](http://knockoutjs.com/) application, you'll find this is not supported out-of-the-box. This blog post describes how we created a [library](https://github.com/ErikSchierboom/knockout-paging) to add paging support to Knockout.

## Creating the plugin

As our goal is to add paging functionality to Knockout observable arrays, we'll write an [extender](http://knockoutjs.com/documentation/extenders.html).

An extender is a function defined in the `ko.extenders` object that takes the observable to extend as its first parameter and an options parameter as its second. Our basic `paged` extender looks as follows:

```js
ko.extenders.paged = function (target, options) {
  return target;
};
```

Note that our extender returns the observable passed in as the first parameter, as our goal is to add features to an existing observable, not create a new one.

Applying our custom extender to an observable array is easy:

```js
var target = ko.observableArray().extend({ paged: {} });
```

Note that for the moment, we'll just pass an empty object as the options parameter.

### Adding observables

Our second step is to add observables for the current page number and page size to the observable being extended:

```js
ko.extenders.paged = function (target, options) {
  // Add new observables to the existing observable
  target.pageNumber = ko.observable(1);
  target.pageSize = ko.observable(10);

  return target;
};
```

We can now do this:

```js
var target = ko.observableArray().extend({ paged: {} });

target.pageNumber(); // Returns 1
target.pageNumber(3); // Set the page number to 3
target.pageNumber(); // Returns 3

target.pageSize(); // Returns 10
target.pageSize(5); // Set the page size to 5
target.pageSize(); // Returns 5
```

#### Options

In the previous example, we saw that the `pageNumber` and `pageSize` observables were assigned default values. To allow users to customize these default values, we'll use the options parameter.

To support this, we modify our observables as follows:

```js
target.pageNumber = ko.observable((options && options["pageNumber"]) || 1);
target.pageSize = ko.observable((options && options["pageSize"]) || 10);
```

Now we can initialize these observables using the options parameter:

```js
var target = ko
  .observableArray()
  .extend({ paged: { pageNumber: 2, pageSize: 5 } });
target.pageNumber(); // Returns 2
target.pageSize(); // Returns 5
```

If you don't specify any options, the observables are set to their default values:

```js
var target = ko.observableArray().extend({ paged: {} });
target.pageNumber(); // Returns 1
target.pageSize(); // Returns 10
```

### Computed values

The second phase in creating our `paged` extender is to add a property that returns the total number of pages. To be able to calculate this value, we need to know two things:

- The number of items in the array.
- The current page size.

The number of items can be retrieved through the `length` property of the underlying array:

```js
target().length;
```

The current page size is stored in the `pageSize` observable we added earlier.

A naive implementation of the `pageCount` property looks like this:

```js
target.pageCount = ko.observable(
  Math.ceil(target().length / target.pageSize())
);
```

If we ignore the possibility of `pageSize()` returning zero, we would indeed get the correct page count. However, there is a problem with this implementation. Suppose we have the following code:

```js
var target = ko
  .observableArray([1, 2, 3, 4, 5])
  .extend({ paged: { pageSize: 2 } });
target.pageCount(); // Returns 3

// Now change the page size which should cause the page count to change
target.pageSize(3);

target.pageCount(); // Returns ??
```

Can you guess what value the second `pageCount()` call returns? Contrary to what you might expect, the returned value is still `3`. The reason for this is that an observable does not dynamically respond to changes in other observables. For that, Knockout has the concept of [computed observables](http://knockoutjs.com/documentation/computedObservables.html).

Let's convert the `pageCount` property to a computed observable:

```js
target.pageCount = ko.pureComputed(function () {
  return Math.ceil(target.itemCount() / target.pageSize());
});
```

Now, due to the dependency tracking of computed observables, the code works as expected:

```js
var target = ko
  .observableArray([1, 2, 3, 4, 5])
  .extend({ paged: { pageSize: 2 } });
target.pageCount(); // Returns 3

// Now change the page size
target.pageSize(3);

target.pageCount(); // Returns 2
```

The following _read-only_ computed values were also added:

- `pageItems`: the array items on the current page.
- `pageCount`: the total number of pages.
- `itemCount`: the total number of items in the array.
- `firstItemOnPage`: the index (one-based) of the first item on the current page.
- `lastItemOnPage`: the index (one-based) of the last item on the current page.
- `hasPreviousPage`: indicates if there is a previous page.
- `hasNextPage`: indicates if there is a next page.
- `isFirstPage`: indicates if the current page is the first page.
- `isLastPage`: indicates if the current page is the last page.
- `pages`: an array of pages.

The following example shows how these values can be used:

```js
var target = ko.observableArray([2, 3, 5, 9, 11]);
target.extend({ paged: { pageSize: 2 } });

target.pageNumber(); // Returns 1
target.pageSize(); // Returns 2
target.pageItems(); // Returns [2, 3]
target.pageCount(); // Returns 3
target.itemCount(); // Returns 5
target.firstItemOnPage(); // Returns 1
target.lastItemOnPage(); // Returns 2
target.hasPreviousPage(); // Returns false
target.hasNextPage(); // Returns true
target.isFirstPage(); // Returns true
target.isLastPage(); // Returns false
target.pages(); // Returns [1, 2, 3]
```

### Functions

To make navigating easier, we add the following functions:

- `toNextPage`: move to the next page (if there is one).
- `toPreviousPage`: move to the previous page (if there is one).
- `toFirstPage`: move to the first page.
- `toLastPage`: move to the last page.

Their implementation is straightforward; this is the implementation of the `toNextPage` function:

```js
target.toNextPage = function () {
  if (target.hasNextPage()) {
    target.pageNumber(target.pageNumber() + 1);
  }
};
```

This example shows how to use these functions:

```js
var target = ko.observableArray([2, 3, 5, 9, 11]);
target.extend({ paged: { pageSize: 2 } }); // page number is 1

target.toNextPage(); // page number becomes 2
target.toLastPage(); // page number becomes 3
target.toNextPage(); // page number remains 3
target.toPreviousPage(); // page number becomes 2
target.toFirstPage(); // page number becomes 1
```

### Page generators

As we saw earlier, the `pages` computed observable returns an array of page numbers. The most simple implementation justs return all pages:

```js
target.pages = ko.pureComputed(function () {
  var pageNumbers = [];

  for (var i = 1; i <= target.pageCount(); i++) {
    pageNumbers.push(i);
  }

  return pageNumbers;
});
```

Note that we define it as a computed value, as it depends on the `pageCount` observable.

Although this implementation is fine for a small number of pages, it becomes unwieldy when there are many pages. In that case, a sliding window of pages, containing the current page and some pages surrounding it, is a better option:

```js
target.pages = ko.pureComputed(function () {
  var windowSize = 5;
  var leftBasedStartIndex = target.pageNumber() - Math.floor(windowSize / 2),
    rightBasedStartIndex = target.pageCount() - windowSize + 1,
    startIndex = Math.max(
      1,
      Math.min(leftBasedStartIndex, rightBasedStartIndex)
    ),
    stopIndex = Math.min(target.pageCount(), startIndex + windowSize - 1);

  var pageNumbers = [];

  for (var i = startIndex; i <= stopIndex; i++) {
    pageNumbers.push(i);
  }

  return pageNumbers;
});
```

The following example show how the sliding window works:

```js
var input = [
  1,
  2,
  3,
  4,
  5,
  6,
  7,
  8,
  9,
  10,
  11,
  12,
  13,
  14,
  15,
  16,
  17,
  18,
  19,
  20,
];

var sliding = ko.observableArray(input).extend({ paged: { pageSize: 2 } });

// Return the pages around the current page (1)
sliding.pages(); // Returns [1, 2, 3, 4, 5]

// Changing the page number changes the returned pages
sliding.pageNumber(7);

// Return the pages around the current page (7)
sliding.pages(); // Returns [5, 6, 7, 8, 9]
```

### Custom page generators

But what if someone wants to supply their own page generator? Ideally, you should be able to just _plugin_ your own page generator (relevant: [open/closed principle](http://en.wikipedia.org/wiki/Open/closed_principle)). The first step towards realizing this is defining what the signature of a page generator is. Let's define it as a function that takes a paged observable and returns an array of pages.

We can rewrite our simple page generator to conform to this definition:

```js
simplePageGenerator = function (pagedObservable) {
  var pageNumbers = [];

  for (var i = 1; i <= target.pageCount(); i++) {
    pageNumbers.push(i);
  }

  return pageNumbers;
};
```

The next step is to modify our extender to be able to plugin a page generator. There are lots of ways to do this. Taking a cue from how extenders are defined in the `ko.extenders` object, we'll define page generators in the `ko.paging.generators` object:

```js
ko.paging.generators["simple"] = function (pagedObservable) {
  var pageNumbers = [];

  for (var i = 1; i <= pagedObservable.pageCount(); i++) {
    pageNumbers.push(i);
  }

  return pageNumbers;
};

// We omit the 'sliding' page generator for brevity
```

We then modify our paged extender to allow the page generator to be specified in the options value:

```js
var pageGeneratorName = (options && options["pageGenerator"]) || "simple";

target.pageGenerator = ko.paging.generators[pageGeneratorName];
```

The final step is to have the `pages` computed value forward calls to the current page generator:

```js
target.pages = ko.pureComputed(function () {
  return target.pageGenerator(target);
});
```

We are now ready to plugin a custom page generator. First we add it to the `ko.paging.generators` object:

```js
ko.paging.generators["custom"] = function (pagedObservable) {
  return [42]; // Always return page number 42
};
```

We can then use it as follows:

```js
var custom = ko
  .observableArray([])
  .extend({ paged: { pageGenerator: "custom" } });
custom.pages(); // Returns [42]
```

#### Customizing page generators

The observant among you might have noticed a problem with the sliding window page generator: the window size could not be changed. This is due to how we defined page generators: as a simple function. To allow customization of page generators, we re-define a page generator as an object with a `generate` function that takes a paged observable and returns an array of pages.

Rewriting the sliding page generator to the new format is easy:

```js
ko.paging.generators['sliding'] = {
  this.windowSize = ko.observable(5);

  this.generate = function(pagedObservable) {
     // Same implementation as before
  }
}
```

The next step is to modify our `pages` computed value to call the `generate` function:

```js
target.pages = ko.pureComputed(function () {
  return target.pageGenerator.generate(target);
});
```

Now we are able to change the window size:

```js
var input = [
  1,
  2,
  3,
  4,
  5,
  6,
  7,
  8,
  9,
  10,
  11,
  12,
  13,
  14,
  15,
  16,
  17,
  18,
  19,
  20,
];

var sliding = ko.observableArray(input).extend({ paged: { pageSize: 2 } });

// Window size is 5 (default)
sliding.pages(); // Returns [1, 2, 3, 4, 5]

// Change the window size
ko.paging.generators["sliding"].windowSize(3);

// Window size is 3
sliding.pages(); // Returns [1, 2, 3]
```

## Installation

The best way to install this library is using [Bower](http://bower.io/):

    bower install knockout-paging

You can also install the library using NPM:

    npm install knockout-paging --save-dev

The library is also available from a [CDN](https://cdnjs.com/libraries/knockout-paging).

## Demo

There is also a [JS Bin demo](http://output.jsbin.com/liruyo/) that shows how the library can be used.

## Conclusion

If you want to do paging in Knockout, the [`paged`](https://github.com/ErikSchierboom/knockout-paging) extender can help you. Building the extender was not very hard, due to the easy, well-documented way in which Knockout can be extended.
