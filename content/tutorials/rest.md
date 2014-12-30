---
layout: 'tutorial'
title: 'RESTful APIs'
---

This tutorial illustrates how to implement inter-service communication using a
[RESTful](http://en.wikipedia.org/wiki/Representational_state_transfer) API.

We'll be implementing two processes during the course of this tutorial.  The
first, [`mathd`](https://github.com/bixbyjs-examples/mathd), is a [daemon](http://en.wikipedia.org/wiki/Daemon_%28computing%29)
that services requests for mathematical operations.  The second, [`mathuid`](https://github.com/bixbyjs-examples/mathuid),
provides a web-based interface that allows users to add and subtract numbers.
`mathuid` sends requests to `mathd` to perform the calculations.

### mathd

##### Install Dependencies

Let's start by cloning the `mathd` repository:

```bash
$ git clone https://github.com/bixbyjs-examples/mathd.git
```

and installing dependencies.

```bash
$ cd mathd
$ npm install
```

Let's take a look at [package.json](https://github.com/bixbyjs-examples/mathd/blob/master/package.json).

```json
{
  "name": "mathd",
  "description": "Service providing mathematical operations.",
  "repository": {
    "type": "git",
    "url": "git://github.com/bixbyjs-examples/mathd.git"
  },
  "dependencies": {
    "bixby-common": "git://github.com/bixbyjs/bixby-common.git",
    "bixby-express": "git://github.com/bixbyjs/bixby-express.git",
    "bixby-sd": "git://github.com/bixbyjs/bixby-sd.git",
    "body-parser": "^1.10.0",
    "bootable": "^0.2.4",
    "electrolyte": "0.0.6",
    "express": "^4.10.6",
    "morgan": "^1.5.0"
  }
}
```

This application uses [Express](http://expressjs.com/) as a web framework and
builds upon various component suites provided by Bixby.js.  As is the case with
all Bixby.js-based applications, [Electrolyte](https://github.com/jaredhanson/electrolyte)
is used as an IoC container.  We'll see examples of how all these fit together
we continue.

##### Create Application

The application is created in [app/app.js](https://github.com/bixbyjs-examples/mathd/blob/master/app/app.js).

The first thing we do is configure the IoC container so that it contains the
components required by the application.

```javascript
IoC.use('handlers', IoC.node(__dirname + '/handlers'));
IoC.use(require('bixby-express'));
IoC.use('sd', require('bixby-sd'));
IoC.use(require('bixby-common'));
```

Application-specific request handers are located in the [app/handlers](https://github.com/bixbyjs-examples/mathd/tree/master/app/handlers)
directory and registered under the "handlers" namespace.  Component suites for
[Express](https://github.com/bixbyjs/bixby-express), [service discovery](https://github.com/bixbyjs/bixby-sd),
and [common](https://github.com/bixbyjs/bixby-common) functionality are also utilized.

Next, we use [Bootable](https://github.com/jaredhanson/bootable) to control the
boot sequence of the application.

```javascript
app.phase(bootable.di.initializers(__dirname + '/init'));
app.phase(bootable.di.routes(__dirname + '/routes'));
app.phase(IoC.create('sd/registry'));
app.phase(IoC.create('boot/httpserver'));
app.phase(IoC.create('sd/boot/announce'));
```

First, the initializers will be run and routes will be drawn.  After that, the
application will connect to the service registry and start the HTTP server to
listen for requests.  Finally, the application will announce its services in the
registry so that other applications can discover them.

##### Draw Routes

The routes supported by the application are drawn in [app/routes.js](https://github.com/bixbyjs-examples/mathd/blob/master/app/routes.js).

```javascript
exports = module.exports = function(IoC) {

  this.post('/add', IoC.create('handlers/add'));
  this.post('/sub', IoC.create('handlers/sub'));
  
};

exports['@require'] = [ '$container' ];
```

Here we encounter our first use of dependency injection using the IoC container.
`$container` is a special component that refers to the IoC container itself, which
can be used to create components.

In this case, all request handlers are implemented as components.  This eliminates
boilerplate code as the handlers themselves can take advantage of dependency
injection.  Let's take a look at [app/handlers/add.js](https://github.com/bixbyjs-examples/mathd/blob/master/app/handlers/add.js)
to see how this works.

```javascript
var bodyParser = require('body-parser');

exports = module.exports = function(logger) {
  
  function add(req, res, next) {
    logger.debug('adding operands: ' + req.body.operands.join(', '));
    
    var operands = req.body.operands
      , result = 0
      , i, len;
    for (i = 0, len = operands.length; i < len; ++i) {
      result += operands[i];
    }
    res.send({ operands: operands, result: result });
  }

  return [ bodyParser.json(),
           add ];
  
}

exports['@require'] = [ 'logger' ];
```

This module exports a _factory function_ which returns a function chain used to
handle requests.  The processing here is straightforward for the sake of a
simple example.  The body is parsed and then any operands within the body are
summed and the result is returned.

Notice that the `add(req, res, next)` function logs the operands for debugging
purposes.  The logger is injected by virtue of the `@require` annotation.

```javascript
exports['@require'] = [ 'logger' ];
```

When `IoC.create('handlers/add')` is invoked, the IoC container will
automatically instantiate the logger and inject it into the exported factory
function as the `logger` parameter.  The factory function then returns the
function chain:

```javascript
return [ bodyParser.json(),
         add ];
```

which is mounted as the `/add` route in the Express application.

```javascript
this.post('/add', IoC.create('handlers/add'));
```

Take a look at [app/handlers/sub.js](https://github.com/bixbyjs-examples/mathd/blob/master/app/handlers/sub.js)
which operates similarly.

##### Listen for HTTP Requests

The HTTP server begins listening for requests when the boot phase created by
`IoC.create('boot/httpserver')` is executed.  This phase is supplied by
[bixby-express](https://github.com/bixbyjs/bixby-express).

The settings that determine how the HTTP server operates are specified in
[etc/development.toml](https://github.com/bixbyjs-examples/mathd/blob/master/etc/development.toml).

```toml
[server]
port = 0
```

In this case, the port is set to `0`, which indicates that the system will
assign the server a random port.  We'll use this to our advantage later.

##### Announce Services

With the HTTP server running, the services provided by `mathd` are announced so
that they can be discovered and used by other applications.  This is done when
the boot phase created by `IoC.create('sd/boot/announce')`, supplied by [bixby-sd](https://github.com/bixbyjs/bixby-sd),
is invoked.

The services provided by the application are declared in [app/services.json](https://github.com/bixbyjs-examples/mathd/blob/master/app/services.json).

```json
{
    "math.common.": {
        "http://schemas.example.com/api/math/v1": "/"
    }
}
```

Service discovery is one of the key pieces of functionality provided by
Bixby.js.  A service registry provides a way for an application to discover
services provided by other applications.  This example is configured to use
[etcd](https://github.com/coreos/etcd) as a service registry.

```toml
[sd]
url = "etcd://127.0.0.1:4001"
```

The JSON above indicates that the service _type_ `http://schemas.example.com/api/math/v1`
is available within the `math.common.` _domain_.  A domain is simply some
logical grouping of functionality.  In this example, `math.common.` was chosen
to indicate mathmatical operations made available to any other application that
wishes to use them.  The service type is a well-known string that identifes the
protocol or API implemented by the service.

##### Start Server

Let's go ahead and start the server.

```bash
$ npm start
```

If all goes well, you should see output similar to the following:

```bash
info: Connecting to service registry 127.0.0.1:4001
debug: Connected to service registry 127.0.0.1:4001
info: HTTP server listening on 0.0.0.0:64509
info: Announcing service http://schemas.example.com/api/math/v1 in math.common. at http://10.200.1.85:64509/
```

Note the port assigned to the server and send it a request.

```bash
$ curl -X POST -H "Content-Type: application/json" --data "{\"operands\":[1,2]}" http://127.0.0.1:64509/add
{"operands":[1,2],"result":3}
```

Great!  `mathd` is operational.  Let's leave it running, open a new terminal,
and proceed with `mathuid`.


### mathuid

##### Install Dependencies

We'll start off with this application in a similar manner.  Clone the `mathuid` repository:

```bash
$ git clone https://github.com/bixbyjs-examples/mathuid.git
```

and install dependencies.

```bash
$ cd mathuid
$ npm install
```

##### Create Application

The setup for `mathuid` is nearly identical to that of `mathd`.  Take moment to
explore [package.json](https://github.com/bixbyjs-examples/mathuid/blob/master/package.json),
[app/app.js](https://github.com/bixbyjs-examples/mathuid/blob/master/app/app.js),
[app/services.json](https://github.com/bixbyjs-examples/mathuid/blob/master/app/services.json),
and [etc/development.toml](https://github.com/bixbyjs-examples/mathuid/blob/master/etc/development.toml).

Notice the similarities.  This boilerplate is kept to a bare minimum and will be
found in most applications that use Bixby.js.

##### Draw Routes

Again, the routes available in this application are drawn in [app/routes.js](https://github.com/bixbyjs-examples/mathuid/blob/master/app/routes.js).

```javascript
exports = module.exports = function(IoC) {

  this.get('/add', IoC.create('handlers/add/show'));
  this.post('/add', IoC.create('handlers/add/calc'));
  
  this.get('/sub', IoC.create('handlers/sub/show'));
  this.post('/sub', IoC.create('handlers/sub/calc'));
  
};

exports['@require'] = [ '$container' ];
```

`mathuid` presents a form to the user in which two numbers can be entered.  When
submitted, the calculation is performed and the result is displayed to the user.
Let's take a look at how addition is implemented.

[app/handlers/add/show.js](https://github.com/bixbyjs-examples/mathuid/blob/master/app/handlers/add/show.js)
renders a form for display in the user's browser.

```javascript
exports = module.exports = function() {
  
  function respond(req, res, next) {
    res.render('add');
  }
  
  return [ respond ];
  
}

exports['@require'] = [];
```

The form itself is contained in the [app/views/add.ejs](https://github.com/bixbyjs-examples/mathuid/blob/master/app/views/add.ejs)
template.

When the form is submitted, the request is processed by [app/handlers/add/calc.js](https://github.com/bixbyjs-examples/mathuid/blob/master/app/handlers/add/calc.js).

```javascript
var request = require('request')
  , bodyParser = require('body-parser')
  , errorHandler = require('errorhandler');

exports = module.exports = function(registry, logger) {
  
  function add(req, res, next) {
    registry.resolve('math.common.', 'http://schemas.example.com/api/math/v1', function(err, records) {
      if (err) { return next(err); }
      
      var baseURL = records[0];
      if (baseURL[baseURL.length - 1] != '/') { baseURL += '/'; }
      
      logger.debug('Sending add request to ' + baseURL);
      request.post({
        url: baseURL + 'add',
        body: { operands: [ parseFloat(req.body['1']), parseFloat(req.body['2']) ] },
        json: true,
        timeout: 60000
      }, function(err, resp, body) {
        if (err) { return next(err); }
        res.locals.operands = body.operands;
        res.locals.result = body.result;
        next();
      });
    });
  }
  
  function respond(req, res, next) {
    res.render('add-result');
  }

  return [ bodyParser.urlencoded(),
           add,
           respond,
           errorHandler() ];
  
}

exports['@require'] = [ 'sd/registry', 'logger' ];
```

Here things get interesting and begin to show the benefits of service discovery.
Let's walk through the important bits.

As is the case with the handlers in `mathd`, we are using dependency injection.
This time, though, the service registry is required in addition to the logger.

```javascript
exports['@require'] = [ 'sd/registry', 'logger' ];
```

When `IoC.create('handlers/add/calc')` is invoked, the IoC container will
automatically instantiate both the service registry and the logger and inject
them into the exported factory function as `registry` and `logger` parameters,
respectively.

The `add(req, res, next)` function does not perform any calculations, but rather
uses the service provided by `mathd`.  Instances of that service are found by
resolving them using the service registry.

```javascript
registry.resolve('math.common.', 'http://schemas.example.com/api/math/v1', function(err, records) {
  if (err) { return next(err); }
      
  var baseURL = records[0];
  // ...
}
```

`mathd` implements the service _type_ `http://schemas.example.com/api/math/v1`
and announces it within the `math.common.` _domain_.  Resolving that service
results in an array of locations, one element for each running instance.  The
first location is chosen, and a request is sent, using the excellent [request](https://github.com/request/request)]
package.

```javascript
request.post({
  url: baseURL + 'add',
  body: { operands: [ parseFloat(req.body['1']), parseFloat(req.body['2']) ] },
  json: true,
  timeout: 60000
}, function(err, resp, body) {
  if (err) { return next(err); }
  res.locals.operands = body.operands;
  res.locals.result = body.result;
  next();
});
```

Once the result is obtained, it is displayed to the user.

```javascript
res.render('add-result');
```

##### Start Server

Let's start the application.

```bash
$ npm start
info: Connecting to service registry 127.0.0.1:4001
debug: Connected to service registry 127.0.0.1:4001
info: HTTP server listening on 0.0.0.0:8080
info: Announcing service http://schemas.example.com/ui/math in math.example.local. at http://10.200.1.85:8080/
```

Navigate to [http://127.0.0.1:8080/add](http://127.0.0.1:8080/add) and add
some numbers.  Excellent!  `mathuid` is serving web forms and `mathd` is
performing calculations.

##### Scale Out

This example is trivial, but lets imagine that mathematical operations were so
popular that we could no longer perform them in using a single machine.  All
we have to do is start another instance of `mathd`.

```bash
$ cd mathd
$ npm start
info: Connecting to service registry 127.0.0.1:4001
debug: Connected to service registry 127.0.0.1:4001
info: HTTP server listening on 0.0.0.0:49355
info: Announcing service http://schemas.example.com/api/math/v1 in math.common. at http://10.200.1.85:49355/
```

Now go ahead and submit some more numbers to be added.  You'll notice that
`mathuid` will send requests to one or the other of the two running `mathd`
instances.  When the instances are resolved, the resulting array will be
randomized.  Picking the first element in the array will balance the load
across the two instances in a roughly equal way.  If load increases further
and two instances are no longer sufficent, simply spin up more instances
as needed.

### Summary





