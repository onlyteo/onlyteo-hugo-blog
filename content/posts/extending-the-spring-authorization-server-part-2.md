+++
title = "Extending the Spring Authorization Server - Part 2"
description = "Getting the most out of the Spring Authorization Server by extending it with production ready features."
image = "/images/posts/2024/spring-logo.svg"
date = "2025-01-05T08:12:42+01:00"
draft = true
meta = true
toc = true
featured = true
unlisted = false
authors = [
    "onlyteo"
]
tags = [
    "OAuth2",
    "OpenID Connect",
    "Spring Framework",
    "Spring Boot",
    "Spring Security"
]
categories = [
    "Security",
    "Programming"
]
+++

## Intro

The [Spring Authorization Server](https://spring.io/projects/spring-authorization-server) is a JVM and Spring based
implementation of the [OAuth 2.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-07) and
[OpenID Connect 1.0](https://openid.net/specs/openid-connect-core-1_0.html) specifications.

Using this framework we will build an OpenID Connect Identity Provider (IdP) and OAuth2 Authorization Server. Starting
out with the minimal setup we will extend it with
[Security Best Current Practice](https://oauth.net/2/oauth-best-practice) and production ready features.

These features include:

* Storing OAuth2 registered clients in a database
* Storing OAuth2 authorized clients in a database
* Storing OAuth2 consents in a database
* Storing identity details in a database
* Using external [RSA](https://datatracker.ietf.org/doc/html/rfc8017) keys for OAuth2 token signing
* Requiring [Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636)

This is the first of two posts. In this post we will set up a minimal skeleton of the needed applications. The 
second part will describe how to extend the applications with the features described above. Stay tuned for Part 2!

## Prerequisites

* Java Runtime - e.g. [Temurin JDK](https://adoptium.net) or [OpenJDK](https://openjdk.org)
* [Docker](https://www.docker.com)

We will implement the project using the [Kotlin](https://kotlinlang.org) programming language, build the application
using the Spring Boot framework, and use the [Gradle](https://gradle.org) build and dependency management tool.

