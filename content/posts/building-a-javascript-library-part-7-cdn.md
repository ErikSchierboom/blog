---
title: "Building a JavaScript library - part 7: CDN"
date: 2015-11-07
tags: 
  - JavaScript
  - Knockout
  - CDN
---

This is the seventh and last in a [series of posts]({{< ref "/posts/building-a-javascript-library" >}}) that discuss the steps taken to publish our library. In our [previous post]({{< ref "/posts/building-a-javascript-library-part-6-typescript" >}}), we added TypeScript support to our library. This post will show how we added our library to a Content Delivery Network.

## Content Delivery Network

A Content Delivery Network (CDN) is a network of servers that deliver content based on the geographic location of the user. In other words, when you request content from a CDN, the server geographically nearest to you will send the content. The main advantage of this is speed, but another is reliability. If one server goes down, another will automatically take over. Another advantage is that your own servers use less bandwith, very useful to cut down on bandwidth costs.

Some well-known CDN providers are [Akamai](https://www.akamai.com/us/en/media-and-delivery.jsp), [CloudFlare](https://www.cloudflare.com/features-cdn/) and [Amazon CloudFront](https://aws.amazon.com/cloudfront/). While most CDN providers are paid services, some offer basic functionality for free. 

## Hosted JavaScript libraries

For developers, CDN's are often used to serve JavaScript libraries. For example, [jQuery](https://jquery.com/) has its own CDN at [code.jquery.com](https://code.jquery.com/) which hosts jQuery, jQueryUI and several others. For a larger list of libraries, you can use the [Microsoft Ajax Content Delivery Network](http://www.asp.net/ajax/cdn) or [Google's hosted libraries](https://developers.google.com/speed/libraries/#libraries). However, if they don't host the library you want to use, you're out of luck, right? Enter cdnjs.

## cdnjs

[cdnjs](https://cdnjs.com/) is a CDN that hosts many JavaScript libraries, a lot more than the aforementioned CDN's. The great thing about cdnjs is that if they don't already host the library you want, you can add it yourelf! Let's do that for our library.

First we [fork the cdnjs repository](https://github.com/cdnjs/cdnjs/fork). In that fork, we create a new directory with our library's name in the `ajax/libs` folder. Within the created folder, we add a `package.json` file using cdnjs's custom [package.json format](https://github.com/cdnjs/cdnjs/blob/master/test/schemata/npm-package.json):

```json
{
  "filename": "knockout-paging.min.js",
  "name": "knockout-paging",
  "version": "0.3.0",
  "description": "Knockout paging",
  "keywords": ["knockout", "paging"],
  "homepage": "https://github.com/ErikSchierboom/knockout-paging",
  "dependencies": { 
    "knockout": "^3.2.0"
  }
}
```

Although the format is similar to the [regular package.json format](https://docs.npmjs.com/files/package.json), the `"filename"` field is new *and* required.

At this point, we can start adding the files we want cdnjs to serve. To do so, we create a subfolder for the version of our library which files we want to host. Within that folder, we then put all files we want to be hosted. For our library, this gives us the following files and folders:

<pre>
ajax
└── libs
    ├── ...  
    └── knockout_paging
        ├── 0.3.0
        |   ├── knockout-paging.js
        |   └── knockout-paging.min.js
        └── package.json
</pre>

Note that we distribute both the regular and minified versions of our library.

The final step is to commit our changes and send it in a pull request. Once accepted, our library will be [available on cdnjs](https://cdnjs.com/libraries/knockout-paging). The full URL for version 0.3.0 of our library's `knockout-paging.min.js` file is: [https://cdnjs.cloudflare.com/ajax/libs/knockout-paging/0.3.0/knockout-paging.min.js](https://cdnjs.cloudflare.com/ajax/libs/knockout-paging/0.3.0/knockout-paging.min.js).

Note that most of the steps to create the correct folders and files can also be done automatically using the [cdnjs-importer](https://github.com/cdnjs/cdnjs-importer) tool.

### Updating versions

To add a new version of a library, you used to create a new subfolder with that version's file(s). Then, you'd commit and send a new pull request. However, the preferred method nowadays is to [enable auto-updating](https://github.com/cdnjs/cdnjs#enabling-npmrecommended-or-git-auto-update). There are two ways libraries can be updated automatically:

1. Through NPM.
2. Through Git.

For our library, we'll use Git. To enable auto-updating from Git, we add the following to our cdnjs library's `package.json` file:

```json
"autoupdate": {
  "source": "git",
  "target": "git://github.com/ErikSchierboom/knockout-paging.git",
  "basePath": "/dist/",
  "files": [
    "knockout-paging.min.js",
    "knockout-paging.js"
  ]
}
```

This will instruct cdnjs to periodically check for new versions at the specified Git repository. It does this by checking the Git tags, which should use [semantic versioning](http://semver.org/). Now, if we commit the updated `package.json` file and submit it to cdnjs, new versions of our library wil automatically be added. Of course, old versions will remain available.

## Conclusion

We made our library available through a CDN by adding it to [cdnjs](https://cdnjs.com/), which was quite simple. Furthermore, we also configured cdnjs to automatically make new versions of our library available through the use of its auto-updating feature.

And that brings us to the end of the last of our [series of posts]({{< ref "/posts/building-a-javascript-library" >}}) on how we published our [knockout-paging](https://github.com/ErikSchierboom/knockout-paging) plugin.
Making our library available through a CDN was easy. 