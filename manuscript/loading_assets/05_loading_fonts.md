# Loading Fonts

Loading fonts is a surprisingly tough problem. There are typically four(!) font formats to worry about, each for certain browser. Inlining all formats at once wouldn't be a particularly good idea. It is acceptable, and even preferable, during development. That said, there are a couple of production specific strategies we can consider.

## Choosing One Format

Depending on your project requirements, you might be able to get away with less formats. If you exclude Opera Mini, all browsers support the *.woff* format. The render result may differ depending on the browser so you might want to experiment here.

If we go with just one format, we can use a similar setup as for images and rely on both *file-loader* and *url-loader* while using the limit option:

```javascript
{
  test: /\.woff$/,
  loader: 'url-loader',
  options: {
    limit: 50000
  }
}
```

A more elaborate way to achieve a similar result would be to use:

```javascript
{
  // Match woff2 in addition to patterns like .woff?v=1.1.1.
  test: /\.woff(2)?(\?v=[0-9]\.[0-9]\.[0-9])?$/,
  loader: 'url-loader',
  options: {
    limit: 50000,
    mimetype: 'application/font-woff',
    name: './fonts/[hash].[ext]'
  }
}
```

## Supporting Multiple Formats

In case we want to make sure our site looks good on a maximum amount of browsers, we might as well use just *file-loader* and forget about inlining. Again, it's a trade-off as we get extra requests, but perhaps it's the right move. Here we could end up with a loader configuration like this:

```javascript
{
  test: /\.woff$/,
  // Inline small woff files and output them below font/.
  // Set mimetype just in case.
  loader: 'url-loader',
  options: {
    name: 'fonts/[hash].[ext]',
    limit: 5000,
    mimetype: 'application/font-woff'
  }
},
{
  test: /\.ttf$|\.eot$/,
  loader: 'file-loader',
  options: {
    name: 'fonts/[hash].[ext]'
  }
}
```

## Generating Font Files Based on SVGs

Sometimes you might have a bunch of SVG files that would be nice to bundle as font files of their own. The setup is a little involved, but [fontgen-loader](https://www.npmjs.com/package/fontgen-loader) achieves just this.

## Manipulating `file-loader` Output Path and `publicPath`

As discussed above and in [webpack issue tracker](https://github.com/webpack/file-loader/issues/32#issuecomment-250622904), *file-loader* allows shaping the output. This way you can output your fonts below `fonts/`, images below `images/`, and so on over using the root.

Furthermore, it's possible to manipulate `publicPath` and override the default per loader definition. The following example illustrates these techniques together:

```javascript
{
  // Match woff2 in addition to patterns like .woff?v=1.1.1.
  test: /\.woff(2)?(\?v=[0-9]\.[0-9]\.[0-9])?$/,
  loader: 'url-loader',
  options: {
    limit: 50000,
    mimetype: 'application/font-woff',
    // Output below the fonts directory
    name: './fonts/[hash].[ext]',
    // Tweak publicPath to fix CSS lookups to take
    // the directory into account.
    publicPath: '../'
  }
}
```

## Using Font Awesome

The ideas above can be wrapped into a configuration part that allows you to work with [Font Awesome](https://www.npmjs.com/package/font-awesome). Here's the idea:

```javascript
exports.loadFonts = function(options) {
  const paths = options.paths;
  const limit = options.limit;
  const name = options.name || 'fonts/[chunkhash].[ext]';

  return {
    module: {
      rules: [
        {
          test: /\.(woff|woff2)(\?v=\d+\.\d+\.\d+)?$/,
          loader: 'url-loader',
          options: {
            limit: limit,
            mimetype: 'application/font-woff',
            name: name
          }
        },
        {
          test: /\.ttf(\?v=\d+\.\d+\.\d+)?$/,
          loader: 'url-loader',
          options: {
            limit: limit,
            mimetype: 'application/octet-stream',
            name: name
          }
        },
        {
          test: /\.eot(\?v=\d+\.\d+\.\d+)?$/,
          loader: 'url-loader',
          options: {
            limit: limit,
            name: name
          }
        },
        {
          test: /\.svg(\?v=\d+\.\d+\.\d+)?$/,
          loader: 'url-loader',
          options: {
            limit: limit,
            mimetype: 'image/svg+xml',
            name: name
          }
        }
      ]
    }
  };
}
```

To include Font Awesome into your project, install it (`npm i font-awesome -S`) and then refer to it. One way to handle that is to point to `path.resolve(__dirname, 'node_modules/font-awesome/css/font-awesome.css')` at your style paths or import the file through your application code. After this you can use Font Awesome classes across your code.

## Conclusion

Loading fonts is similar as loading other assets. Here we have extra concerns to consider. We need to consider the browsers we want to support and choose the loading strategy based on that.
