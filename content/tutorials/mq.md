---
layout: 'tutorial'
title: 'Message Queues'
---

This tutorial illustrates how to use a [message queue](http://en.wikipedia.org/wiki/Message_queue)
for inter-service communication.  Message queues are often used to distribute
tasks to background worker process, such as sending email or syncing profile
data from a social network.

We'll be implementing two processes during the course of this tutorial.  The
first, [`smsuid`](https://github.com/bixbyjs-examples/smsuid), is a [daemon](http://en.wikipedia.org/wiki/Daemon_%28computing%29)
that provides a web-based interface that allows users to send text messages.
The second, [`smsd`](https://github.com/bixbyjs-examples/smsd) is a worker
process that actually transmits the text messages using a SMS gateway.
