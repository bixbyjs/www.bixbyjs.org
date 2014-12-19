---
layout: 'guide'
title: 'Microservices'
---

### Microservices

A [microservices](http://en.wikipedia.org/wiki/Microservices) architecture is
one which constructs an application as a suite of small, independedent services
that communicate with each other via a network.  As [noted](http://martinfowler.com/articles/microservices.html)
by Martin Fowler, there is no precise definition of this architecture, but
rather a set of common characteristics that offer guidelines about how to
architect an application.

Microservices are organized around capabilities, such as authorization, billing,
or recommendations, each running in its own process and exposing a protocol or
API that can be consumed by other services.  Centralized management is kept to a
minimum, and each service can be developed by a separate team, written in a
different language, and deployed and scaled independently.

As with any architectural decision, adopting microservices comes with tradeoffs.
Microservices rely on inter-service communication, and thus shift some
complexity into the network.  To address this, Bixby.js provides components that
assist with new requirements introduced by a microservices architecture,
including service discovery, message queuing, and security.

In providing these components, Bixby.js acknowledges that it is the network and
protocols that are most important - not the programming language.  Bixby.js is
implemented in JavaScript, but all protocols are accompanied by specifications.
Because of this, services developed in other programming languages can
communicate with those developed with Bixby.js and vice versa.
