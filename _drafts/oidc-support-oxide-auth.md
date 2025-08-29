---
layout: post
title: "An oxide-auth primer"
tags: rust authentication
categories: rust
---

This article makes use of terminology described in the [OAuth2
RFC](https://datatracker.ietf.org/doc/html/rfc6749) and the [OpenID Connect
Standard](https://openid.net/specs/openid-connect-core-1_0.html).

## Introduction

[//]: # TODO: Rewrite the whole introduction to talk about the importance of
[//]: # familiarising oneself with the crate to later on introduce OIDC on
[//]: # top of it.

I'm currently working on a project whose end goal is providing an authentication
service based on the user's provided digital certificate on the browser. This
project, written in Rust, makes use of the `oxide-auth` crate to provide its
OAuth2 service.

However, OAuth2 is an _authorization_ standard, and the whole purpose of the
project I want to develop is _authentication_. As there aren't any protected
resources on my service, it's fairly safe to say OAuth2 isn't the right tool
for the job.

Spending the time to support OAuth2 in my project isn't a mistake, though.
The plan is to implement OpenID Connect, an _authentication_ protocol built on
top of OAuth2. Its purpose is providing information about the resource owner to
the client.

## Getting to know oxide-auth

The crate we're going to explore, `oxide-auth`, helps developers implement
OAuth2 by providing a set of abstractions over the various state machines
involved in an OAuth2 flow. I have heard it described as a "protocol as a
library" and I think it's a pretty accurate description. Because of this, I've
personally had a bit of a hard time familiarising myself with the tools it
provides and how it expects you to do things. That's right, skill issue.


