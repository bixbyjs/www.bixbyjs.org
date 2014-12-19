---
layout: 'guide'
title: 'Interface-based Programming'
---

### Interface-based Programming

Individual services that comprise an application typically share common
infrastructure, such as a service registry or logging facility.  This
infrastructure can be implemented in a variety of ways, each making different
tradeoffs depending on application requirements.

For example, service discovery can be accomplished using a centralized registry
such as [etcd](https://github.com/coreos/etcd) or [ZooKeeper](http://zookeeper.apache.org/)
or a peer-to-peer protocol like [mDNS](http://en.wikipedia.org/wiki/Multicast_DNS).
While the implementation of each are very different, the functionality provided
is largely the same.

Respecting the time-tested advice to ["program to an interface, not an implementation"](http://en.wikipedia.org/wiki/Design_Patterns),
Bixby.js defines components in terms of interfaces, and uses [dependency injection](http://martinfowler.com/articles/injection.html)
to allow the implementation to vary according to run-time configuration.
This technique increases the modularity and maintainability of an application.
Components implementing support for common infrastructure can be reused
across services, while still respecting application-specific settings.

Bixby.js provides implementations of components that are typically used in a
microservices architecture, including service discovery, message queuing, and
security.  The implementations in use are determined via configuration settings,
allowing development teams to tune an application according to their own
requirements.  For example, a service registry can be configured to use
either etcd or ZooKeeper.  Furthermore, as long as the interface is respected,
developers are free to override any component with a custom implementation.
