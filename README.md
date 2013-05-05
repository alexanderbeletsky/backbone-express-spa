# Backbone.js + Express.js SPA boilerplate

To build single pages application is seconds, not hours.

SPA infrastruction setup could be time consuming. It's typicall problem, to configure `requirejs`, intial routing and view manager, to prevent memory leaks. This project could be used as good start to build own single page application.

## Application

'TheMailer' - simple app for managing emails, contacts, tasks.

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

Run app (development mode),

```
$ node app.js
```

## Description

This project is complete and minimal setup for building single page applications running on ``Express.js`` framework as back-end and ``Backbone.js`` as front-end.

SPA itself is rather simple concept, but it requires some infrascture to have in place, before build up new application. This project already includes this infrastructure.

## Express.js

[Express.js](http://expressjs.com/) is used as back-end development framework. It's simple and easy to configure for SPA.

In API-oriented architecture back-end is responsible for 2 main purposes:

* Serving master page html
* Providing API end-points for web client

### Master page

[Master page](views/master.ejs) is main (and typically one) html page returned from server. It includes all styles and javascript, provides very basic layout and placeholder for application.

```html
<div class="container">
	<div id="app" class="container"></div>
</div>
```

After master page is served back to client the rest of UI and logic is build by ``Backbone.js``.

### Serving master page

To serve master pages application includes middleware component [serveMaster.js](source/middleware/serveMaster.js). It would respond with master page html for any request, expect the requests for `/api`, `/components`, `/css/` or `/js`.

### API end-points

API is HTTP, JSON based end-points. Sources are located at ``sourse/api``. Each API module retunrs a function that takes ``app`` instance and setup HTTP verb handler for a route.

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

To enable API end-point, you should modify ``app.js`` file, like

```js
// api endpoinds
require('./source/api/emails')(app);
require('./source/api/contacts')(app);
require('./source/api/tasks')(app);
```

## Backbone.js

[Backbone.js](http://backbonejs.org/) is the one of most popular front-end development framework (library). It provides abstractions for models, views, collections and able to handle client-side routing.

Front-end architecture is build on modular structure and relying on [AMD](https://github.com/amdjs/amdjs-api/wiki/AMD) to allow build scallable applications.

### RequireJS and CommonJS

[RequireJS](http://requirejs.org/) picked up as asynchronous javascript module loading. ``RequireJS`` uses it's own style for defining modules, specifying the dependency as array of strings.

```js
define([
	'/some/dep',
	'another/dep',
	'yet/another/dep',
	'text!./templates/template.html,
	jQuery,
	Backbone'], function(SomeDep, AnotherDep, YetAnotherDep, template, $, Backbobe) {
		// module implementation...
	});
```

With some time spent on Node.js programming, CommonJS style becomes more convenient to use. Fortunatelly ``RequireJS`` has [CommonJS](http://requirejs.org/docs/commonjs.html) style implementation.

```js
define(function (require) {
	// dependencies
	var SomeDep = require('/some/dep');
	var AnotherDep = require('another/dep');

	// export
	return {};
});
```

### Routing

All routing logic is placed in [/core/router.js](public/js/core/router.js). There are 3 routes defined in boilerplace.

Each route handler is responsible for *starting up* new application. Application `run` function takes ``ViewManager`` instance.

### View manager

SPA application typical threat is *memory leaks*. Memory leaks might appear for a few reasons, one of the most famous reason for Backbone applications are, so called, [zombie views](http://lostechies.com/derickbailey/2011/09/15/zombies-run-managing-page-transitions-in-backbone-apps/).

[/core/viewManager.js](public/js/core/viewManager.js) is responsible for disposing views during switch from one router to another.

### Applications

Application is concept of grouping `models`, `collections`, `views` of unit in one place. The rule is, "one route - one application". Router matches the route, loading the application entry point and passes `viewManager` (or any other parameters, like id's or query strings) into application.

All applications are [apps](public/js/apps) folder.

[app.js](public/js/apps/home/app.js) is entry point of application and it's responsible for several things:

* Fetching intial application data
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

### Templates

[Handlebars](http://handlebarsjs.com/) is picked up as templating engine, powered by [require-handlebars-plugin](https://github.com/SlexAxton/require-handlebars-plugin). Templates are stored on application level in `template` folder. Handlebars plugin is configured to keep templates in `.html` files.

View is loading template throught `!hbs` plugin and uses that in `render()` function.

```js
var HeaderView = Backbone.View.extend({
	template: require('hbs!./../templates/HeaderView'),

	render: function () {
		this.$el.html(this.template({title: 'Backbone SPA boilerplate'}));
		return this;
	}
});
```

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
