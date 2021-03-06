# Separating Manifest

When webpack writes bundles, it writes something known as **manifest** as well. You can find it in the generated *vendor* bundle in this project. The manifest describes what files webpack should load. When it comes to hashing, the default behavior becomes problematic as it leads to a validation problem.

The problem is that if our application bundle changes somehow, this means its hash will change as well. And when the hash changes, so does the manifest. Because the manifest changes, this means the vendor bundle hash needs to change. This cascade effect invalidates both and it's something that's better to be avoided to get a stable hashing behavior.

The easiest way to solve the problem is to tell webpack to extract the manifest to a file of its own.

## Extracting Manifest

We have done most of the work already when we set up `extractBundle`. To extract the manifest, a single change is required:

**webpack.config.js**

```javascript
...

module.exports = function(env) {
  if (env === 'production') {
    return merge(
      ...
      parts.extractBundle({
        name: 'vendor',
        entries: ['react']
      }),
leanpub-start-insert
      parts.extractBundle({
        name: 'manifest'
      }),
leanpub-end-insert
      parts.minify(),
      parts.setupCSS(PATHS.app)
    );
  }

  ...
};
```

If you build the project now (`npm run build`), you should see something like this:

```bash
Hash: 295e44e81d90cb11f12d
Version: webpack 2.2.0-rc.1
Time: 1223ms
                           Asset       Size  Chunks           Chunk Names
  vendor.6fcd8ee954db093c19c0.js    19.7 kB  0, 2[emitted]  vendor
     app.5caccf3ee1ca2f876fe1.js    4.06 kB  1, 2[emitted]  app
manifest.9236efcd5974e259d881.js    1.42 kB  2[emitted]  manifest
                      index.html  357 bytes  [emitted]
  [15] ./app/component.js 136 bytes {1} [built]
  [16] ./app/main.css 904 bytes {1} [built]
  [17] ./~/css-loader!./app/main.css 190 bytes {1} [built]
  [32] ./app/index.js 124 bytes {1} [built]
  [33] multi vendor 28 bytes {0} [built]
    + 29 hidden modules
Child html-webpack-plugin for "index.html":
        + 4 hidden modules
```

This simple change gave us a separate file that contains the manifest. To get a better idea of its contents, comment out `parts.minify()` and examine the resulting manifest. You should see familiar names there.

Try adjusting *app.js* as well and see how the hashes change. This time around it should **not** invalidate the vendor bundle.

### inline-manifest-webpack-plugin

In our current setup you end up with an extra request to fetch the small manifest file. If you want to save that, a good option is to inline it within your *index.html*. [inline-manifest-webpack-plugin](https://www.npmjs.com/package/inline-manifest-webpack-plugin) works in tandem with *html-webpack-plugin* to allow this. Setting it up requires some configuration and a custom template:

**webpack.config.js**

```javascript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
leanpub-start-insert
const InlineManifestWebpackPlugin = require('inline-manifest-webpack-plugin');
leanpub-end-insert
const merge = require('webpack-merge');

...

const common = {
  entry: {
    style: PATHS.style,
    app: PATHS.app
  },
  output: {
    path: PATHS.build,
    filename: '[name].js'
  },
  plugins: [
    new HtmlWebpackPlugin({
leanpub-start-delete
      title: 'Webpack demo'
leanpub-end-delete
leanpub-start-insert
      title: 'Webpack demo',
      template: './index.ejs'
leanpub-end-insert
    })
  ]
};

module.exports = function(env) {
  if (env === 'production') {
    return merge(
      common,
      {
        devtool: 'source-map',
        output: {
          path: PATHS.build,
          filename: '[name].[chunkhash].js',
          // This is used for code splitting. The setup
          // will work without but this is useful to set.
          chunkFilename: '[chunkhash].js',
          // Tweak this to match your GitHub project name
          publicPath: '/webpack-demo/'
leanpub-start-delete
        }
leanpub-end-delete
leanpub-start-insert
        },
        plugins: [
          new InlineManifestWebpackPlugin({
            name: 'webpackManifest'
          })
        ]
leanpub-end-insert
      },
      ...
    );
  }

  ...
};
```

In addition, a custom template is required to inject the manifest within *index.html*:

**index.ejs**

```ejs
<!doctype html>
<html>
  <head>
    <meta charset="UTF-8">
    <title><%= htmlWebpackPlugin.options.title %></title>
  </head>
  <body>

  <%= htmlWebpackPlugin.files.webpackManifest %>

  </body>
</html>
```

This would be a good place to perform other template specific tweaks, but this is enough for this demonstration.

If you build the project (`npm run build`) now, you should see output without the separate manifest file:

```bash
Hash: 295e44e81d90cb11f12d
Version: webpack 2.2.0-rc.1
Time: 1223ms
                           Asset       Size  Chunks           Chunk Names
  vendor.6fcd8ee954db093c19c0.js    19.7 kB  0, 2[emitted]  vendor
     app.5caccf3ee1ca2f876fe1.js    4.06 kB  1, 2[emitted]  app
                      index.html    1.95 kB  [emitted]
  [15] ./app/component.js 136 bytes {1} [built]
  [16] ./app/main.css 904 bytes {1} [built]
  [17] ./~/css-loader!./app/main.css 190 bytes {1} [built]
  [32] ./app/index.js 124 bytes {1} [built]
  [33] multi vendor 28 bytes {0} [built]
    + 29 hidden modules
Child html-webpack-plugin for "index.html":
        + 4 hidden modules
```

*index.html* is larger as expected. If you inspect it, you will see it contains the manifest.

## Conclusion

Even though our project has neat caching behavior now, adding hashes to our filenames brings a new problem. If a hash changes, we still have possible older files within our output directory. To eliminate this problem, we can set up a little plugin to clean it up for us.
