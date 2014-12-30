---
layout: 'tutorial'
title: 'Message Queues'
---

This tutorial illustrates how to use a [message queue](http://en.wikipedia.org/wiki/Message_queue)
for inter-service communication.  Message queues are often used to distribute
tasks to background worker processes, such as sending email or syncing profile
data from a social network.

We'll be implementing two processes during the course of this tutorial.  The
first, [`smsuid`](https://github.com/bixbyjs-examples/smsuid), is a [daemon](http://en.wikipedia.org/wiki/Daemon_%28computing%29)
that provides a web-based interface that allows users to send text messages.
The second, [`smsd`](https://github.com/bixbyjs-examples/smsd) is a worker
process that actually transmits the text messages using a SMS gateway.

### smsuid

##### Install Dependencies

Let's start by cloning the `smsuid` repository:

```bash
$ git clone https://github.com/bixbyjs-examples/smsuid.git
```

and installing dependencies.

```bash
$ cd smsuid
$ npm install
```

Let's take a look at [package.json](https://github.com/bixbyjs-examples/smsuid/blob/master/package.json).

```json
{
  "name": "smsuid",
  "description": "UI for sending text messages.",
  "repository": {
    "type": "git",
    "url": "git://github.com/bixbyjs-examples/smsuid.git"
  },
  "dependencies": {
    "bixby-common": "git://github.com/bixbyjs/bixby-common.git",
    "bixby-crane": "git://github.com/bixbyjs/bixby-crane.git",
    "bixby-express": "git://github.com/bixbyjs/bixby-express.git",
    "bixby-sd": "git://github.com/bixbyjs/bixby-sd.git",
    "body-parser": "^1.10.0",
    "bootable": "^0.2.4",
    "ejs": "^1.0.0",
    "electrolyte": "0.0.6",
    "errorhandler": "^1.3.0",
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

The application is created in [app/app.js](hhttps://github.com/bixbyjs-examples/smsuid/blob/master/app/app.js).

The first thing we do is configure the IoC container so that it contains the
components required by the application.

```javascript
IoC.use('handlers', IoC.node(__dirname + '/handlers'));
IoC.use(require('bixby-express'));
IoC.use('mq', require('bixby-crane'));
IoC.use('sd', require('bixby-sd'));
IoC.use(require('bixby-common'));
```

Application-specific request handers are located in the [app/handlers](https://github.com/bixbyjs-examples/smsuid/tree/master/app/handlers)
directory and registered under the "handlers" namespace.  Component suites for
[Express](https://github.com/bixbyjs/bixby-express), [message queues](https://github.com/bixbyjs/bixby-crane),
[service discovery](https://github.com/bixbyjs/bixby-sd), and [common](https://github.com/bixbyjs/bixby-common)
functionality are also utilized.

Next, we use [Bootable](https://github.com/jaredhanson/bootable) to control the
boot sequence of the application.

```javascript
app.phase(bootable.di.initializers(__dirname + '/init'));
app.phase(bootable.di.routes(__dirname + '/routes'));
app.phase(IoC.create('sd/registry'));
app.phase(IoC.create('mq/adapter'));
app.phase(IoC.create('boot/httpserver'));
app.phase(IoC.create('sd/boot/announce'));
```

First, the initializers will be run and routes will be drawn.  After that, the
application will connect to both the service registry and message queue, and
then start the HTTP server to listen for requests.  Finally, the application
will announce its services in the registry so that other applications can
discover them.

##### Configuration

The settings for this application are located in [etc/development.toml](https://github.com/bixbyjs-examples/smsuid/blob/master/etc/development.toml).

```
[sd]
url = "etcd://127.0.0.1:4001"

[mq]
url = "amqp://10.200.0.215:5672"
exchange = "amq.direct"
queues = [ "message/sms" ]

[logger]
level = "debug"
```

This example is configured to use [etcd](https://github.com/coreos/etcd) as a
service registry and [RabbitMQ](http://www.rabbitmq.com/) as a message queue.
Ensure that these services are running in order to the example code to work
properly.

##### Draw Routes

The routes supported by the application are drawn in [app/routes.js](https://github.com/bixbyjs-examples/smsuid/blob/master/app/routes.js).

```javascript
exports = module.exports = function(IoC) {

  this.get('/', IoC.create('handlers/compose'));
  this.post('/send', IoC.create('handlers/send'));
  
};

exports['@require'] = [ '$container' ];
```

Here we encounter the use of dependency injection using the IoC container.
`$container` is a special component that refers to the IoC container itself, which
can be used to create other components.

In this case, all request handlers are implemented as components.  This eliminates
boilerplate code as the handlers themselves can take advantage of dependency
injection.  Let's take a look at [app/handlers/send.js](https://github.com/bixbyjs-examples/smsuid/blob/master/app/handlers/send.js)
to see how this works.

```javascript
var bodyParser = require('body-parser')
  , errorHandler = require('errorhandler');


exports = module.exports = function(mq, logger) {
  
  function enqueue(req, res, next) {
    var sms = {
      to: req.body.to,
      message: req.body.message
    }
    
    logger.debug('Enqueueing SMS message');
    mq.enqueue('message/sms', sms, function(err) {
      if (err) { return next(err); }
      next();
    });
  }
  
  function respond(req, res, next) {
    res.redirect('/');
  }

  return [ bodyParser.urlencoded(),
           enqueue,
           respond,
           errorHandler() ];
  
}

exports['@require'] = [ 'mq/adapter', 'logger' ];
```

This module exports a _factory function_ which returns a function chain used to
handle requests.  The processing here is straightforward for the sake of a
simple example.

Notice that the `enqueue(req, res, next)` function logs a message for debugging
purposes and enqueues a message on the message queue.  Both the queue and the
logger are injected by virtue of the `@require` annotation.

```javascript
exports['@require'] = [ 'mq/adapter', 'logger' ];
```

When `IoC.create('handlers/send')` is invoked, the IoC container will
automatically instantiate the MQ adapter and logger and inject them into the
exported factory function as `mq` and `logger` parameters, respectively.  The
factory function then returns the function chain:

```javascript
return [ bodyParser.urlencoded(),
         enqueue,
         respond,
         errorHandler() ];
```

which is mounted as the `/send` route in the Express application.

```javascript
this.post('/send', IoC.create('handlers/send'));
```

##### Start Server

Let's start the application.

```bash
$ npm start
info: Connecting to service registry 127.0.0.1:4001
debug: Connected to service registry 127.0.0.1:4001
info: Connecting to message queue 10.200.0.215:5672
debug: Connected to message queue 10.200.0.215:5672
info: Declaring queue message/sms
info: HTTP server listening on 0.0.0.0:8080
info: Announcing service http://schemas.example.com/ui/sms in sms.example.local. at http://10.200.1.85:8080/
```

Navigate to [http://127.0.0.1:8080/](http://127.0.0.1:8080/) and send a
message.  A message will be queued for transmission, but it won't yet be sent as
no worker process is subscribed to the queue.  Let's leave `smsuid` running,
open a new terminal, and proceed with `smsd`.
