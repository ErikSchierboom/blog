---
title: "Building a JavaScript library - part 4: package managers"
date: 2015-10-05
tags:
  - javascript
  - knockout
  - package-manager
---

This is the fourth in a [series of posts]({{< ref "/posts/building-a-javascript-library" >}}) that discuss the steps taken to publish our library. Our [previous post]({{< ref "/posts/building-a-javascript-library-part-3-open-sourcing" >}}) went into detail on how we open-sourced our library. This post describes how we added support for package managers.

## Package managers

When building software, you'll likely use (and thus depend on) software built by others. Here is how you'd manually add a software dependency to your project:

- Find the website of the software you want to use.
- Search the website for the correct version of the software.
- Download the software.
- Copy the software to your project.
- Possibly run some installation scripts.
- Add the software to version control.

This might not seem that bad, but what if you have many dependencies or want to upgrade to a newer version? You'd have to repeat the previous steps for each dependency, which is both tedious and error prone.

A package manager can help you manage your software dependencies (also referred to as _packages_). It allows you to:

- Search for packages.
- Install and uninstall packages.
- Upgrade packages.
- Restore packages.

Any time you install, uninstall or upgrade a package, the package manager modifies a configuration file, which contains all dependencies.

As the package manager can restore all dependencies using the configuration file, you should only add the configuration file to source control, not the installed dependencies.

### Implementations

There is a package manager for virtually every major language/platform:

