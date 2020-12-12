---
title: "Building a JavaScript library - part 2: automation"
date: 2015-08-12
tags:
  - javascript
  - knockout
  - automation
---

This is the second in a [series of posts]({{< ref "/posts/building-a-javascript-library" >}}) that discuss the steps taken to publish our library. The [previous post]({{< ref "/posts/building-a-javascript-library-part-1-testing" >}}) showed how we tested our library. In this post, we'll look at how to automate repetitive tasks.

## Task automation

Developing a library is not just about writing code. We also have the following repetitive tasks:

- Running tests.
- Creating a distribution version of our library.
- Creating a minified version of our distribution file.
- Setting the correct version number in our distribution file.
- Incrementing version numbers.

All these repetitive tasks can be done manually, but we'll automate them. Automated tasks have several advantages over manual tasks:

- They run faster.
- They run reliably.
- They can be scheduled.

To define and run automated tasks, we'll use a tool called a task runner. At the moment, the two most popular ones are [Grunt](http://gruntjs.com/) and [Gulp](http://gulpjs.com/). Their most striking difference is that Grunt uses configuration files to define tasks and Gulp uses code.

Internally, Grunt works with files whereas Gulp work with streams. Using streams means that Gulp does less writing to disk and allows tasks to be [chained efficiently](http://blog.carbonfive.com/2014/05/05/roll-your-own-asset-pipeline-with-gulp/), which improves performance.

For our library, we'll use Gulp as we prefer doing our configuration in code.

### Installing Gulp

As Gulp runs on [node.js](https://nodejs.org/), the first step is to [install node.js](https://nodejs.org/download/). Once that is done, we can use the [Node Package Manager](https://docs.npmjs.com/getting-started/what-is-npm) (NPM) to install [Gulp](https://www.npmjs.com/package/gulp):

```
npm install --global gulp
```

Note that we install Gulp globally, which allows us to run Gulp everywhere.

Next we install Gulp as a development dependency of our project:

```
npm install --save-dev gulp
```

Our final step is to create a `gulpfile.js` file in our project's root in which our tasks are defined:

```js
var gulp = require("gulp");

gulp.task("default", function () {
  // place code for your default task here
});
```

Here, we just define an empty task named `"default"`.

To run a task, just call `gulp <taskname>`, which in our case means:

```
gulp default
```

As `"default"` is, well, the default, we could have also used:

```
gulp
```

Now we're ready to start creating tasks.

### Running tests

As you [might recall]({{< ref "/posts/building-a-javascript-library-part-1-testing" >}}), running our tests required us to open the test runner file in a web-browser. To run our tests using Gulp, being a command-line application, we need to use a _headless_ browser, which is a browser without a GUI that can be run from the command-line.

We'll use the [PhantomJS](http://phantomjs.org/) _headless_ browser using the [gulp-mocha-phantomjs](https://www.npmjs.com/package/gulp-mocha-phantomjs) plugin:

```
npm install gulp-mocha-phantomjs --save-dev
```

Next, we create a `"test"` task that uses PhantomJS to open our test runner file:

```js
var mochaPhantomJS = require("gulp-mocha-phantomjs");

gulp.task("test", function () {
  return gulp.src("runner.html").pipe(mochaPhantomJS());
});
```

If we run `gulp test`, the following will be printed to the screen:

```bash
D:\Programming\knockout-paging>gulp test
[14:02:54] Using gulpfile D:\Programmeren\knockout-paging\gulpfile.js
[14:02:54] Starting 'test'...

  paged extender
    V pageCount on empty paged observable array is 1

  1 passing (6ms)

[14:02:59] Finished 'test' after 4.8 s
```

You can see that phantomJS has opened our test runner file and written its output to the console. But what happens when one of the tests fail? Well, you'll see something like this:

```bash
D:\Programming\knockout-paging>gulp test
[14:07:05] Using gulpfile D:\Programmeren\knockout-paging\gulpfile.js
[14:07:05] Starting 'test'...

  paged extender
    1) pageCount on empty paged observable array is 1

  0 passing (7ms)
  1 failing

  1) paged extender pageCount on empty paged observable array is 1:

      AssertionError: expected 1 to equal 329
      + expected - actual

      +329
      -1

[14:07:09] 'test' errored after 4.79 s
[14:07:09] Error in plugin 'gulp-mocha-phantomjs'
test failed
```

The test task now outputs the failed assertion and an error message: `Error in plugin 'gulp-mocha-phantomjs'`. This is Gulp warning us that the `'gulp-mocha-phantomjs'` task returned an error code, which it does when one or more tests fail.

You can use this error returning feature to define tasks that only succeed when all tests pass, ideal for continuous integration scenarios. For example, we can define a `"integration"` task that depends on the `"test"` task to run successfully:

```js
gulp.task("integration", ["test"], function () {
  // This will only execute when the 'test' task is successful
});
```

### Creating a distribution file

Our library is developed in the `index.js` file in our root folder. However, it is customary to place the files you want to distribute in a separate folder. We'll use the `dist` folder for that. What we want is for Gulp to copy our `index.js` file to `dist/knockout-paging.js`. For that we'll need the [gulp-rename](https://www.npmjs.com/package/gulp-rename) plugin:

```
npm install gulp-rename --save-dev
```

The actual Gulp task is quite simple:

```js
gulp.task("dist", function () {
  gulp
    .src("./index.js")
    .pipe(plugins.rename("knockout-paging.js"))
    .pipe(gulp.dest("./dist"));
});
```

When we run `gulp dist`, the `index.js` file's contents will be copied to `dist/knockout-paging.js`.

### Creating minified distribution file

In the previous step, we created a distribution file for our library, which was not minified. As it is good practice to also distribute a minified version of your library, we'll create a task for this. For that, we'll use the [gulp-uglify](https://www.npmjs.com/package/gulp-uglify) plugin:

```
npm install gulp-uglify --save-dev
```

Applying minification is just a matter of piping our library file to `plugins.uglify()`:

```js
gulp.task("dist-minified", function () {
  gulp
    .src("./index.js")
    .pipe(plugins.uglify())
    .pipe(plugins.rename("knockout-paging.min.js"))
    .pipe(gulp.dest("./dist"));
});
```

Now when we run `gulp dist-minified`, a minified version of the `index.js` file is written to `dist/knockout-paging.min.js`. Note that the `index.js` file itself is _not_ modified.

With Gulp, you can chain commands. This allows us to combine the previous two tasks into a single task:

```js
gulp.task('dist-combined', function () {
  gulp.src('./index.js')
      .pipe(plugins.rename('knockout-paging.js'))
      .pipe(gulp.dest('./dist'));
      .pipe(plugins.uglify())
      .pipe(plugins.rename('knockout-paging.min.js'))
      .pipe(gulp.dest('./dist'));
});
```

Running `gulp dist-combined` will create both the unmodified `dist/knockout-paging.js` and minified `dist/knockout-paging.min.js` files, once again not modifying the `index.js` file.

### Setting the version number

When we distribute our library, the distribution file's header should contain its version number. We could manually add it to our distribution file each time we release a new version, but let's automate it.

First, we install the [gulp-replace](https://www.npmjs.com/package/gulp-replace) package:

```
npm install gulp-replace --save-dev
```

Then we add a placeholder for the version number in our `index.js` file's header:

```js
/*!
  Knockout paged extender {{ "{{ version " }}}}
  License: Apache 2
*/
```

To automatically replace the `"{{ "{{ version " }}}}"` placeholder with the current version number, we retrieve the version number specified in our `package.json` file and use the `replace` plugin we just installed to do the replacing:

```js
gulp.task("dist-version", function () {
  var pkg = require("./package.json");

  gulp
    .src("./index.js")
    .pipe(plugins.replace('{{ "{{ version " }}}}', pkg.version))
    .pipe(plugins.rename("knockout-paging.js"))
    .pipe(gulp.dest("./dist"))
    .pipe(plugins.uglify())
    .pipe(plugins.rename("knockout-paging.min.js"))
    .pipe(gulp.dest("./dist"));
});
```

Now suppose that our `package.json` looks like this:

```js
{
  "name": "knockout-paging",
  "version": "0.2.1",
  ...
}
```

If we run `gulp dist-version`, the `dist/knockout-paging.js` file's header will contain the current version number:

```js
/*!
  Knockout paged extender 0.2.1
  License: Apache 2
*/
```

Note that the minified file does not contain the header, as the minification process removes comments.

### Incrementing version numbers

With more and more libraries using [semantic version](http://semver.org/) (semver), managing the current version number of your library is an important task. In semver, a version number is comprised of three parts: `<major>.<minor>.<patch>`. When you want to do a new release, semver defines which part of the version number to increment:

- Major: incompatible API changes.
- Minor: functionality added in a backwards-compatible manner.
- Patch: bug fixes (also backwards-compatible).

Using `1.2.5` as our current version number, here's how the version number should be modified for each change type:

- Major: `2.0.0`
- Minor: `1.3.0`
- Patch: `1.2.6`

The main benefit of semver is that version numbers now have meaning. For example, if only the patch part of a version number has changed, you should be able to update safely.

We can automate these version number changes using [gulp-bump](https://www.npmjs.com/package/gulp-bump):

```
npm install gulp-bump --save-dev
```

We then define three tasks, for each of the three semver change types:

```js
function updateVersion(importance) {
  return gulp
    .src("./package.json")
    .pipe(plugins.bump({ type: importance }))
    .pipe(gulp.dest("./"));
}

gulp.task("patch-release", function () {
  return updateVersion("patch");
});
gulp.task("minor-release", function () {
  return updateVersion("minor");
});
gulp.task("major-release", function () {
  return updateVersion("major");
});
```

Now if we run `gulp major-release`, the version number in our `package.json` file will be updated according to the rules defined for major changes.

#### Updating in multiple files

Our previous example only modified the `package.json` file, but you can also update multiple files by passing an array of filenames:

```js
function updateVersion(importance) {
  return gulp
    .src(["./package.json", "./bower.json"])
    .pipe(plugins.bump({ type: importance }))
    .pipe(gulp.dest("./"));
}
```

This modified task updates the version in both the `package.json` and `bower.json` files.

## Conclusion

Usually, maintaining a library means doing repetitive tasks, like minifying files or running tests. For our library, we automated these tasks using Gulp. Defining these tasks was easy, as we could just use already available tasks. Automating our tasks not only saved us time, but also prevented us from making mistakes.

In the [next post]({{< ref "/posts/building-a-javascript-library-part-3-open-sourcing" >}}), we'll look at how we open-source our code.
