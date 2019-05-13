---
title: "Building a JavaScript library - part 3: open-sourcing"
date: 2015-08-22
tags: 
  - JavaScript
  - Knockout
  - Open-source
---

This is the third in a [series of posts]({{< ref "/posts/building-a-javascript-library" >}}) that discuss the steps taken to publish our library. Our [previous post]({{< ref "/posts/building-a-javascript-library-part-2-automation" >}}) showed how we automated repetitive tasks. This post describes how we open-sourced our library.

## Why open-source?

Before we'll describe *how* we made our library open-source, let's first ask: *why* ? What exactly are the benefits of making code open-source?

As it turns out, open-sourcing code has several advantages:

- You don't have to do everything yourself. People all over the world can contribute (e.g. [node.js](https://github.com/nodejs/node) has over 700 contributors). This means the software can be developed faster and cheaper.
- You benefit from the knowledge of others. They will find bugs you didn't and come up with new ideas. Furthermore, they will help you improve your code, making you a better programmer.
- Your code becomes part of your CV. Not only does it demonstrate your skills as a programmer, it also implies you are passionate about developing software. Recruiters also actively scan open-source projects for potential candidates.
- You use open-source software everyday (even without knowing it), why not give back to the community? It is the right thing to do. 
- It forces you to think on how to setup your project. If the code is difficult to work with or hard to build, you won't find many people helping you.
- It is fun! Collaboratively developing software is one of my greatest joys in programming.

Of course there are also disadvantages to open-sourcing your code:

- People expect you to actively maintain your code. Not only will they expect you to fix bugs, but also to add new features.
- You can mistakenly publish sensitive data (more on that later).
- You can get into license trouble (more on this later too).
- Monetizing your software becomes harder.

We feel the advantages overwhelmingly outweigh the disadvantages, which is why we chose to open-source our library.

## Preparing for open-sourcing code

Before we'll actually open-source our code, there are some things left to do.

### Write documentation 

Although open-source code allows people to inspect the code to find out how it works, you *should* still write documentation. If you are not convinced of the benefits of writing documentation, try using a new library without reading any of its documentation. ;)

The documentation should at least specify:

- What functionality the library offers.
- How to use the library, preferrably using examples showing common use cases.
- How to install the library.

For open-source projects, the documentation should also contain:

- The license the code is released under.
- A description on how people can contribute.

A `README` file containing this minimal documentation should be stored alongside your source code for easy access. Note that this documentation file should be relatively small, people should be able to read it quickly. 

#### Separate documentation website