- [NuGet](https://www.nuget.org/) (.NET)
- [Maven](https://maven.apache.org/) (Java)
- [RubyGems](https://rubygems.org/) (Ruby)
- [Pip](https://pip.pypa.io/en/stable/) (Python)

As our library was written in JavaScript, we'll add support for its three most popular package managers:

- [NPM](https://www.npmjs.com/)
- [Bower](http://bower.io)
- [JSPM](http://jspm.io/)

### NPM

The first package manager we'll look at is Node's package manager: [NPM](https://www.npmjs.com/). In Node, a project defines its metadata in a file named `package.json`, located in the project's root.

A minimal version of this file only contains the library's name and version:

```json
{
  "name": "knockout-paging",
  "version": "0.2.1"
}
```

There are [many more fields](https://docs.npmjs.com/files/package.json) you can use though:

```json
{
  "name": "knockout-paging",
  "version": "0.2.1",
  "description": "Adds paging functionality to Knockout.",
  "main": "dist/knockout-paging.js",
  "files": ["LICENSE", "README.md", "index.js", "dist"],
  "repository": {
    "type": "git",
    "url": "git@github.com/ErikSchierboom/knockout-paging.git"
  },
  "keywords": ["knockout", "paging"],
  "author": "Erik Schierboom",
  "license": "Apache",
  "bugs": {
    "url": "https://github.com/ErikSchierboom/knockout-paging/issues"
  },
  "homepage": "https://github.com/ErikSchierboom/knockout-paging"
}
```

In general, the more fields you specify, the better. Of special note is the `"files"` field, which specifies what files are copied when the package is installed.

#### Submitting

To be able to submit a project to NPM, we first have to authenticate to NPM:

```bash
npm adduser
```

Just follow the instructions to create an account or login to an existing account.

Then we navigate to the project's root (where the `package.json` file is stored) and do:

```bash
npm publish
```

This command parses the `package.json` file and submits its metadata to NPM. And that's all! Our library has now been published and has its own [page at NPMJS.com](https://www.npmjs.com/package/knockout-paging).

#### Installing

At this point, people can install our library using NPM:

```bash
npm install knockout-paging
```

This will install the library in the `node_modules` directory, ready for use:

<pre>
node_modules
└── knockout_paging
   ├── LICENSE
   ├── README.md
   ├── dist
   |   ├── knockout-paging.js
   |   └── knockout-paging.min.js
   ├── index.js
   ├── node_modules
   |   └── knockout
   |       └── ...
   ├── LICENSE
   └── package.json
</pre>

You can see that the files specified in the `"files"` field in the `package.json` file were copied, along with the `package.json` file itself and the `knockout` package dependency, which is stored in a nested `node_modules` directory.

##### Sources

NPM can install packages from the following sources:

- NPM registry
- Local folder
- Git endpoints

That means we could also have used:

```
npm install git://github.com/ErikSchierboom/knockout-paging.git
```

or

```
npm install /Users/erikschierboom/Programming/knockout-paging
```

to install our library.

#### Updating

If you have made changes to your project and you want to publish the updated version on NPM, you have to do two things:

1. Update the version number in your `package.json` file (using [semantic versioning](http://semver.org/)).
2. Do an `npm publish`.

Again, the `npm publish` command will parse the `package.json` file and update the NPM repository. Note that older versions will remain available.

Note that you can install a specific version using `<package>@<version>`:

```bash
npm install knockout-paging@0.0.1
```

### Bower

The second package manager we'll be supporting is [Bower](http://bower.io), which is a bit different from NPM. Whereas NPM is primarily used to distribute software, Bower is often also used to distribute images, CSS and such.

For our library, we'll use Bower to distribute software, like we did with NPM. We start by creating a file in our project's root named `bower.json`.

A minimal `bower.json` file only contains the name of the library:

```json
{
  "name": "knockout-paging"
}
```

Once again, there are [many more fields](https://github.com/bower/bower.json-spec) you can use:

```json
{
  "name": "knockout-paging",
  "main": "dist/knockout-paging.js",
  "homepage": "https://github.com/ErikSchierboom/knockout-paging",
  "authors": ["Erik Schierboom"],
  "description": "Adds paging functionality to Knockout.",
  "keywords": ["knockout", "foreach", "paging"],
  "license": "Apache",
  "ignore": ["*", "!dist/*"],
  "dependencies": {
    "knockout": "^3.2.0"
  }
}
```

Once again, the more fields you supply, the better. By default, Bower copies all files in your project when installed. However, you can exclude files and folders using the `"ignore"` field.

#### Submitting

Before we can submit our library, we need to look at how Bower handles versioning of packages. Whereas NPM uses the `"version"` field in the `package.json` file, Bower scans the available [Git](https://git-scm.com/book/en/v2/Git-Basics-Tagging) or [SVN](http://svnbook.red-bean.com/en/1.7/svn.branchmerge.tags.html) tags. If those tags are named using [semantic versioning](http://semver.org/), Bower will be able to infer the available versions.

As an example, to create a version `0.0.1` package for our library, we create a tag named "`0.0.1`" using the following commands:

```
# commit files for version
git commit -am "First release"

# tag the commit
git tag -a 0.0.1 -m "Version 0.0.1"

# push commit to GitHub
git push

# push tag to GitHub
git push origin 0.0.1
```

These commands first create a commit, then create a tag named `"0.1.1"` linked to that commit and finally push both the commit and the tag to GitHub.

Having done that, we can now register our library with Bower. First we need to install Bower globally using NPM:

```
npm install -g bower
```

Then all we have to do is:

```
bower register knockout-paging git@github.com/ErikSchierboom/knockout-paging.git
```

This command registers a Bower package named `knockout-paging` located at the specified Git endpoint.

#### Installing

To install the library through Bower, we can now use:

```
bower install knockout-paging
```

When this command executes, Bower does the following things:

- Find the source for the `knockout-paging` package.
- Checkout the source to the `bower_components/knockout-paging` directory.
- Remove all files matching any of the rules in the `"ignore"` field.
- Install all dependencies listed in the `"dependencies"` field.

After the command has completed, it will have created the following files:

<pre>
bower_components
├── knockout
|  └── ...
└── knockout-paging
   ├── bower.json
   └── dist
       ├── knockout-paging.js
       └── knockout-paging.min.js
</pre>

You can see that Bower used the `"ignore"` field to copy everything but the `dist` folder. Note that the `bower.json` is always copied.

If we look at the console output of the `bower install knockout-paging` command, we can clearly see that Bower automatically inferred the package version to `"0.0.1"`, even though we did not specify a version:

```bash
bower cached        git://github.com/ErikSchierboom/knockout-paging.git#0.0.1
bower validate      0.0.1 against git://github.com/ErikSchierboom/knockout-paging.git#*
bower cached        git://github.com/SteveSanderson/knockout.git#3.2.0
bower validate      3.2.0 against git://github.com/SteveSanderson/knockout.git#^3.2.0
bower install       knockout-paging#0.0.1
bower install       knockout#3.2.0

knockout-paging#0.2.1 bower_components/knockout-paging
└── knockout#3.2.0

knockout#3.2.0 bower_components/knockout
```

##### Sources

Bower can install packages from the following sources:

- Bower registry
- Local folders
- Git endpoints
- Subversion endpoints
- URL's

As can be seen, you can also use Bower to install libraries that have not been submitted to Bower. For example, we could also install our library using its Git endpoint:

```
bower install git://github.com/ErikSchierboom/knockout-paging.git
```

or install it from a local folder:

```
bower install /Users/erikschierboom/Programming/knockout-paging
```

#### Updating

To update a package, just create a new tag with the correct semver name:

```
# commit files for new version
git commit -am "Awesome update"

# tag the commit
git tag -a 0.0.2 -m "Version 0.0.2"

# push commit to GitHub
git push

# push tag to GitHub
git push origin 0.0.2
```

Now, when someone tries to install the `knockout-paging` package using Bower, it will automatically install version `0.0.2`.

Note that you can also explicitly specify the version Bower should install using `<package>#<version>`:

```
bower install knockout-paging#0.0.1
```

### JSPM

The third and last package manager we'll add support for is [JSPM](http://jspm.io/), which is the _new kid on the block_. JSPM is a package manager for the [SystemJS universal module loader](https://github.com/systemjs/systemjs), which can load AMD, CommonJS, ES6 or global modules.

#### Submitting

Like NPM, JSPM uses the `package.json` file for describing the package. JSPM [extends the package.json spec](https://github.com/jspm/registry/wiki/Configuring-Packages-for-jspm) with new properties, most notably the following two properties:

- `"registry"`: the registry to use when JSPM install the package. Supported values are: `"jspm"`, `"npm"` and `"github"`.
- `"format"`: the module format used by the package. Its value is either, `"es6"`, `"amd"`, `"cjs"` or `"global"`.

To indicates that we want JSPM to use the NPM registry for our library, which is written as a global module, we modify our library's `package.json` file:

```json
{
  ...
  "registry": "npm",
  "format": "global"
}
```

To allow JSPM to use these fields, we must publish new versions of our library, that include the changed `package.json` file, to NPM and GitHub.

Having done that, the final step is to add our library to the [JSPM registry](http://kasperlewau.github.io/registry/). This is done by adding our library to the [`registry.json`](https://github.com/jspm/registry/blob/master/registry.json) file in the [JSPM repository](https://github.com/jspm/registry).

To do so, first we [fork the JSPM repository](https://github.com/ErikSchierboom/registry). Next, we'll clone our fork to our local machine. There, we'll modify the `registry.json` file to define a JSPM package for our library:

```json
"knockout-paging": "npm:knockout-paging",
```

The key is our package name (which is used to install the package) and the value is a combination of the registry and the registry's source.

Finally, we'll commit this change, push it to our fork on GitHub and send a pull-request. Once the pull-request has been accepted, our library has been [added to the JSPM registry](http://kasperlewau.github.io/registry/#/?q=knockout-paging).

#### Installing

To install our library, we first need to install JSPM globally:

```
npm install -g jspm
```

Once installed, we can do:

```
jspm install knockout-paging
```

This will cause JSPM to search its registry for the source of the `knockout-paging` package (which we set to NPM earlier). You can see this happening if you examine the output of the `jspm install` command we just executed:

```bash
Updating registry cache...
Looking up npm:knockout-paging
Downloading npm:knockout-paging@0.2.2
Looking up npm:knockout
Installed npm:knockout@^3.2.0 (3.3.0)
...
```

Once the command has completed, JSPM will have created the following files:

<pre>
jspm_packages
├── knockout@3.3.0
|  └── ...
├── knockout@3.3.0.js
└── knockout-paging@0.2.2
|  ├── ...
|  └── dist
|      ├── knockout-paging.js
|      └── knockout-paging.min.js
├── knockout-paging@0.2.2.js
└── ...
</pre>

##### Sources

JSPM supports installing packages from the following sources:

- JSPM registry
- NPM registry
- GitHub

As an example, we could install [jQuery](https://jquery.com/) using each of the three sources:

```
jspm install jquery               // Use the JSPM registry

jspm install npm:jquery           // Use the NPM registry

jspm install github:jquery/jquery // Use GitHub
```

#### Updating

To update a package, you just use the registry's specific updating strategy. For NPM, that means updating the `package.json` file followed by a publish to the NPM registry. For GitHub, that means creating a release/tag using the proper semantic versioning name.

## Conclusion

As most software is installed through package managers nowadays, it was vital that our library could be installed using a package manager. We added support for the most popular JavaScript package managers: NPM, Bower and JSPM. Supporting those package managers was easy, it involved not much more than creating and publishing a JSON config file describing the package. When creating and updating packages, be aware that the package managers expect packages to use [semantic versioning](http://semver.org/).

The [next post]({{< ref "/posts/building-a-javascript-library-part-5-build-server" >}}) shows how we enabled build servers to automatically build and test our software.
