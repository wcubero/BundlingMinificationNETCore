# BundlingMinificationNETCore

How we do bundling and minification in ASP.NET Core
Facebook Twitter LinkedIn Email
Written by Thomas Ardal, October 27, 2020
 Contents
When migrating our websites to ASP.NET Core, we had to find a new way to bundle and minify JavaScript and CSS files. We used the System.Web.Optimization namespace in ASP.NET MVC, combined with the AspNetBundling and BundleTransformer packages for some additional features like generating source maps. In this post, I'll go through the possibilities we went through when migrating and what we ended up choosing.


Let's start by looking at how the documentation suggests implementing bundling and minification in ASP.NET Core. I don't want to go into too much detail, since you can simply read through the documentation if you want. The changes we made were from a baseline of the official docs, so let's dive in.  

In ASP.NET Core you typically add your static files in the wwwroot folder. There is a folder for CSS (css), one for JavaScript (js), and a third one for external libraries (lib). Bundling and minification don't care where you place your files, but I tend to like this structure why the new elmah.io websites are based on it. To include bundling and minification, add a new file in the root of the web project named bundleconfig.json. Here's an example of how the content could look like in the default template:

[
  {
    "outputFileName": "wwwroot/css/site.min.css",
    "inputFiles": [
      "wwwroot/lib/bootstrap/dist/css/bootstrap.css",
      "wwwroot/css/site.css"
    ]
  },
  {
    "outputFileName": "wwwroot/js/site.min.js",
    "inputFiles": [
      "wwwroot/js/site.js"
    ],
    "minify": {
      "enabled": true,
      "renameLocals": true
    },
    "sourceMap": false
  }
]

Let's go through the content fast. For each bundle you want to create, there's a JSON object with an outputFileName and a set of files to include in the inputFiles array. In this example, we build a minified CSS bundle named site.min.css and a minified JavaScript bundle named site.min.js. I'll use only the CSS part in the rest of this post to simplify things, but everything applies to JavaScript files as well. To have Visual Studio and dotnet build the bundles, you can install the BuildBundlerMinifier NuGet package:

dotnet add package BuildBundlerMinifier

When building through either Visual Studio or dotnet build you now get the bundles generated and you can commit and push them. On localhost, you never want to use bundled and minified files, why you can use tag helpers to reference the individual files locally and the bundles everywhere else:

<environment include="Development">
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="~/css/site.css" />
</environment>
<environment exclude="Development">
    <link rel="stylesheet" href="~/css/site.min.css" />
</environment>

A pretty simple solution, but with a range of downsides:

You need to compile the project to rebuild bundles (not a problem if using unbundled files locally, but still).
Bundled files are in source control. You don't want that since that causes merge conflicts all the time.
You need to maintain a list of files in two places (`bundleconfig.json` and in HTML).
This translates to a range of goals that I want to achieve with the solution presented in the rest of this post:

Don't include the BuildBundlerMinifier package.
No minified bundles in Git.
The list of needed CSS and JavaScript files for each page should only be written once.
Minification and bundling should be done on the build server.
In the following sections, I have a solution for each bullet. The first part doesn't require more explanation. If you have the BuildBundlerMinifier package installed, go ahead and uninstall it.

No minified bundles in Git
The solution suggested in this post is based on not building the minified bundles locally. In theory, this means that the .min files will not even exist in your local checkout. You probably still want to exclude these files through .gitignore, since someone will end up generating these files locally either by mistake or to test out a problem only existing on the bundled files.

You can either create a set of wildcard URLs to exclude or you can move the generated bundles entirely like this:

[
  {
    "outputFileName": "wwwroot/bundles/site.min.css",
    "inputFiles": [
      "wwwroot/lib/bootstrap/dist/css/bootstrap.css",
      "wwwroot/css/site.css"
    ]
  },
  {
    "outputFileName": "wwwroot/bundles/site.min.js",
    "inputFiles": [
      "wwwroot/js/site.js"
    ],
    "minify": {
      "enabled": true,
      "renameLocals": true
    },
    "sourceMap": false
  }
]

The only change is the path in the outputFileName value which now avoids generating the bundled files inside the css and js folders. This way you can simply ignore this path in .gitignore:

# Don't include bundled files in git
**/wwwroot/bundles

Only declare dependencies once
As we already saw in the example code, we have a reference to both bootstrap.css and site.css in both the bundleconfig.json file and in the HTML file needing this (typically the _Layout.cshtml file). To combine these into one, there's a nice little NuGet package named BundlerMinifier.TagHelpers that we can install:

dotnet add package BundlerMinifier.TagHelpers

As the name suggests, this package comes with a range of ASP.NET Core tag helpers to help with bundling and minification. To use the tag helpers in cshtml files, insert the following code in the _ViewImports.cshtml file:

@addTagHelper *, BundlerMinifier.TagHelpers

With this in place, you can replace all of the <link> elements from the previous example with this:

<bundle name="wwwroot/bundles/site.min.css" />

To make sure that the browser always knows when to fetch an updated bundle, include a version number of the bundled file by adding the following code to the ConfigureServices method in the Startup.cs file:

services.AddBundles(options =>
{
    options.AppendVersion = true;
});

That's it. The tag helper generates a nice list of individual files when running on localhost:

<link href="/lib/bootstrap/dist/css/bootstrap.css" rel="stylesheet" />
<link href="/css/site.css" rel="stylesheet" />

And a bundled, minified, and versioned file when running somewhere else:

<link href="/bundles/site.min.css?v=Hr2K_e4FFmONl0h--fZbjZJrI6JwyQ7kHuXgHE85RxM" rel="stylesheet" />

Minify and bundle on the build server
The only part missing now is building the bundles. We no longer have the BuildBundlerMinifier to help us out, why we need something to execute the bundling and minification tasks. The easiest way I have found is using Gulp. I know, there are multiple options out there like Webpack and similar. But since there's an easy migration path to Gulp and it is very well supported on Azure DevOps, I have chosen that stack for elmah.io.

To start using Gulp, install the Bundler & Minifier Visual Studio extension. After restarting VS, you can right-click the bundleconfig.json file and Convert to Gulp:


This will produce a new gulpfile.js file in the root of the project. After generating that file, you should uninstall the Bundler & Minifier extension. You can have gulp build the bundles by executing the following command:

gulp min

Unless there's are heavy demand for it, I'll skip the part of setting this up on Azure DevOps. It should work for all build servers, but will require you to include an install step before being able to execute gulp:

npm install

Since Gulp is now in charge of bundling and minification, you can clean up parts of the bundleconfig.json file:

[
  {
    "outputFileName": "wwwroot/bundles/site.min.css",
    "inputFiles": [
      "wwwroot/lib/bootstrap/dist/css/bootstrap.css",
      "wwwroot/css/site.css"
    ]
  },
  {
    "outputFileName": "wwwroot/bundles/site.min.js",
    "inputFiles": [
      "wwwroot/js/site.js"
    ]
  }
]

I removed the minify and sourceMap stuff as these tasks are now handled by different npm packages. The bundleconfig.json file now only serves as a list of input and output files for the Gulp script.

That's the solution implemented for now. I hope that is can serve as an inspiration for others needing to implement something similar. To be honest, I found the official documentation a bit confusing and presenting a solution with too many downsides. All of the bits and pieces are there (mostly because Mads Kristensen made them), you just need to find them and put together a solution that fits your project.
