# Backbone.js + Express.js SPA boilerplate

This project is supposed to be a *starter kit* helping the community build Single Page Application (SPA) architectures based on Backbone.js and Express.js frameworks.

A Pragmatic approach is key. The aim of this project is to provide the  simplest possible implementation, to just show the way instead of providing a finalized application. It covers aspects of front-end design, as well as open APIs.

You are the one, who can help project to grow. Join now and contribute, your help is much appreciated.

NOTE: It is still in **progress**. If you would like to contribute, please join the discussion/implementation of currently opened [issues](https://github.com/alexanderbeletsky/backbone-express-spa/issues).

## Contents

* [Description](#description)
* [Application example](#example)
* [Installation](#installation)
* [Express.js](#expressjs)
	* [Serving the master page](#serving-master-page)
	* [API end-points](#api-endpoints)
	* [Authorization and CORS](#authorization-cors)
* [Backbone.js](#backbonejs)
	* [RequireJS and CommonJS](#requirejs-and-commonjs)
	* [Routing](#routing)
	* [View Manager](#view-manager)
	* [Applications](#applications)
	* [Main view and subviews](#main-view-and-subviews)
	* [Transitions](#transitions)
* [Testing](#testing)
	* [Backbone.js (front-end) tests](#front-end-tests)
	* [Express.js (back-end) tests](#back-end-tests)
	* [Functional (web driver) tests](#functional-tests)
* [SEO](#seo)
* [Build for production](#build-for-production)
	* [Concatenate and minify](#concatenate-and-minify)
	* [Gzip content](#gzip-content)
	* [Development and production](#gzip-content)
	* [Cache busting](#cache-busting)
	* [Optimization results](#optimization-results)
* [Deployment](#deployment)

<a name="description"/>
## Description

The SPA infrastructure setup can be time consuming. Typical problems are, to configure `requirejs`, to setup the initial routing and view manager or to prevent memory leaks. This project may be used as a starting point to build your own single page application.

This project is a complete and minimal setup for building an SPA running on the `Express.js` framework as the back-end and `Backbone.js` as the front-end.

The concept of SPAs itself is rather simple, but it requires some infrastructure to have in place, before building up a new application. This project already includes the necessary infrastructure.

<a name="example"/>
## Application example

'TheMailer' - simple app for managing emails, contacts, tasks.

<a name="installation"/>
## Installation

Clone github repository,

```
$ git clone git@github.com:alexanderbeletsky/backbone-express-spa.git
```

Install npm dependencies,

```
$ npm install
```

Install bower dependencies,

```
$ bower install
```

Run tests,

```
$ npm test
```

Run app (development mode),

```
$ npm start
```

<a name="expressjs"/>
## Express.js

[Express.js](http://expressjs.com/) is used as back-end development framework. It's simple and easy to configure for SPAs.

In the API-oriented architecture of SPAs, the back-end serves two main purposes:

* Serving the master page HTML
* Providing API end-points for the web client

### Master page

In SPAs there is typically only one [Master page](views/master.ejs), the ) the main HTML file returned from the server. It includes all styles and javascript, provides just a very basic layout with placeholders for the application.

```html
<div class="container">
	<div id="app" class="container"></div>
</div>
```

After the master page is served to the client, the rest of the UI and logic is build by ``Backbone.js``.

<a name="serving-master-page"/>
### Serving master page

To serve the master page, the application uses the custom middleware component [serveMaster.js](source/middleware/serveMaster.js). It serves the master page for any request, except the requests for `/api`, `/components`, `/css/` or `/js`.

<a name="api-endpoints"/>
### API end-points

The API provides JSON based end-points for the HTTP requests. The API source files are located at ``source/api``. Each API module returns a function that takes an ``app`` instance and sets up an HTTP verb handler for the route.

```js
module.exports = function (app) {
	app.get('/api/emails', function (req, res) {
		res.json({status: 'GET /api/users'});
	});

	app.post('/api/emails', function (req, res) {
		res.json({status: 'POST /api/users'});
	});

	app.put('/api/emails/:id', function (req, res) {
		res.json({status: 'PUT /api/users/' + req.params.id});
	});

	app.del('/api/emails/:id', function (req, res) {
		res.json({status: 'DELETE /api/users/' + req.params.id});
	});
};
```

To enable the API end-points, they have to be rquired in the main ``app.js`` file:

```js
// api endpoints
require('./source/api/emails')(app);
require('./source/api/contacts')(app);
require('./source/api/tasks')(app);
```

<a name="authorization-cors"/>
## Authorization and CORS

Designing an open API, authorization is one of the most important topics to consider.

There are many ways authorization can be implemented, and the chosen method will have a strong impact on how well the API will be usable and scalable. THis project is highly inspired by this blog post by [Vinay Sahni](https://twitter.com/veesahni) regarding [Pragmatic RESTFull APIs](http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api).

Traditional authorization algorithms are typically using some kind of `token-based` authorization. That means, each user of an API is getting registered with a special token (pseudo-random looking string), which is issued and associated with that user. The token contains the user identification and some meta info, and is encrypted with some simple algorithm such as `base64`. For each request, the users sends this token (in the HTTP headers of each query) to the server, the server checks for a matching `userId` and treats the request as authenticated if it finds one.

Tokens always have to be transported by a secured channel such as SSL.

The only problem with that approach is, that each request requires at least one *expensive* database call. This can be avoided by applying some cryptography on the authorization procedure. One easy to implement and  scale approache is the usage of self-signed [HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code) tokens.

### HMAC based API authorization

The HMAC routine works as follows:

1. The client signs up to the API by sending his `username` and `password`.
2. The client get registered, the hashed `password` is saved to the `users` collection in the database.
3. The authorization token is *issued*.
4. The client receives the token back.
5. For all further requests, the client sends the token as the `password` in the [authorization header](http://tools.ietf.org/html/rfc1945#section-10.2) of the HTTP request.
6. The server *validates* the token and treats the request as authenticated if it succeeds.

Note that this type of token *validation* procedure doesn't require any database request, since the whole process depends entirely on computation.

### HMAC token

Let's take a look how to issue new authorization token. We typically want to reduce the token lifetime, in case it is stolen. So we include some meta information about the exact time it was issued to the username. This information is concatenated and the HMAC algorithm is applied, using servers *private* key. As a lest step we we add the same info in open text  and encrypt result by `base64`.

```js
var timespamp = moment();
var message = username + ';' + timespamp.valueOf();
var hmac = crypto.createHmac('sha1', AUTH_SIGN_KEY).update(message).digest('hex');
var token = username + ';' + timespamp.valueOf() + ';' + hmac;
var tokenBase64 = new Buffer(token).toString('base64');
```

In this example, the `AUTH_SIGN_KEY` is the server's private key and `tokenBase64` is the final result, which is sent back to client.

### Authorization

The client stores the token to the cookie or to the localstorage and uses it for *each* API request as a part of the basic authentication header.

Request.js example:

```js
request.get({url: url, auth: {user: username, password: token}}, function (err, resp) {
	error = err;
	response = resp;
	done();
});
```

jQuery example:

```js
$.ajax
({
	type: "GET",
	url: "index1.php",
	dataType: 'json',
	async: false,
	username: username,
	password: token,
	success: function (){
		done()
	}
});
```

Backbone.sync example:

```js
Backbone.ajax = function() {
	var defaults = arguments[0];
	_.extend(defaults, {
		username: username,
		password: token
	});

	return Backbone.$.ajax.apply(Backbone.$, arguments);
};
```

### Token validation

When the server receives the token back, the request gets authenticated as follows.

1. The server decodes the token with `base64` and parses out the token information (which is simply split by ';').
2. The server computes HMAC signature of received `username` and `timestamp` using the *same* private server key.
3. It compares it with the HMAC signatuure received in the token.
4. If both HMACs are different, the request is not authorized and the server returns 401.
5. Otherwise, the server checks the token's timestamp, and returns a 401 if it is expired.
6. Otehrwise, the request is authenticated.

If the token is compromised or wrong, the HMAC procedure guarantees that the signatures will never match, except the attacker is aware the of server's private key.

### API Authorization implementation

The authorization API exposes only three methods,

```
/api/auth/signup
/api/auth/login
/api/auth/validate
```

The `signup` method is used as the initial client registration. The `login` method is called each time, new token have to issued. The `validate` method is used to check the token validity (it's used as internal method mostly).

The [source/middleware/auth.js](source/middleware/auth.js) file exposesthe  `createToken` and the `validateToken` functions. The `createToken` function is applied to the `signup` and `login` API methods, and the  `validateToken` function is applied on every API that requires authorization.

The [test/api/auth.specs.js](test/api/auth.specs.js) file specifies how authorization works in detail and the [source/api/auth.js](source/api/auth.js) file specifies the API end-point implementation.

### CORS

CORS ([Cross Origin Resource Sharing](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing)) has to be enabled to allow the API to be used from client applications, running on a different domain than the APIs domain. For instance, the API might be deployed at `https://api.example.com`, and the at `https://app.example.com`.

The CORS middleware function  [/source/middleware/cors.js](/source/middleware/cors.js) takes care of that:

```js
function cors() {
	return function (req, res, next) {
		res.header('Access-Control-Allow-Origin', '*');
		res.header('Access-Control-Allow-Methods', 'GET,PUT,POST,DELETE');
		res.header('Access-Control-Allow-Headers', 'X-Requested-With, X-Access-Token, X-Revision, Content-Type');

		next();
	};
}

module.exports = cors;
```

The CORS middleware function has to be added during the application initialization:

```js
	app.use(middleware.cors());
```

<a name="backbonejs"/>
## Backbone.js

[Backbone.js](http://backbonejs.org/) is the one of most popular front-end development framework (library). It provides abstractions for models, views, collections and able to handle client-side routing.

Front-end architecture is build on modular structure and relying on [AMD](https://github.com/amdjs/amdjs-api/wiki/AMD) to allow build scalable applications.

<a name="requirejs-and-commonjs"/>
### RequireJS and CommonJS

[RequireJS](http://requirejs.org/) picked up as asynchronous javascript module loading. ``RequireJS`` uses it's own style for defining modules, specifying the dependency as array of strings.

```js
define([
	'/some/dep',
	'another/dep',
	'yet/another/dep',
	'text!./templates/template.html,
	jQuery,
	Backbone'], function(SomeDep, AnotherDep, YetAnotherDep, template, $, Backbone) {
		// module implementation...
	});
```

With some time spent on Node.js programming, CommonJS style becomes more convenient to use. Fortunately ``RequireJS`` has [CommonJS](http://requirejs.org/docs/commonjs.html) style implementation.

```js
define(function (require) {
	// dependencies
	var SomeDep = require('/some/dep');
	var AnotherDep = require('another/dep');

	// export
	return {};
});
```

<a name="routing"/>
### Routing

All routing logic is placed in [/core/router.js](public/js/core/router.js). There are 3 routes defined in boilerplate.

Each route handler is responsible for *starting up* new application. Application `run` function takes ``ViewManager`` instance.

<a name="view-manager"/>
### View manager

SPA application typical threat is *memory leaks*. Memory leaks might appear for a few reasons, one of the most famous reason for Backbone applications are, so called, [zombie views](http://lostechies.com/derickbailey/2011/09/15/zombies-run-managing-page-transitions-in-backbone-apps/).

[/core/viewManager.js](public/js/core/viewManager.js) is responsible for disposing views during switch from one router to another.

Besides of that, it handles *transitions* during application switch.

<a name="applications"/>
### Applications

Application is concept of grouping `models`, `collections`, `views` of unit in one place. The rule is, "one route - one application". Router matches the route, loading the application entry point and passes `viewManager` (or any other parameters, like id's or query strings) into application.

All applications are [apps](public/js/apps) folder.

[app.js](public/js/apps/home/app.js) is entry point of application and it's responsible for several things:

* Fetching initial application data
* Instantiating Main View of application

```js
define(function(require) {
	var MainView = require('./views/MainView');

	return {
		run: function (viewManager) {
			var view = new MainView();
			viewManager.show(view);
		}
	};
});
```

<a name="mainview-and-subviews"/>
### Main view and subviews

*Main view* responsible for UI of application. It's quite typically that main view is only instantiating subviews and passing the models/collections further down.

[MainView.js](public/js/apps/home/views/MainView.js) keeps track of subviews in ``this.subviews`` arrays. Each subview will be closed by `ViewManager` [dispose](public/js/core/viewManager.js#L22) function.

```js
var MainView = Backbone.View.extend({
	initialize: function () {
		this.subviews = [];
	},

	render: function () {
		var headerView = new HeaderView();
		this.subviews.push(headerView);
		this.$el.append(headerView.render().el);

		var footerView = new FooterView();
		this.subviews.push(footerView);
		this.$el.append(footerView.render().el);

		return this;
	}
});
```

<a name="templates"/>
### Templates

[Handlebars](http://handlebarsjs.com/) is picked up as templating engine, powered by [require-handlebars-plugin](https://github.com/SlexAxton/require-handlebars-plugin). Templates are stored on application level in `template` folder. Handlebars plugin is configured to keep templates in `.html` files.

View is loading template through `!hbs` plugin and uses that in `render()` function.

```js
var HeaderView = Backbone.View.extend({
	template: require('hbs!./../templates/HeaderView'),

	render: function () {
		this.$el.html(this.template({title: 'Backbone SPA boilerplate'}));
		return this;
	}
});
```

<a name="transitions"/>
## Transitions

Transitions is a very nice feature for single pages applications. It adds the visual effects of switching from one application to another.

Boilerplate is relying on wonderful [animate.css](https://github.com/daneden/animate.css) library. [core/transition.js](public/js/core/transition.js) is responsible for applying transition style. It's being called from [/core/viewManager.js](public/js/core/viewManager.js).

Once you decide to have transitions in your app, simply modify [master.ejs](views/master.ejs) and add ``data-transition`` attribute to application ``div``.

```html
<div class="container">
	<div id="app" class="container" data-transition="fadeOutLeft"></div>
</div>
```

Checkout the list of available transitions on [animate.css](https://github.com/daneden/animate.css) page. You can apply anything you want, please note "Out" transition type is suited the best.

<a name="testing"/>
## Testing

Testing is key of quality product. Both sides (front and back) have to covered with tests to tackle the complexity.

### Execute all tests

To execute all tests, run

```
$ npm test
```

<a name="front-end-tests"/>
### Backbone.js (front-end) tests

TODO.

<a name="back-end-tests"/>
### Express.js (back-end) tests

TODO.

<a name="functional-tests"/>
### Functional (web driver) tests

TODO.

<a name="seo"/>
## SEO

TODO.

<a name="build-for-production"/>
## Build for production

Modern web applications contain a lot of JavaScript/CSS files. While application is *loading* all that recourses have to be in-place, so browser issuing HTTP requests to fetch them. As more application grow, as more requests need to be done.. as slower initial loading is. There are two ways of *optimization* of initial application loading:

* concatenate and minify (decrease HTTP request)
* gzip content (decrease payload size)

Application could operate in several modes - development, production. In development mode, we don't care about optimizations at all. Even more, we are interested to get not processed source code, to be able to debug easily. In production mode, we have to apply as much effort as possible to decrease initial load time.

<a name="concatenate-and-minify"/>
### Concatenate and minify

[RequireJS](http://requirejs.org/) comes together with [optimization](http://requirejs.org/docs/optimization.html) tool, called `r.js`. It's able to concatenate and minify both JavaScript and CSS code.

To simplify the process, we'll use [GruntJS](http://gruntjs.com/) tasks runner. GruntJS is very handy tool, with great community around and very rich contrib library. There is a special task to handle `RequireJS` optimizations, called [grunt-contrib-requirejs](https://github.com/gruntjs/grunt-contrib-requirejs).

[Gruntfile.js](Gruntfile.js) contains all required configuration. To run grunt,

```
$ grunt
```

The result of the grunt run is new folder [/public/build](/public/build/) that contains 2 files: `main.css`, `main.js` - concatenated and minified JavaScript and CSS code.

<a name="gzip-content"/>
### Gzip content

Besides concatenation, it's important to compress resources. Express.js includes this functionality out of the box, as [compress()](http://expressjs.com/api.html#compress) middleware function.

<a name="development-and-production"/>
### Development and production

The configuration distinction goes in [app.js](app.js) file.

```js
app.configure('development', function(){
	app.use(express.errorHandler());							// apply error handler
	app.use(express.static(path.join(__dirname, 'public')));
	app.use(middleware.serveMaster.development());				// apply development mode master page
});

app.configure('production', function(){
	app.use(express.compress());								// apply gzip content
	app.use(express.static(path.join(__dirname, 'public'), { maxAge: oneMonth }));
	app.use(middleware.serveMaster.production());				// apply production mode master page
});
```

[serveMaster.js](source/middleware/serveMaster.js) middleware component is would serve different version of master page, for different mode. In development mode, it would use uncompressed JavaScript and CSS, in production mode, ones that placed in [/public/build](/public/build/) folder.

<a name="cache-busting" />
### Cache busting

Caching is in general good since it helps to application to be loaded faster, but it could hurt while you re-deploy application. Browsers do not track the actual content of file, so if the content has changed, but URL still the same, browser will ignore that.

Besides, different browsers have different caching strategies. IE for instance, is 'famous' with is aggressive caching.

Cache busting is widely adopted technique. There are some different implementations for that, but one of the most effective is: name your resources in the way, so if content has changed the name of resource would change as well. Basic implementation is to prefix file names with **hash** computed on file contents.

Boilerplate uses [grunt-hashres](https://github.com/Luismahou/grunt-hashres) task for that (currently I'm using my own [fork](https://github.com/alexanderbeletsky/grunt-hashres), hope that changes are promoted to main repo soon). That task transforms the [grunt-contrib-requirejs](https://github.com/gruntjs/grunt-contrib-requirejs) output files `main.js`, `main.css` into something like `main-23cbb34ffaabd22d887abdd67bfe5b2c.js`, `main-5a09ac388df506a82647f47e3ffd5187.css`.

It also produces [/source/client/index.js](/source/client/index.js) file [serveMaster.js](source/middleware/serveMaster.js) uses to render production master page correctly.

Now, everything that either `.js` or `.css` content is changed, build would produce new files and they are guaranteed to be loaded by browser again.

<a name="optimization-results"/>
### Optimization results

On a left side you see application running in development mode, on a right side in  production mode.

![optimization results](/public/img/optimizations.png?raw=true)

Even for such small application as 'TheMailer', the benefits are obvious:

* Requests: 55 / 4 ~ 14 times fewer.
* Payload: 756Kb / 43.4Kb ~ 17 times smaller.
* Load time: 898ms / 153ms ~ 6 times faster.

<a name="deployment"/>
## Deployment

TODO.


<a name="generator"/>
## Yeoman generator

TODO.

# Legal Info (MIT License)

Copyright (c) 2013 Alexander Beletsky

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
