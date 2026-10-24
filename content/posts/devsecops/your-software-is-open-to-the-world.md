---
title: "Your Software Is Open to the World"
description: "An application's attack surface extends beyond visible features and documented endpoints. Publicly exposed information can also reveal additional infrastructure and services."
date: 2026-10-21
lastmod: 2026-10-21
draft: false
categories:
  - "The Security Mindset"
tags:
  - "application security"
  - "security mindset"
---

When it comes to security, there are a lot of things I'd love to have known when I started my career as a software engineer. Now, as an engineer focused on cybersecurity, there are things I'd like my product engineering colleagues to know.

There are books, videos, and other resources out there that can teach you about vulnerabilities, what they are, and how they can be exploited. Others teach tools and methods for scanning code, finding vulnerabilities, and fixing them. Some of these resources, usually the good ones, bring more than just technical expertise. They help you develop a security mindset.

This post is the first in a series where I want to share, from my experience, some habits and perspectives that can help software engineers develop a strong security mindset.

<!--more-->

## The attack surface goes beyond the design

When we build software, we naturally think about the functionality we intend to deliver: the pages users navigate, the requests our frontend makes, and the APIs and backend services. While we tend to focus on that intended functionality, once the software is on the internet, people (and nowadays automated agents) can interact with it without following the workflows we designed. Some will deliberately look for things we never intended to make available, things we thought were "hidden enough," and others we had no idea could be reachable.

To understand the attack surface, we need to look beyond the design and consider which endpoints and URL paths respond, what HTTP methods they accept, and how request parameters, cookies, headers, and other inputs can affect their behavior.

For example, the backend might accept an attribute that isn't exposed anywhere in the UI, or process a parameter that the frontend never sends but could be inferred by someone inspecting the application. The same goes for paths that aren't linked from any page and redirect flows that may reveal otherwise unlinked endpoints or expose sensitive values in URLs. Someone using an HTTP client, an intercepting proxy, a script, or an AI agent can interact with those interfaces directly, without going through the application the way we expect a user to.

Of course, being reachable doesn't mean something is necessarily vulnerable. It does mean that someone can inspect it, send unexpected requests, validate and stress edge cases, and look for weaknesses. The controls that protect an application have to work independently of what the UI or API documentation makes visible or what we assume people will discover.

A motivated actor can sometimes understand an application's exposed behavior better than the engineers maintaining it, especially when it comes to legacy systems.

## Common reconnaissance techniques

What many engineers overlook is that when we expose an endpoint on the internet, the information an application reveals can help someone discover infrastructure and services beyond the application itself. By examining its technologies, configuration details, and infrastructure characteristics—what we often call its fingerprint—someone may find clues that lead to other systems associated with the organization. Even the content published on a website can reveal useful information, including details that could support social engineering.

Some common techniques for mapping an attack surface include:

- **Subdomain enumeration:** Use DNS information and [Certificate Transparency logs](https://www.rfc-editor.org/rfc/rfc9162.html) to discover hostnames associated with a domain.
- **Network ownership research:** Investigate autonomous system numbers (ASNs), announced IP address ranges, and WHOIS/RDAP records.
- **Application inspection:** Examine URL paths, HTML, JavaScript, HTTP headers, cookies, and network requests for exposed information and additional endpoints.
- **Source code searches:** Look through public GitHub repositories for credentials, configuration details, endpoints, or infrastructure references.
- **Search engines and archives:** Use advanced search operators to find indexed resources and the [Wayback Machine](https://web.archive.org/) to inspect historical content.
- **Cloud resource discovery:** Look for publicly accessible storage buckets and other exposed cloud resources.
- **Public container images:** Search public registries and inspect images for credentials, infrastructure details, and additional context.
- **Published software and supply-chain research:** Examine publicly distributed libraries, SDKs, packages, and other software artifacts for configuration details, dependencies, endpoints, and references to supporting services.
- **Internet-wide service searches:** Use [Shodan](https://www.shodan.io/) or [Censys](https://search.censys.io/) to identify exposed hosts and network services.
- **Network scanning:** Scan identified IP addresses and services with tools such as [Nessus](https://www.tenable.com/products/nessus) when authorized to do so.

This is far from a complete list of reconnaissance techniques. There are other, less obvious approaches, such as investigating company acquisitions, identifying relationships between organizations through shared infrastructure or analytics identifiers, and correlating systems using things like favicons. These techniques can reveal connections and resources that aren't immediately apparent when investigating an application.

Finding a hostname or an IP address doesn't necessarily mean it belongs to the company being investigated. Public records can be outdated, and cloud or network infrastructure is often shared, so those associations need to be verified. Even with these limitations, the amount of information available through an application's public footprint can be surprising.

As software engineers, we should look beyond the features we intentionally expose and consider what someone else can actually reach or learn from our systems. Undocumented paths, backend attributes, request metadata, and information disclosed through other public resources can all give an attacker more places to look for weaknesses.

> **AI assistance acknowledgment:** This post was written with AI support for paragraph construction, organization, and reviewing sentence clarity and arguments. The ideas, experiences described, perspective, and editorial decisions remain with the author.