If you feel that the minimal `README` file cannot describe your library in enough detail, you should consider creating a separate documentation website. The [Hangfire](https://github.com/HangfireIO/Hangfire) library does precisely this, with a compact [README](https://github.com/HangfireIO/Hangfire/blob/master/README.md) to get you started but also a separate [documentation website](http://docs.hangfire.io/en/latest/) that goes into far more detail.

#### Documentation tools

There are many tools for writing documentation, such as [Jekyll](http://jekyllrb.com/) (used on [getbootstrap.com](http://getbootstrap.com/)) and [Sphinx](http://sphinx-doc.org/) (used on [docs.asp.net](http://docs.asp.net/en/latest/)). One benefit of Sphinx is that [readthedocs.org](https://readthedocs.org/) provides free hosting for your documentation!

To make things really open, you can also open-source your documentation! This is done by the aforementioned [docs.asp.net](http://docs.asp.net/en/latest/), which contents are [hosted on GitHub](https://github.com/aspnet/Docs).

Note: some people use documentation generators like [JSDoc](https://github.com/jsdoc3/jsdoc) or [Doxygen](http://www.stack.nl/~dimitri/doxygen/), which generate documentation by parsing source code. While this is a convenient way to quickly "write" some documentation, it should only be used in addition to hand-written documentation.

#### Demos

A great way to show how to use your library is to create one or more demos. There are several websites where you can create HTML demo applications for free, but we particularly like:

- [JS Bin](http://jsbin.com/)
- [JSFiddle](http://jsfiddle.net/)

Of course, you should link to any demo applications from your `README`.

Note: there are also "Fiddle" sites for other languages/platforms:

- [.NET fiddle](https://dotnetfiddle.net/) (C#, F# and VB)
- [Scala fiddle](http://scalafiddle.net/console)
- [SQL fiddle](http://sqlfiddle.com/)
- [Regex fiddle](http://refiddle.com/)

You can find a comprehensive list of "Fiddle" sites at [fiddles.io](https://fiddles.io/).

### Choosing a license

When code is open-source, it is important for other people to know what they are allowed to do with it. This is defined by a license. If you need help choosing a license, check out [choosealicense.com](http://choosealicense.com/). 

For our library, we chose the [Apache 2 license](http://choosealicense.com/licenses/apache-2.0/), which lets other people freely use our code. The main difference with the popular [MIT license](http://choosealicense.com/licenses/mit/) is that the Apache license has better protection against patent issues. You should probably also choose the Apache license if you want to [raise venture capital](http://tomtunguz.com/open-source-exits-by-license/). 

 You should mention the license in both your source code and the `README` file, as well as include it in a separate [LICENSE](https://github.com/ErikSchierboom/knockout-paging/blob/master/LICENSE) file.

### Cleaning up

At this point, you should clean-up your code. As you are making your code public, why not make it as nice as you can? Note that if you have written tests, you should be able to safely refactor your code.

Having cleaned-up code is also in your best interest, as people are less likely to contribute code to your library if the code is hard to read or maintain.

#### Removing sensitive data

As part of the clean-up, please make sure that your code does not contain sensitive data like SSH keys or passwords. Even though GitHub has a page dedicated to [dealing with sensitive data](https://help.github.com/articles/remove-sensitive-data/), a simple [GitHub search](https://github.com/search?p=2&q=rackspace_api_key&ref=searchresults&type=Code) shows that many repositories still contain sensitive data.

Due to the distributed nature of open-source software (anyone could have a copy of your code), published sensitive data *must* be considered compromised, so make sure to remove sensitive data from your code *before* open-sourcing it.

### Hosting

Our last task is to decide where to host our code. As we use [Git](https://git-scm.com/) as our version control system, we are looking for websites that can host Git code. Luckily, there are several such websites:

- [GitHub](https://github.com/)
- [Bitbucket](https://bitbucket.org/)
- [Codeplex](http://www.codeplex.com/)
- [Google Code](https://code.google.com/) (will be shutdown in 2016)

Hosting open-source code is completely free on all four websites. We chose GitHub mainly for its great collaboration options:

- Contributing code through through [pull-requests](https://help.github.com/articles/using-pull-requests/).
- Keeping track of issues through the built-in [issue tracker](https://help.github.com/articles/about-issues/). 

There are other benefits of using GitHub:

- As most open-source projects are hosted on GitHub, most open-source contributors are also on GitHub.
- An easy to setup [wiki](https://help.github.com/articles/about-github-wikis/).
- [Free website hosting](https://pages.github.com/).

Having chosen GitHub to host our code, we then created a [repository](https://github.com/ErikSchierboom/knockout-paging) for our library's code. If you need help creating a GitHub repository, check out this [tutorial](https://help.github.com/articles/create-a-repo/).

### Publishing

Finally, we are ready to open-source our code! As our code already was in a local Git repository, we just add our GitHub repository's URL as a remote and push to it:

```
# Link our local Git repository
git remote add origin git@github.com:ErikSchierboom/knockout-paging.git

# Push local commits to GitHub
git push origin master
```

And with that, [our code](https://github.com/ErikSchierboom/knockout-paging) has been open-sourced.

## Conclusion

Open-sourcing code is easy: just pick an online host and push your code to it. However, to create a great open-source project, you should write documentation, choose a license and clean-up your code. Although these tasks may seem like chores, they will give your project a much better chance at reaping the main benefit of open-sourcing code: getting help from others.

Our [next post]({{< ref "/posts/building-a-javascript-library-part-4-package-managers" >}}) examines how we allowed our library to be installed through package managers.