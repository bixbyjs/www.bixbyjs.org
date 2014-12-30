---
layout: 'tutorial'
title: 'RESTful APIs'
---

This tutorial illustrates how to implement inter-service communication using a
[RESTful](http://en.wikipedia.org/wiki/Representational_state_transfer) API.

We'll be implementing two processes during the course of this tutorial.  The
first, [mathd](https://github.com/bixbyjs-examples/mathd), is a [daemon](http://en.wikipedia.org/wiki/Daemon_%28computing%29)
that services requests for mathematical operations.  The second, [mathuid](https://github.com/bixbyjs-examples/mathuid),
provides a web-based interface that allows users to add and subtract numbers.
`mathuid` sends requests to `mathd` to perform the calculations.

### mathd

##### Install Dependencies

Let's start by cloning the `mathd` repository:

```bash
ssh clone git@github.com:bixbyjs-examples/mathd.git
```

and installing dependencies.

```bash
cd mathd
npm install
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

This service uses [Express](http://expressjs.com/) as a web framework and builds
upon various component suites provided by Bixby.js.  As is the case with all
Bixby.js-based applications, [Electrolyte](https://github.com/jaredhanson/electrolyte)
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

Next, we declare the boot sequence for the application.

```
app.phase(bootable.di.initializers(__dirname + '/init'));
app.phase(bootable.di.routes(__dirname + '/routes'));
app.phase(IoC.create('sd/registry'));
app.phase(IoC.create('boot/httpserver'));
app.phase(IoC.create('sd/boot/announce'));
```

First, the initializers will be run and routes will be drawn.  After that, the
application will connect to the service registry and start the HTTP server to
listen for requests.  Finally, it will application will announce its services in
the registry so that other applications can discover them.

##### Draw Routes

The routes supported by this application are drawn in [app/routes.js](https://github.com/bixbyjs-examples/mathd/blob/master/app/routes.js).

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

Notice that the `add` function logs the operands for debugging purposes.  The
logger is requested and injected by virtue of the `@require` annotation.

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
