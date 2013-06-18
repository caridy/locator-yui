Express YUI
===========

[Express][] extension for YUI Applications.

[![Build Status](https://travis-ci.org/yahoo/express-yui.png?branch=master)][Build Status]


[Express]: https://github.com/visionmedia/express
[Build Status]: https://travis-ci.org/yahoo/express-yui


Goals & Design
--------------

This compontent extends express by adding a new member `app.yui` to the
express application. It is responsible for controlling and exposing the yui
configuration and the app state into the client side as well has controlling
the yui instance on the server.


Installation
------------

Install using npm:

```shell
$ npm install express-yui
```


Features
--------

### Features

 * expose yui config and seed files per request
 * provide basic configurations for cdn, debug, and other common conditions in yui
 * provide middleware to server `static` assets from origin server, including
combo capabilities built-in.
 * provide middleware to `expose` the app state and the yui config into the
 view engine to be used in the templates to boot YUI and the app in the client side.


### Other features

 * built-in integration with [locator][] component to analyze and build applications and dependencies using [shifter][].
 * provide basic express view engine to rely on views registered at the server side thru the `app.yui.use()` as compiled templates.

[locator]: https://github.com/yahoo/locator
[shifter]: https://github.com/yui/shifter


Usage
-----

### Extending express functionalities

`express-yui` is a conventional `express` extension, which means it will extend
the functionalities provided on `express` by augmenting the express app instance
with a new member called `yui`. At the same time, `express-yui` provides a set of
static methods that you can call directly off the `express-yui` module, those
methods are utility methods and express middleware.

Aside from that, `express-yui` will try to extend the `express` peer dependency to
augment the app instance automatically everytime you call `express()` to create a
brand new instance. This is useful, and in most cases, it is just enough. Here is an example:

```
var express = require('express'),
    yui = require('express-yui'),
    app = express();

app.yui.applyConfig({ fetchCSS: false });
```

As you can see in the example above, the `yui` member is available off the app instance.
But this is not always the case, sometimes you have a 3rd party module that is requiring
`express`, and even creating the app under the hood, in which case you can just augment
an existing express app instance by using the static utility `augment`, this is how:

```
var yui = require('express-yui'),
    express = require('express'),
    app = express();

// calling a yui static method to augment the `express` app instance
yui.augment(app);

app.yui.applyConfig({ fetchCSS: false });
```


### Exposing app state into client

To expose the state of the app, which includes the yui configuration computed based
on the configuration defined thru the express app instance, you can call the `expose`
middleware for any particular route:

```
var yui = require('express-yui'),
    express = require('express'),
    app = express();

app.get('/foo', yui.expose(), function (req, res, next) {
    res.render('templates/foo');
});
```

By doing `yui.expose()`, `express-yui` will provision a property call `state` that
can be use in your templates as a `javascript` blob that sets up the page to run
YUI with some very specific settings coming from the server. If you use `handlebars`
you will do this:

```
<script>{{{state}}}</script>
<script>
app.yui.use('node', function (Y) {
    Y.one('body').setContent('<p>Ready!</p>');
});
</script>
```

And this is really the only thing you should do in your templates to get YUI ready to roll!


### Using the locator plugin to build the app

`express-yui` provides many features, but the real power of this package can be seeing when
using it in conjunction with [locator][] component.

```
var express = require('express'),
    yui = require('express-yui'),
    app = express();

// serving static yui modules built by locator
app.use(yui.static());

app.get('/foo', yui.expose(), function (req, res, next) {
    res.render('templates/foo');
});

// using locator to analyze the app and its dependencies
new (require('locator'))({
    buildDirectory: 'build'
}).plug(app.yui.plugin({
    // provision any yui module to be available on the client side
    registerGroup: true,
    // only needed if you want yui modules available on the server runtime as well
    registerServerModules: true,
    // only needed if you want yui modules to be attached automatically on the server runtime
    useServerModules: true
})).parseBundle(__dirname, {});

app.listen(8080);
```

As a result, any yui module under `__dirname` folder or any npm dependency marked as
a locator bundle will be built by the `express-yui`'s locator plugin, and automatically
become available on the client, and potentially on the server as well. This means you
no longer need to manually define loader metadata or any kind of yui config to load those
modules, and `express-yui` will be capable to handle almost everthing for you.


### Using yui modules on the server side

Using modules on the server is exactly the same that using them on the client thru
`app.yui.use()` statement. Here is an example of the use of yql module to load the
weather forecast and passing the result into the template:

```
app.get('/forecast', yui.expose(), function (req, res, next) {
    req.app.yui.use('yql', function (Y) {
        Y.YQL('select * from weather.forecast where location=90210', function(r) {
            // r contains the result of the YQL Query
            res.render('templates/forecast', {
                result: r
            });
        });
    });
});
```

_note: remember that `req.app` holds a reference to the `app` object for conviniece._


### Registering yui groups manually


If you are not using [locator][] component for whatever reason, you will be responsible
for building yui modules, grouping them into yui groups and registering them thru
`app.yui.registerGroup` method. Here is how you register a folder that has the build
result with all yui modules compiled thru [shifter][]:

```
app.yui.registerGroup('foo', 'path/to/foo-1.2.3');
```

Again, this is absolutely not needed if you use [locator][].

### Serving static assets from app origin

Ideally, you will use a CDN to serve all static assets for your application, but your
express app is perfectly capable to do so, and even serve as origin server for your CDN.

```
app.yui.setCoreFromAppOrigin();
app.use(yui.static());
```

With this configuration, a group called `foo` with version `1.2.3`, and `yui` version `3.10.2`, it will produce urls like these:

  * /combo~/yui-3.10.2/yui-base/yui-base-min.js~/foo-1.2.3/bar/bar-min.js~/foo-1.2.3/baz/baz-min.js
  * /yui-3.10.2/yui-base/yui-base-min.js
  * /foo-1.2.3/bar/bar-min.js

Any of those urls will be valid because `express-yui` static middleware will serve them and
combo them when needed based on the configuration of yui.

### Serving static assets from CDN

If you plan to serve the `build` folder, generated by [locator][], from your CDN, then make
sure you set the proper configuration for all groups so loader can know about them.
Here is the example:

```
app.yui.setCoreFromCDN(); // this is the default config btw
app.set('yui combo config', {
    comboBase: 'http://mycdn.com/path/to/combo?',
    comboSep: '&',
    maxURLLength: 1024
});
app.set('yui default base', 'http://mycdn.com/path/to/static/{{groupDir}}/');
app.set('yui default root', 'static/{{groupDir}}/');
```

In this case you don't need to use `yui.static` middleware since you are not
serving local files, unless the app should work as origin server.

With this configuration, a group called `foo` with version `1.2.3` will produce urls like these:

  * http://mycdn.com/path/to/combo?static/foo-1.2.3/bar/bar-min.js&static/foo-1.2.3/baz/baz-min.js
  * http://mycdn.com/path/to/static/foo-1.2.3/bar/bar-min.js


API Docs
--------

You can find the [API Docs][] under `apidocs` folder, and you can browse it thru this url:

http://rawgithub.com/yahoo/express-yui/master/apidocs/index.html

[API Docs]: https://github.com/yahoo/express-yui/tree/master/apidocs


License
-------

This software is free to use under the Yahoo! Inc. BSD license.
See the [LICENSE file][] for license text and copyright information.

[LICENSE file]: https://github.com/yahoo/express-yui/blob/master/LICENSE.md


Contribute
----------

See the [CONTRIBUTE file][] for info.

[CONTRIBUTE file]: https://github.com/yahoo/express-yui/blob/master/CONTRIBUTE.md
