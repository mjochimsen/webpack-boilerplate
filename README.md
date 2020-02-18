# Setting up a development environment for a web client

First off create a directory for the environment and run `git init` to
initialize a Git repository. This can be linked to a networked repo if
desired.

We'll be using [Webpack](https://webpack.js.org/) to build our assests and
serve them while we do our development. Webpack is a Node.js toolset, so
first we need to get Node and npm installed.

The easy way to get the project set up to use npm is with `npm init`. This
will ask a bunch of questions about the project to fill out a default
`package.json` file. The version should probably be a `0.x.y` or something
along those lines, and the entry point should be `index.js`. Don't worry
about a test command at this time. The git repository can also be skipped,
unless you already have something set up. Including at least one keyword
is a good idea, but not strictly necessary.

Once the `package.json` file is created, feel free to open it up in a text
editor and tweak anything which needs tweaking. If there is no plan to
publish this as an npm package (likely) then adding a `"private": true`
line to the file is probably a good idea.

The next step is to add the Webpack software. We'll be using `webpack`,
`webpack-cli` and `webpack-dev-server` as a base. These are installed with
npm as follows:

```
npm install webpack webpack-cli webpack-dev-server --save-dev
```

This will take a few moments to download and install. The needed files
will be downloaded into the `node_modules/` subdirectory. This
subdirectory should be added to the `.gitignore` file.

Note that there will also be a `package-lock.json` file added to the
project. This should be added to the Git repository, but it should
probably be its own commit so as to not distract from the data in the
commit containing the changes to `package.json`. This file is used to
ensure that a consistant, working set of packages are installed for a
given project. If it is not committed then the lastest packages will be
used, which may or may not work as expected.

Also note that the packages are only included as a development dependency.
This is appropriate, as the Webpack npm packages are only used when
building the files in the project.

Once the base Webpack tools are installed, we can add a couple of scripts
to the `package.json` file. Add the following lines to the `"scripts"`
section of `package.json`:

```
"build": "webpack",
"serve": "webpack-dev-server --open",
```

These scripts can be run with the `npm run [script]` command. There may be
a need to adjust the trailing commas to make sure that the file is still
valid JSON.

At this point we can add the Webpack configuration. This lives in
`webpack.config.js`, and looks like the following:

```
const path = require('path');

module.exports = {
  mode: 'development',
  entry: './src/index.js',
  output: {
    filename: 'main.js',
    path: path.resolve(__dirname, 'dist'),
  },
  devtool: 'inline-source-map',
  devServer: {
    contentBase: './dist'
  },
  module: {},
  plugins: []
};
```

This will use the `src/index.js` file as our entrypoint, and pull in any
files which are referenced by it. The files will get processed into a
`main.js` file and some additional set of files (images, fonts, HTML,
etc.) These will be placed in the `dist/` subdirectory, which should also
be added to the `.gitignore` file.

## Production

So far we've set Webpack up to build in development mode. Our production
builds will look a bit different, since we can elide our source mapping
and we will want to do minification, tree shaking, and asset optimization.
We manage this by separating our Webpack configurations for each
environment.

Note that there will be a fair amount of common configuration between
development and production. This configuration will be kept in its own
file, which will then be merged into the development and production
configurations. This will minimize the amount of redundant code kept
between the configurations.

In order to be able to merge our configurations we'll need to add the
[webpack-merge](https://github.com/survivejs/webpack-merge) utility. This
is done with `npm install --save-dev webpack-merge`, which will update the
`package.json` and `package-lock.json` files.

The `webpack.config.js` file is then split into three. The first,
`webpack.common.js` has the common code:

```
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'main.js',
    path: path.resolve(__dirname, 'dist'),
  },
  module: {},
  plugins: []
};
```

The second, `webpack.dev.js`, contains the development specific code:

```
const merge = require('webpack-merge');
const common = require('./webpack.common.js');

module.exports = merge(common, {
  mode: 'development',
  devtool: 'inline-source-map',
  devServer: {
    contentBase: './dist'
  },
});
```

Finally, we have the `webpack.prod.js` configuration:

```
const merge = require('webpack-merge');
const common = require('./webpack.common.js');

module.exports = merge(common, {
  mode: 'production',
});
```

Once this is done we can add a `"release"` script to `package.json`, and
adjust the other scripts to use the correct configuration file:

```
"scripts": {
  "build": "webpack --config webpack.dev.js",
  "release": "webpack --config webpack.prod.js",
  "serve": "webpack-dev-server --open --config webpack.dev.js",
  "test": "echo \"Error: no test specified\" && exit 1"
}
```

Now we can produce both development and production builds.

## index.html

We will need to include an `index.html` homepage if this is to be used in
a web browser. The simplest way to do this is with the
[`html-webpack-plugin`](https://github.com/jantimon/html-webpack-plugin).
Install the plugin with `npm install html-webpack-plugin --save-dev` and
add the following to `webpack.common.js`:

```
const HtmlWebpackPlugin = require('html-webpack-plugin');
...
module.exports = {
  ...
  plugins: [
    new HtmlWebpackPlugin({
      title: 'Sample'
    })
  ]
  ...
};
```

Spend a bit of time understanding what is going on here. We load the
plugin from the `html-webpack-plugin` module installed by npm, then create
an instance of it under `plugins` using an option object. This can be done
for any number of plugins, including minification and complession.

## CSS

CSS can be processed using the `style-loader` and `css-loader` Webpack
loaders. Since these are npm packages they need to be installed by using
`npm install --save-dev style-loader css-loader`. They are then added to
the `rules` section of `webpack.common.js`:

```
module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      }
    ]
  }
};
```

Spend some time understanding the rule used here, as we will be using this
for other loaders. The `test` key tells us how to detect the files to be
processed with this loader; normally this will be done with a regular
expression. The `use` key tells us what to apply to a given file type. In
this case it tells us to use the `style-loader` and `css-loader` to
process `*.css` files.

The `use` clause can also accpet objects as well as strings, which permits
us to configure loaders. For example:

```
{
  test: /\.css$/,
  use: [
    'style-loader',
    {
      loader: 'css-loader',
      options: {
        sourceMap: true
      }
    }
  ]
}
```

Other possible loaders to consider for use with CSS include `sass-loader`,
`less-loader`, and `stylus-loader`.

## Assets

It is likely that our web application will also require assets of some
sort, such as images and fonts, or possibly some other form of data. The
`file-loader` Webpack loader can handle these sorts of data easily.
Install it with `npm install --save-dev file-loader`, then add the
following to `webpack.common.js`:

```
module.exports = {
  module: {
    rules: [
      {
        test: /\.(png|svg|jpg|gif)$/,
        use: [
          'file-loader'
        ]
      }
    ]
  }
};
```

This should cause the file to be copied to the `dist/` directory once an
`import` statement is added to `index.js` referencing the image. Other
image formats can also be specified, as can font formats (`woff`, `woff2`,
`eot`, `ttf`, `otf`, etc.) Note that the images and fonts can also be
referenced in anything which is in turn referenced by `index.js`, such as
a stylesheet.

Depending on how they are being used, it may also make sense to use
`url-loader`, which will convert the file data into a Base64 encoded
`data:` URL. For small file sizes this can actually speed download times,
since there are fewer requests made. `url-loader` can be configured with
`file-loader` as a fallback for file larger than a certain size.
