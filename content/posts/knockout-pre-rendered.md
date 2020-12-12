---
title: "Using Knockout with existing HTML content"
date: 2015-04-25
tags:
  - javascript
  - knockout
  - html
url: /2015/04/25/knockout-pre-rendered/
---

When creating an MVVM JavaScript application, [Knockout](http://knockoutjs.com/) is one of the most used libraries. One of its core principles is that all rendering is done client-side. One disadvantage is that this makes [indexing the website harder](https://developers.google.com/webmasters/ajax-crawling/docs/learn-more). Another is that client-side rendering usually takes longer than server-side rendering.

The [Knockout pre-rendered library](https://github.com/ErikSchierboom/knockout-pre-rendered) adds two new binding handlers to Knockout that allow observables to be initialized from pre-rendered HTML content:

- `init`: initialize an observable to a value retrieved from existing HTML content.
- `foreachInit`: wraps the [`foreach` binding](http://knockoutjs.com/documentation/foreach-binding.html), with the observable array's elements bound to existing HTML elements.

## Init binding

Suppose we have the following view model:

```javascript
function ViewModel() {
  this.name = ko.observable();
}
```

Normally, you'd bind the `name` observable to an HTML element as follows:

```html
<span data-bind="text: name">Michael Jordan</span>
```

Once the binding has been applied however, the text within the `<span>` element will be cleared, as the bound observable did not have a value (existing HTML content is ignored).

We can fix this by specifying the `init` binding handler _before_ the `text` binding handler:

```html
<span data-bind="init, text: name">Michael Jordan</span>
```

Now, the text within the `<span>` element is left unchanged. This is due to the `init` binding handler setting the observable's value to the text content of the bound element. As Knockout binding handlers are executed left-to-right, when the `text` binding executes the `init` binding will already have initialized the observable.

You can combine the `init` handler with any binding, as long as you ensure that it is listed before the other bindings:

```html
<span data-bind="init, textInput: name">Michael Jordan</span>

<!-- This binding will use the "value" attribute to initialize the observable -->
<input data-bind="init, value: name" value="Larry Bird" type="text" />
```

### Converting

By default, the `init` binding will set the observable's value to a string. If you want to convert to a different type, you can specify a convert function:

```html
<span data-bind="init: { convert: parseInt }, text: height">198</span>
```

Now, the observable's value will be set to what the `convert` function, with the `innerText` value as its parameter, returns.

#### Custom conversion

It is also possible to use your own, custom conversion function. You could, for example, define it in your view model:

```javascript
function CustomConvertViewModel() {
  this.dateOfBirth = ko.observable();

  this.parseDate = function (innerText) {
    return new Date(Date.parse(innerText));
  };
}
```

You can then use this custom convert function as follows:

```html
<span data-bind="init: { convert: parseDate }, text: dateOfBirth"></span>
```

### Virtual elements

You can also use the `init` binding handler as a virtual element:

```html
<!-- ko init: { field: name } -->Michael Jordan<!-- /ko -->
```

Converting works the same as before:

```html
<!-- ko init: { field: height, convert: parseInt } -->198<!-- /ko -->
```

Note that we now need to explicitly specify the `field` parameter, which points to the observable to initialize. In our previous examples, the `init` binding was able to infer this due to it being combined with the `text`, `textInput` or `value` binding and using the observable they were pointing to. As a consequence, the following bindings are equivalent:

```html
<span data-bind="init, text: name">Michael Jordan</span>
<span data-bind="init: { field: name }, text: name">Michael Jordan</span>
```

Although you'd probably not need it, you could have the `init` handler initialize a different observable than the one `text`, `textInput` or `value` binding uses:

```html
<span data-bind="init: { field: name }, text: year">Michael Jordan</span>
```

The result of this binding is that the `name` observable will have its value initialized to `"Michael Jordan"`, after which the element will be bound to the `year` observable.

## Foreach init binding

This binding handler wraps the `foreach` binding, but instead of creating new HTML elements, it binds to existing HTML elements. Consider the following view model:

```javascript
function PersonViewModel() {
  this.name = ko.observable();
}

function ForeachViewModel() {
  this.persons = ko.observableArray();

  this.persons.push(new PersonViewModel());
  this.persons.push(new PersonViewModel());
  this.persons.push(new PersonViewModel());
}
```

We can bind the elements in the `persons` observable array to existing HTML elements by using the `foreachInit` binding handler:

```html
<ul data-bind="foreachInit: persons">
  <li data-template data-bind="text: name"></li>
  <li data-init data-bind="init, text: name">Michael Jordan</li>
  <li data-init data-bind="init, text: name">Larry Bird</li>
  <li data-init data-bind="init, text: name">Magic Johnson</li>
</ul>
```

There are several things to note:

1. There must be one child element with the `data-template` attribute. This element will be used as the template when new items are added to the array.
2. Elements to be bound to array items must have the `data-init` attribute.
3. You can use the `init` binding handler to initialize the array items themselves.

### Template

You can also use a template that is defined elsewhere on the page:

```html
<ul data-bind="foreachInit: { name: 'personTemplate', data: persons }">
  <li data-init data-bind="init, text: name">Michael Jordan</li>
  <li data-init data-bind="init, text: name">Larry Bird</li>
  <li data-init data-bind="init, text: name">Magic Johnson</li>
</ul>

<script type="text/ko-template" id="personTemplate">
  <li data-template data-bind="text: name"></li>
</script>
```

### Create missing array elements

Up until now, we assumed that the observable array already contained an item for each HTML element bound inside the `foreachInit` section. However, you can also have the `foreachInit` binding handler dynamically add an element for each bound element.

Take the following view model:

```javascript
function ForeachDynamicViewModel() {
  this.persons = ko.observableArray();

  this.addPerson = function () {
    this.persons.push(new PersonViewModel());
  };
}
```

Note that the `persons` observable array does not contain any elements, but that it does have a function to add a new element to the array.

We can use the `foreachInit` binding handler as follows:

```html
<ul
  data-bind="foreachInit: { name: 'personTemplate', createElement: addPerson }"
>
  <li data-template data-bind="text: name"></li>
  <li data-init data-bind="init, text: name">Michael Jordan</li>
  <li data-init data-bind="init, text: name">Larry Bird</li>
  <li data-init data-bind="init, text: name">Magic Johnson</li>
</ul>
```

What happens is that for each element with the `data-init` attribute, the function specified in the `createElement` parameter is called.

## Installation

The best way to install this library is using [Bower](http://bower.io/):

    bower install knockout-pre-rendered

You can also install the library using NPM:

    npm install knockout-pre-rendered --save-dev

The library is also available from a [CDN](https://cdnjs.com/libraries/knockout-pre-rendered).

## Demos

Here are some JSBin demo's that show how to use the binding handlers:

- [`foreachInit` binding](http://jsbin.com/nocaro)
- [`init with text` binding](http://jsbin.com/jazeke)
- [`init with value` binding](http://jsbin.com/xuluye/)
- [`init` binding](http://jsbin.com/wikaji/)

## Acknowledgements

Many thanks go out to [Brian M Hunt](https://github.com/brianmhunt), which [`fastForEach` binding handler](https://github.com/brianmhunt/knockout-fast-foreach) formed the basis of the `foreachInit` binding handler.

## Conclusion

For newly-developed Knockout applications, you probably don't need this library, unless you worry greatly about search engine indexation. However, if you have an existing application that already serves HTML, this library will allow you to integrate Knockout in an easy and non-disruptive way.
