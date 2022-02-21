# Why and how to transpile dependencies

If you’re a web developer, I’m sure that you use bundlers (e.g., [Webpack](https://webpack.js.org), [Rollup](https://rollupjs.org/guide/en/), or [Parcel](https://parceljs.org)) which transpile the JavaScript code of your application with [Babel](https://babeljs.io) under the hood. No doubt that you also use various dependencies to cut off the development time.

However, I rarely see developers transpiling the code of their dependencies because everything seems to work fine without it, right? Wrong! Here’s why...

## Adoption of ESM

Before browsers and Node.js got native support for ES modules, an npm package could contain several variants of source code:

- [CommonJS](https://en.wikipedia.org/wiki/CommonJS) variant that doesn’t use [modern features](http://es6-features.org/#ExpressionBodies) of JavaScript such as arrow functions. It’s compatible with most versions of Node.js and browsers. The source code location is specified in the `main` field of `package.json`.
- module variant that has ES6 imports/exports but still doesn’t use modern features of JavaScript. Modules allow for [tree-shaking](https://developer.mozilla.org/en-US/docs/Glossary/Tree_shaking), i.e., excluding unused code from a bundle. The source code location is specified in the `module` field of `package.json` (see [this discussion](https://stackoverflow.com/questions/42708484/what-is-the-module-package-json-field-for) for details).

Obviously, not every npm package follows this convention. It’s a choice that every author of a library makes on their own. However, it’s quite common for browser-side and universal libraries to be distributed in two variants.

[Web browsers](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) have natively supported ES modules for more than three years already, and [Node.js](https://nodejs.org/api/esm.html) supports them since version 12.20 (released in November 2020). That’s why authors of libraries now include one more variant of source code for execution environments that natively support ES modules, and many packages have completely removed the support for CommonJS.

## Perks and features of ESM

It’s important to understand that native ES modules are very much different than modules that have ES6 imports/exports. Here are a few reasons:

- The way we are used to using imports/exports will not work natively. Such code is intended for further processing by a bundler or a transpiler.
- Native ES modules don’t support `index` file name and extension resolution, so you have to specify them explicitly in the import path:

    ```js
    // Won't work:
    import _ from './utils'

    // Works:
    import _ from './utils/index.js'
    ```

- If an npm package has ES modules, you have to add `"type": "module"` to `package.json` and specify the source code location in the `exports` field (see [docs](https://nodejs.org/api/packages.html#exports) for details).

You can check [this blog post](https://2ality.com/2019/10/hybrid-npm-packages.html) by Axel Rauschmayer to learn more about ES modules.

## Support for JavaScript features

Look! Since we know which versions of web browsers and Node.js support ES modules, we also know for sure which features of JavaScript we can use in the source code.

For instance, all web browsers that support ES modules natively also support [arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions), so we don’t need to avoid using them or use Babel to transpile them to regular functions. Authors of libraries know that and ship the source code that leverages all modern features of JavaScript.

## Transpilation of dependencies’ code

But wait! What can you do to make sure that your web application works in legacy browsers? Also, what to do if any of your application’s dependencies use modern features of JavaScript that aren’t supported by popular browsers?

In both cases, you need to transpile the code of the dependencies.

### Manual transpilation

Let’s assume that we’re using webpack and [babel-loader](https://webpack.js.org/loaders/babel-loader/#usage). Often, the config would look like this:

```js
module: {
  rules: [
    {
      test: /\.jsx?$/,
      exclude: /node_modules/,
      use: {
        loader: 'babel-loader',
        options: {
          presets: [
            ['@babel/preset-env', { targets: 'defaults' }]
          ]
        }
      }
    }
  ]
}
```

It’s suggested in the documentation and usage examples for Babel and `babel-loader` to exclude `node_modules` from transpilation (`exclude: /node_modules/`) to optimize the performance.

By removing the `exclude` rule, we’ll **enable the transpilation of dependencies’ code** in exchange for the increased bundling time. By providing a custom function as the `exclude` rule, we also can transpile just a subset of all dependencies:

```js
exclude: _ => /node_modules/.test(_) && !/node_modules\/(nanostores|p-limit)/.test(_)
```

We can also select files by their filename extensions:

```js
exclude: _ => /node_modules/.test(_) && !/(\.babel\.js|\.mjs|\.es)$/.test(_)
```

### Manual transpilation — the benchmark

Let’s check how `babel-loader` configuration affects the bundle size and bundling time. Consider an application with three very different dependencies:

- [svelte](https://www.npmjs.com/package/svelte) (uses modern features of JavaScript such as arrow functions)
- [p-limit](http://npmjs.com/package/p-limit) (uses bleeding-edge features of JavaScript such as private class fields)
- [axios](http://npmjs.com/package/axios) (regular ES5 code)

| Config | Transpilation | Compatibility | Bundle size | Bundling time |
| --- | --- | --- | --- | --- |
| Basic | — | No way to predict which web browsers will work with this bundle | 21 KB | 1.8 s |
| `target: defaults and supports es6-module` | To ES6 code. Private class fields will be downgraded, arrow functions and classes will remain as is | Modern browsers  | 22 KB | 2.6 s |
| `target: defaults` with polyfills | To ES5 code | All browsers | 123 KB | 6.1 s |

You can see that the total bundling time for modern browsers and all browsers with `babel-loader` is 8.7 s. Please also note that the basic, non-transpiled bundle won’t work with legacy browsers because of `p-limit`.

(By the way, I also have a [blog post](https://dev.to/dangreen/speed-up-with-browserslist-30lh) that explains in detail how to build several bundles for different browsers.)

Okay, but what if you don’t want to tinker with configs and specify files and packages to be transpiled manually? Actually, there’s a readily available tool for that!

### Transpilation with optimize-plugin

Meet [optimize-plugin](https://github.com/developit/optimize-plugin) for webpack by Jason Miller from Google ([@_developit](https://twitter.com/_developit)). It will take care of everything and even more:

- It will transpile your application’s source code and the code of all dependencies.
- If needed, it will generate two bundles (for modern and legacy browsers) using the [module/nomodule pattern](https://jasonformat.com/modern-script-loading/).
- On top of that, it can also upgrade ES5 code to ES6 using [babel-preset-modernize](https://github.com/developit/babel-preset-modernize)!

Let’s see what `optimize-plugin` will do to our example application with three dependencies:

| Config | Transpilation | Compatibility | Bundle size | Bundling time |
| --- | --- | --- | --- | --- |
| Basic | To ES6 code. Also, to ES5 code with polyfills | All browsers | 20 KB for modern browsers. 92 KB for legacy browsers (including 67 KB of polyfills) | 7.6 s |

The total bundling time with `optimize-plugin` is 7.6 s. As you can see, `optimize-plugin` is not only faster than `babel-loader`, but it also produces a smaller bundle. You can check my results using the code from my [optimize-plugin-demo](https://github.com/dangreen/optimize-plugin-demo) repository.

### Why optimize-plugin wins

The performance boost is possible because the code is analyzed and bundled only once. After that, `optimize-plugin` transpiles it for modern and legacy browsers.

Smaller bundle size is possible thanks to [babel-preset-modernize](https://github.com/developit/babel-preset-modernize). Chances are that you use ES6+ features in your application’s code but you never can predict which features are used in the source code of the dependencies. Since `optimize-plugin` works with the bundle that already has the code of all dependencies, it can transpile it as a whole.

Here’s how `babel-preset-modernize` works. Consider this code snippet:

```js
const items = [{
  id: 0,
  price: 400
}, {
  id: 1,
  price: 300
}, {
  id: 2,
  price: 550
}];
const sum = items.reduce(function (sum, item) {
  const price = item.price;
  return sum + price;
}, 0);

console.log(sum);
```

After transpilation to ES6, we’ll get this code:

```js
const items = [{
  id: 0,
  price: 400
}, {
  id: 1,
  price: 300
}, {
  id: 2,
  price: 550
}];
const sum = items.reduce((sum, {
  price
}) => sum + price, 0);

console.log(sum);
```

Here’s what has changed:

- A regular anonymous function was upgraded to an arrow function.
- `item.price` field access was replaced with the function argument destructuring.

Code size shrinked from 221 to 180 bytes. Note that we applied only two transformations here but `babel-preset-modernize` can do a lot more.

## What’s next?

`optimize-plugin` works really great but it still has some room for improvement. Recently, I’ve [contributed](https://github.com/developit/optimize-plugin/releases/tag/1.1.0) a few pull requests, including the support for webpack 5.

If `optimize-plugin` looks promising to you, I encourage you to [give it a try](https://github.com/developit/optimize-plugin#install) in your projects and maybe contribute some improvements as well.

Anyway, starting today, please always transpile the code of the dependencies, whether with `optimize-plugin` or not, to make sure that you have full control over your application’s compatibility with modern and legacy browsers. Good luck!
