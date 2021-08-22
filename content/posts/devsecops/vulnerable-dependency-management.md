---
title: "Vulnerable Dependency Management"
description: "changeme"
date: 2021-08-22T12:52:20-04:00
lastmod: 2021-08-22T12:52:20-04:00
draft: false
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
tags:
---

Let's talk about something that most developers and engineering teams don't pay
the necessary attention. Do you have a process to update your application
dependencies? How likely is it that your application could be compromised
because of a vulnerable dependency?

> **TL;DR** - In this post, you will see that you shouldn't be omissive in terms
of upgrading your dependencies and you should strongly consider automating
dependency vulnerability check in your pipeline.

## Context

Regardless of the platform you use - Java, Go, Python, PHP, .Net, etc - your
application likely relies on external dependencies, probably dozens and often
hundreds of them. In fact, we can fairly say that most of the application code
for any modern application comes from third-party open source libraries and
frameworks. For example, the [State of Software Security Report v11](https://www.veracode.com/state-of-software-security-report), has found that 97% of a typical Java
application is made up of open source libraries.

{{< notice type="tip" id="overview" title="Dependency Management quick review" >}}
Any modern application uses dependency management tools like Maven, NuGet,
NPM, Pip, etc to automate the process of installing and configuring external
libraries. Developers will add a new library to their application when they need
a feature that has already been implemented. Using third-party components
increases developers' productivity allowing them to focus on the domain while
reusing cross-cutting specialized features. These features can vary from simple
utilities, like string formatting or arrays manipulation, to more complex
features that take care of some core behavior of the application, such as an
HTTP handler, an ORM, or a Dependency Injection framework. It's important to
make this distinction because upgrading a utility library and a core framework
will require different levels of effort and concern to be upgraded.
{{< /notice >}}