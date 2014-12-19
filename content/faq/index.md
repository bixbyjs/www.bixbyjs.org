---
layout: 'guide'
title: 'FAQ'
---

#### How does Bixby.js deal with services written in other languages?

Microservices rely on inter-process communication to communicate between
services.  Typicially this communication occurs via lightweight mechanisms, such
as RESTful APIs.

All network protocols used by Bixby.js are either industry standard or have open
specifications.  By adopting these standards, Bixby.js can communicate with
services written in other languages and vice versa.


#### Can I use Bixby.js without adopting a micoservice architecture?

Yes.  Bixby.js provides a suite of common components that are generally useful
to any application, regardless of architecture.  However, the majority of
components provided by Bixby.js deal with the requirements of implementing
microservices, and that is the primary focus of development.
