---
layout: 'guide'
title: 'Introduction'
---

### Introduction

Bixby.js provides both an approach to and a set of components for developing
maintainable, scalable, and secure applications using [Node.js](http://nodejs.org/).
This guide outlines the philosophies and rationale behind the approach taken by
Bixby.js and serves as a reference to the components used to implement that
approach.

Users expect modern applications to be accessible from anywhere, be it an office
desktop, a tablet at home, a phone when on-the-go, or a web browser at a public
terminal.  The challenges developers face in meeting these expectations are
numerous.  User interfaces need to be developed for web and mobile platforms.
Data needs to be available via an API for third-party integrations.  Code must
move from development to deployment in a matter of hours.  Security must be
ensured to protect sensitive information.

To meet these challenges, an effective software architecture needs to facilitate
rapid development of high-quality, testable software.  This is no easy task, but
it is one that Bixby.js aims to accomplish.  It does so by adopting a
[microservices](http://en.wikipedia.org/wiki/Microservices) architecture along
with [interface-based programming](http://en.wikipedia.org/wiki/Interface-based_programming).

As you use Bixby.js, you'll be following the hard-learned lessons distilled by
[The Twelve Factors](http://12factor.net/), resulting in applications that are
easier to develop and operate, whether you are part of a small team or a large
organization.
