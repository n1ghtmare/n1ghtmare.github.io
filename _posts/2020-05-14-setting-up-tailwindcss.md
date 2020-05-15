---
layout: post
title: "Setting up tailwindcss"
description: "Integrating Tailwind into an existing project with production optimizations"
date: 2020-05-14
tags: [programming, react, javascript, tailwindcss, css]
comments: false
share: true
---

I've started working with [Tailwindcss](https://tailwindcss.com/) recently and I'm loving it so far. This is a short write-up of how I've set everything up in my project.

First I've added tailwind into my project dependencies:

```
npm install --save-dev tailwindcss
```

Next I created a css file and called it `tailwind.css` which contains the following:

```css
@import "tailwindcss/base";

@import "tailwindcss/components";

@import "tailwindcss/utilities";
```

This is the file that is used by tailwind to generate an output css file which I will use in my app. You can also add your own utility styles/classes in between the imports (more info on that [here](https://tailwindcss.com/docs/adding-new-utilities)), although since I'm using tailwind I honestly have not needed it (yet).

Ok at this point I was ready to run tailwind and have it generate an output css file.

```
npx tailwindcss build css/tailwind.css -o css/tailwind.generated.css
```

Unfortunately this generates an output file that is quite large in size. That's because by default tailwind includes every single css class variation. The good news is that, tailwind now includes an option that can [purge unused classes](https://tailwindcss.com/docs/controlling-file-size), leaving you with only the ones that you use and reducing the size of the output file dramatically (in my app, it went down to 12KB!).

To setup purging I created a tailwind config file using the tailwind cli:

```
npx tailwindcss init
```

This will create `tailwind.config.js`, which can be used to configure and customize tailwind. By default it has the following contents:

```js
module.exports = {
    theme: {},
    variants: {},
    plugins: [],
}
```

To enable purging, I changed it to:

```js
module.exports = {
    // This points to the folder that contains my react components
    purge: ["./App/**/*.tsx", "./App/**/*.jsx"],
    
    // By default purging won't work on dev (NODE_ENV), if you want to force it, use the below
    // enabled: true,
    
    theme: {},
    variants: {},
    plugins: [],
}
```

Furthermore, I've modified the config file to include some helper classes for dark mode and some extra spacing sizes (allowing me to do `<div className="bg-gray-100 dark:bg-gray-900">`):

```js
module.exports = {
    purge: ["./App/**/*.tsx"],
    
    // By default purging won't work on dev (NODE_ENV), if you want to force it, use the below
    // enabled: true,

    theme: {
        extend: {
            screens: {
                dark: { raw: "(prefers-color-scheme: dark)" }
                // => @media (prefers-color-scheme: dark) { ... }
            },
            spacing: {
                "72": "18rem",
                "84": "21rem",
                "96": "24rem"
            }
        }
    },
    variants: {},
    plugins: []
};
```

Now that purging is configured, I modified my `package.json` to incorporate tailwind into my build:

```json
{
    //...
    "scripts": {
        "build:tailwind-dev": "tailwindcss build src/css/tailwind.css -o src/css/tailwind.generated.css",
        "build:tailwind-prod": "cross-env NODE_ENV=production tailwindcss build src/css/tailwind.css -o src/css/tailwind.generated.css",
        "prestart": "npm run build:tailwind-dev",
        "prebuild": "npm run build:tailwind-prod",
        // ...
    },    
    // ...rest of package.json
}
```

This was pretty easy to setup and the output is significantly smaller. Great! The problem was that the output was not minified, and I felt I could/had to optimize it even further. Unfortunately as of the time of writing this, tailwind doesn't support minimization the same way it supports purging (i.e. a config setting).

A way to fix this is to backtrack a little bit remove purging from the `tailwind.config.js` file and set it up using PostCSS instead.

I referred to the official tailwind [documentation](https://tailwindcss.com/docs/controlling-file-size/#setting-up-purgecss-manually). 

First, I installed the `postcss-cli` tools as well as `autoprefixer` (which is a postcss plugin that automatically adds vendor prefixes (`-moz-`, `-webkit-` etc.) to css classes:

```
npm install --save-dev postcss-cli autoprefixer
```

Then I created a `postcss.config.js` file, as follows:

```js
module.exports = {
    plugins: [
        require("tailwindcss"), 
        require("autoprefixer")
    ]
};
```

Now I tried to run the following command and see what the output would be:

```
npx cross-env NODE_ENV=production postcss src/css/tailwind.css -o src/css/tailwind.generated.css
```

An output file was built, but not purged (as expected, since I didn't enable it yet). In order to to that I included [PurgeCSS](https://purgecss.com/) as a dev dependency and included it in my `postcss.config.js` file. In fact, that's what tailwind is doing under the hood for its `purge` option from earlier. 

```
npm install --save-dev @fullhuman/postcss-purgecss
```

Then in `postcss.config.js`:

```js
const purgecss = require("@fullhuman/postcss-purgecss")({
    // Specify the paths to all of the template files in your project
    content: [
        "./src/**/*.tsx"
        // etc.
    ],

    // This is the function used to extract class names from your templates
    defaultExtractor: (content) => {
        // Capture as liberally as possible, including things like `h-(screen-1.5)`
        const broadMatches = content.match(/[^<>"'`\s]*[^<>"'`\s:]/g) || [];

        // Capture classes within other delimiters like .block(class="w-1/2") in Pug
        const innerMatches = content.match(/[^<>"'`\s.()]*[^<>"'`\s.():]/g) || [];

        return broadMatches.concat(innerMatches);
    }
});

module.exports = {
    plugins: [
        require("tailwindcss"),
        require("autoprefixer"),
        
        // Include purgecss only on production
        ...(process.env.NODE_ENV === "production" ? [purgecss] : [])
    ]
};

```

If I now run this again:

```
npx cross-env NODE_ENV=production postcss src/css/tailwind.css -o src/css/tailwind.generated.css
```

The output is purged as expected. Great!

Next thing I did was add minification. I used [cssnano](https://cssnano.co/):

```
npm install --save-dev cssnano
```

And then I modified our `postcss.config.js` to include it:

```js
// Purge css classes that are not being used
const purgecss = require("@fullhuman/postcss-purgecss")({
    // Specify the paths to all of the template files in your project
    content: [
        "./src/**/*.tsx"
        // etc.
    ],

    // This is the function used to extract class names from your templates
    defaultExtractor: (content) => {
        // Capture as liberally as possible, including things like `h-(screen-1.5)`
        const broadMatches = content.match(/[^<>"'`\s]*[^<>"'`\s:]/g) || [];

        // Capture classes within other delimiters like .block(class="w-1/2") in Pug
        const innerMatches = content.match(/[^<>"'`\s.()]*[^<>"'`\s.():]/g) || [];

        return broadMatches.concat(innerMatches);
    }
});

// Minification of CSS
const cssnano = require("cssnano")({
    preset: "default"
});

module.exports = {
    plugins: [
        require("tailwindcss"),
        require("autoprefixer"),
        
        // Here we include both purging and minification
        ...(process.env.NODE_ENV === "production" ? [purgecss, cssnano] : [])
    ]
};

```

Now I tried running this again:

```
npx cross-env NODE_ENV=production postcss src/css/tailwind.css -o src/css/tailwind.generated.css
```

The output file was purged and minimized (and tiny! - about 4KB down from 12KB).

One thing left to do was to modify my `package.json` to use the `postcss-cli` rather than the `tailwind-cli`:

```json
{
    //...
    "scripts": {
        "build:tailwind-dev": "postcss src/css/tailwind.css -o src/css/tailwind.generated.css",
        "build:tailwind-prod": "cross-env NODE_ENV=production postcss src/css/tailwind.css -o src/css/tailwind.generated.css",
        // ...
    }
    //...
}
```

At this point I was ready to go with my tailwind setup with purging and minimization and all the jazz.