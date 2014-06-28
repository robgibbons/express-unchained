Express... Unchained
=========

As wonderful as the [Express](https://github.com/visionmedia/express) framework is, many people have voiced concern that it lacks enough structure, especially compared to popular web frameworks like Django. While it's fairly easy to establish a more modular project structure, it's may not always be clear how to get there. 

Unchained is a [Node.js](https://github.com/joyent/node) module which abstracts the underlying Express framework, providing a clear MVC structure for your Node.js projects. Unchained breaks everything into pieces, and maps it all to Express for you. Unchained aims to provide a simple layer of abstraction above Express, and is fully compatible with existing Express modules and middleware.

### How's it work?

Unchained takes care of requiring Express for you, as well as pulling together all of your views, models and middleware. To define a view, model, or middleware function, it's as easy as creating a .js file in the appropriate folder. Routes are defined declaratively with a simple dictionary (object literal) inside urls.js. Template rendering is provided out of the box with [Swig](https://github.com/paularmstrong/swig) ([Django](https://github.com/django/django)-style templates). Control all of your Express app's boilerplate settings inside config.js.

A typical project structure starts out with the following:

    /middleware
    /models
    /node_modules
    /templates
    /views
    app.js
    config.js
    urls.js

## Getting Started

Get started by installing ([npm install unchained](https://www.npmjs.org/package/unchained)) and requiring Unchained inside your **app.js**. Pass in the root module and app directory to allow Unchained to require the rest of your modules:

```javascript
// app.js
app = require('unchained')(module, __dirname);
```

### urls.js

If you're building an app, you might need to define some routes. In your main app directory, create the **urls.js** module. Route definitions are stored here as a simple dictionary (Object-literal), with keys defining routes, and values specifying matched controllers:

```javascript
// urls.js
module.exports = {
    '/': view.Home,
    '/about/': view.About,
    '/profile/': view.Profile,
};
```

### /views, /models, /middleware, oh my

All view, model and middleware components are defined as .js modules in their assigned folder, with the name of the module specifying the name of the component. Modules are automatically namespaced under the globals **view**, **model** and **m** (for middleware).

### Views

To create a new view called view.Profile, create a .js file in the **/views** directory named Profile.js:

```javascript
// views/Profile.js
// Simple Function-based view
module.exports = function (req, res) {
    res.render('profile'); // Renders /templates/profile.html
};
```

View definitions can be composed of Functions, Objects or Arrays. The Function-based view above does not specify any specific HTTP method, so by default it matches **all** HTTP methods. You can override the default HTTP method within **config.js**. Or, you can just define your view as an Object-literal. With an Object-literal view, you can specify explicit HTTP methods for a given route:

```javascript
// views/Profile.js
// Object-literal syntax (Explicit HTTP Methods)
module.exports = {
    get: function (req, res) {
        res.render('profile');
    },
    post: function (req, res) {
        // Do something with POST request
        res.render('profile');
    }
};
```

### Middleware in Views

You can assign Route-Specific Middleware directly to your views by wrapping them with an Array, always passing your view object as the last item in the Array. Any number of middleware functions may be passed in this style:

```javascript
// views/Profile.js
// Array syntax (Route Middleware) -- Maps to all()
module.exports = [m.requireLogin, m.exampleWare, function (req, res) {
    res.render('profile');
}];
```

Either type of view object can be wrapped with a middleware Array, whether it be a simple Function-based view as above, or an Object-literal view, with multiple HTTP methods defined:

```javascript
// views/Profile.js
// Wrapping both HTTP methods with Middleware
module.exports = [m.requireLogin, m.exampleWare, {
    get: function (req, res) {
        res.render('profile');
    },
    post: function (req, res) {
        // Do something with POST request
        res.render('profile');;
    }
}];
```

You can choose to wrap only a specific HTTP method with a middleware Array, instead of the entire view Object (which assigns to each of the methods defined):

```javascript
// views/Profile.js
// Wrapping a single HTTP method with Middleware
module.exports = {
    get: function (req, res) {
        res.render('profile');
    },
    post: [m.requireLogin, function (req, res) {
        // Do something with POST request
        res.render('profile');
    }]
}];
```

Unchained also allows **nested** Middleware definitions within Object-literal views. Notice the view below, wrapped with a middleware Array, and its POST method wrapped again inside. Middleware nested in this style is executed **outside-in**, so POST requests received by the view will first call requireLogin, then validateInput. GET requests will call only the requireLogin middleware:

```javascript
// views/Profile.js
// Nested middleware in Object-literal view
module.exports = [m.requireLogin, {
    get: function (req, res) {
        res.render('profile');
    },
    post: [m.validateInput, function (req, res) {
        // Do something with POST request
        res.render('profile');
    }]
}];
```

### Middleware in urls.js

If you prefer, you can declare your route-specific middleware directly in urls.js. The middleware Array syntax is the same, with view objects passed as the last Array item:

```javascript
// urls.js
// Middleware assigned directly in Routes
module.exports = {
    '/': view.auth('home'),
    '/about/': [m.requireLogin, view.About],
    '/profile/': [m.requireLogin, view.Profile],
    '/login/': {
        get: [m.redirectUser, view.render('login')],
        post: [m.loginUser, view.redirect('/')],
    },
    '/logout/': [m.logoutUser, view.redirect('/login')],
    '/error/(:err_no)?/?': view.Error,
    '*': view.redirect('/error/404/'),
};
```

You might have noticed a few helper methods above, attached to the view object. The methods **view.auth**, **view.render** and **view.redirect** are actually reusable view Generators, which take in arguments and return customized views. The /about and /profile view definitions above are functionally equivalent to **view.auth**.

Generator methods can leverage middleware, models, and can be created like normal modules. You can define them inside **/views**, **/models** and **/middleware**, but I recommend storing them in the **index.js** of their respective folder.

### Global Middleware & Bare-Metal Express

You can easily apply urls.js to declare your "global" middleware. But you also have the option to configure your Express app directly, within **config.js**:


```javascript
// config.js
// Configuring global middleware and other Express options
module.exports = function (app) {

    app.set('listen_port', 8080);
    app.set('default_method', 'get'); // Default HTTP method for basic view Functions
    app.set('view engine', 'html');
    app.set('views', app.get('root_dir') + '/templates');
    app.engine('html', swig.renderFile);
    app.enable('strict routing');
    app.use(m.addSlashes());
    app.use(express.urlencoded());
    app.use(express.json());
    app.use(express.logger());
    app.use(express.cookieParser());
    app.use(express.methodOverride());
    app.use(express.session({ secret: '.PLEASE_CHANGE-ME*1a2b3c4d5e6f7g8h9i0j!' }));
    app.use(passport.initialize());
    app.use(passport.session());

    return app;
};
```
