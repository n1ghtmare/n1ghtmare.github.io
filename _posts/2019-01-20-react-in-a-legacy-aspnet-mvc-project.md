---
layout: post
title: "React in a legacy ASP.NET MVC project"
description: "Integrating React into a ASP.NET MVC 5 legacy project"
date: 2019-01-20
tags: [programming, react, javascript, asp.net, mvc, c#, webpack]
comments: false
share: true
---

I was working on an existing "legacy" ASP.NET MVC 5 project recently. The end-users requested some (fancy) upgrades to the UI. The client UI was just HTML and jQuery, which to be honest worked out great up to this point. The upgrade though, introduces a lot of new dynamic UI elements and views that would be difficult to manage with just jQuery, therefore React feels like a perfect fit for the task. I decided against a re-write but instead, introduce React gradually into the project, and mainly use it for the new UI and only selected areas (components/features) of existing pages. I would like to have multiple "bundles" and entry points for each feature and its code. This is a write-up of the process.

The first thing that I tried was to use [ReactJS.NET](https://reactjs.net). The idea is that you write your React components and bundle them into the ASP.NET pipeline. So in your `BundleConfig.cs` you would do:


``` csharp
// In BundleConfig.cs
bundles.Add(new JsxBundle("~/bundles/my-react-bundle").Include(
    "~/Scripts/App/Component1.jsx",
    "~/Scripts/App/Component2.jsx",
    // You get the idea ..
    
    // .. mix in your js into the bundle
    "~/Scripts/utilities.js",
));
```


ReactJS.NET also supports server-side rendering and dynamic (on-the-fly) JSX to JS compilation. 

I used ReactJS.NET for a while, and although it's a cool project, I found it slow, hard to debug, and tricky to deploy. The documentation seemed a bit lacking and the whole project seemed a bit ... stale.

I decided to go with a standard `npm`, `webpack` route instead and build my components and dependencies outside of the ASP.NET pipeline.

First thing I did was, in the root of my ASP.MVC project folder, run:

```
npm init
```

Next thing I would need is `webpack` and `webpack-cli` (so that you can use webpack in the command line), so I installed them as dev dependencies:

```
npm install --save-dev webpack webpack-cli
```

Then I installed `react` and `react-dom`:

```
npm install --save react react-dom
```

Next I installed babel (and babel related dependencies):

- babel core (transpile ES6 to ES5) 
- babel loader (webpack loader for babel)
- babel preset env (polyfills, transformations etc - for various browsers)
- babel preset react (compile React specific things like JSX down to javascript)

I ran the following:

```
npm install --save-dev @babel/core babel-loader @babel/preset-env @babel/preset-react
```

After which I created a babel config (a file called `.babelrc` that specifies the presets):

``` js
{
    "presets": ["@babel/preset-env", "@babel/preset-react"]
}
```

Then I created a minimal webpack config (`webpack.config.js`):

``` js
module.exports = {
    resolve: {
        // gain the ability to import from different folders and omit the ".jsx"
        extensions: [".js", ".jsx"]
    },
    module: {
        rules: [
            {
                test: /\.(js|jsx)$/,
                exclude: /node_modules/,
                use: {
                    loader: "babel-loader"
                }
            }
        ]
    }
};
```

This config is pretty straight forward - it simply says that anything with an extension `.js` or `.jsx` should go through babel-loader to be compiled down to ES5 (from ES6).

At this point I was ready to build my application. I decided to keep all my client code into a folder called ... well "Client" (again on the root of my ASP.NET MVC project), as well as an output folder called "Dist" (for the end-result of the webpack build) in the already-existing Scripts folder. Then I went ahead and updated the `webpack.config.js` to reflect those changes:

``` js
const path = require("path");

module.exports = {
    entry: {
        app: path.resolve(__dirname, "Client/Boot/app.jsx")
    },
    output: {
        filename: "[name].bundle.js",
        path: path.resolve(__dirname, "Scripts/Dist")
    },
    resolve: {
        extensions: [".js", ".jsx"]
    },
    module: {
        rules: [
            {
                test: /\.(js|jsx)$/,
                exclude: /node_modules/,
                use: {
                    loader: "babel-loader"
                }
            }
        ]
    }
};
```

The output file will correspond to the entry name, so in my case the output file will be `/Scripts/Dist/app.bundle.js`. 

The `/Client/Boot/app.jsx` looked as follows:

``` js
import React from "react";
import ReactDOM from "react-dom";

const TestComponent = () => <h1>Our test component</h1>;

ReactDOM.render(
    <TestComponent />,
    document.getElementById("#component-container")
);
```

At this point I had React wired up and I needed to include the output bundle into my MVC Razor View. In my `Index.cshtml` view (of Home/Index) I added the following:

``` cs
<div id="component-container"></div>

@section Scripts {
    @Scripts.Render("~/Scripts/Dist/app.bundle.js");
}
```

Now I ran `npm run build`, and ran my project with Visual Studio (F5). Everything worked at this point. Great!

Next, I wanted to run a dev build and I didn't want to run the start command after every file change so I added the `--watch` flag in the `start` command, I modified my `package.json`:

``` json
"scripts": {
    "start": "webpack --mode development --display-error-details --watch",
    "build": "webpack --mode production"
},
```

When added, the `--watch` flag re-compiles your app every time you change a file in the webpack build tree automatically. So my workflow at this point was - run the project in Visual Studio (F5), in the terminal run `npm start` and open the dev URL to see changes. Modify JS files -> refresh the browser.

One final thing, webpack allows you to have multiple entries and multiple corresponding outputs, which was perfect for the requirements I had, and the way I wanted to use it (a bundle per component/feature). So in my MVC view I could for example have:

``` csharp
<div id="notifications"></div>
<div id="comments"></div>
<div id="fancy-grid"></div>

@section Scripts {
    @Scripts.Render("~/Scripts/Dist/notifications.bundle.js");
    @Scripts.Render("~/Scripts/Dist/comments.bundle.js");
    @Scripts.Render("~/Scripts/Dist/fancy-grid.bundle.js");
}
```

And my `webpack.config.js` could look like this:

``` js
const path = require("path");

module.exports = {
    entry: {
        "notifications": path.resolve(__dirname, "Client/Boot/Notifications.jsx"),
        "comments": path.resolve(__dirname, "Client/Boot/Comments.jsx"),
        "fancy-grid": path.resolve(__dirname, "Client/Boot/FancyGrid.jsx"),
    },
    output: {
        filename: "[name].bundle.js",
        path: path.resolve(__dirname, "Scripts/Dist")
    },
    resolve: {
        extensions: [".js", ".jsx"]
    },
    module: {
        rules: [
            {
                test: /\.(js|jsx)$/,
                exclude: /node_modules/,
                use: {
                    loader: "babel-loader"
                }
            }
        ]
    }
};
```

My `Client` folder I've structured as follows:

- `Boot` (folder for entry points for features)
- `Components` (actual components)
- `Helpers`
- ... etc.

One final note - I had to of course include the `/Scripts/Dist` folder to the `.gitignore` (I also double-checked that `node_modules` was included in the `.gitignore`, you can never be too careful).

Overall, I'm happy with how things turned out. I'm looking for tools that will automate the build process and integrate it into Visual Studio, although I don't particularly mind typing the necessary build commands.
