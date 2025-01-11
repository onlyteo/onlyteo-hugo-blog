+++
title = "Extending the Spring Authorization Server - Part 1"
description = "Getting the most out of the Spring Authorization Server by extending it with production ready features."
image = "/images/posts/2024/spring-logo.svg"
date = "2024-11-26T22:13:03+01:00"
draft = false
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

Using this framework we will build an OAuth2 Authorization Server and OpenID Connect Identity Provider (IdP). Starting
out with the minimal setup we will extend it with
[Security Best Current Practice](https://oauth.net/2/oauth-best-practice) and production ready features.

These features include:

* Requiring [Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636)
* Storing user credentials in a database
* Storing OAuth2 registered clients in a database
* Storing OAuth2 authorizations in a database
* Storing OAuth2 authorization consents in a database
* Using external [RSA](https://datatracker.ietf.org/doc/html/rfc8017) keys for OAuth2 token signing

This is the first of two posts. In this post we will set up a minimal skeleton of the needed applications. The 
second part we describe how to extend the Authorization Server with the features described above.

### Parts
* Extending the Spring Authorization Server - Part 1 (this post)
* Extending the Spring Authorization Server - Part 2 (coming soon!)

## Prerequisites

* Java Runtime - e.g. [Temurin JDK](https://adoptium.net) or [OpenJDK](https://openjdk.org)
* [Docker](https://www.docker.com)

We will implement the project using the [Kotlin](https://kotlinlang.org) programming language, configure the application
using the Spring Boot framework, and use the [Gradle](https://gradle.org) build and dependency management tool.

## Create a new Webapp to secure
First we create a simple Spring MVC Webapp which we later will secure using the Spring Authorization Server. The
Webapp will use the Thymeleaf templating engine to render webpages.

### Create webapp using the Spring Initializr website
The easies way to bootstrap a new Spring project is by using Spring's own Initializr tool. Spring offers several
different ways to use the Initializr tool. They have a dedicated CLI, as well as plugins for popular IDEs like
IntelliJ and Eclipse. But we will use the Spring Initializr website [start.spring.io](https://start.spring.io).

Go to the Spring Initializr website and select the needed options.

Important selections include:
* **Project type**: Gradle - Kotlin
* **Language**: Kotlin
* **Spring Boot version**: latest stable (currently 3.4.0)
* **Packaging**: Jar
* **Java version**: 21
* **Dependencies**:
  * **Spring Web**
  * **Thymeleaf**

![spring-initializr-website-oauth2-webapp](/images/posts/2024/spring-authorization-server/spring-initializr-website-oauth2-webapp.png)

Click the `[GENERATE]` button to generate and download the project.

### Extract and clean up the project
Once downloaded we have a `.zip` file containing a template project. Unzip the archive into a suitable location and
open it in your favorite editor or IDE.

Delete unnecessary files until your project look something like this:
```diff
▼ spring-oauth2-webapp
   ⯈ gradle
   ▼ src
      ▼ main
         ▼ kotlin
            ▼ com
               ▼ onlyteo
                  ▼ sandbox
                     SpringOauth2WebappApplication.kt
         ▼ resources
            application.properties
   .gitignore
   build.gradle.kts
   gradlew
   gradlew.bat
   settings.gradle.kts
```

For better readability we will swap the configuration file `application.properties` from a properties file to a yaml
file `application.yaml`. The contents should be modified into YAML format:
```yaml
### SPRING ###
spring:
  # Application
  application:
    name: spring-oauth2-webapp

### LOGGING ###
logging:
  level:
    root: INFO
    com.onlyteo: DEBUG
```
There has also been added properties for logging.

The added comments are to easily separate different sections of the YAML file.

### Add a Thymeleaf home page
First we add a Spring Controller bean `ViewController` to the `/src/main/kotlin/` path under the
`com.onlyteo.sandbox.controller` package with the following contents:
```kotlin
package com.onlyteo.sandbox.controller

import org.springframework.stereotype.Controller
import org.springframework.web.bind.annotation.GetMapping

@Controller
class ViewController {

    @GetMapping(path = ["/"])
    fun getIndexPage(): String {
        return "index"
    }
}
```

Next we add a simple stylesheet `style.css` to the `/src/main/resources/static/css/` path with the following contents:
```css
body {
    margin: 0;
    padding: 0;
    font-family: system-ui, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
}

main {
    margin: 0;
    padding: 20px 40px;
}
```

Lastly we add a Thymeleaf enabled home page `index.html` to the `/src/main/resources/templates/` path with the 
contents:
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Spring OAuth2 Webapp</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
    <link rel="stylesheet" type="text/css" th:href="@{~/css/style.css}"/>
</head>
<body>
<main>
    <h2>Spring OAuth2 Webapp</h2>
</main>
</body>
</html>
```

With these changes the project should look like this:
```diff
▼ spring-oauth2-webapp
   ⯈ gradle
   ▼ src
      ▼ main
         ▼ kotlin
            ▼ com
               ▼ onlyteo
                  ▼ sandbox
                     ▼ controller
                        ViewController.kt
                     SpringOauth2WebappApplication.kt
         ▼ resources
            ▼ static
               ▼ css
                  style.css
            ▼ templates
               index.html
            application.yaml
   .gitignore
   build.gradle.kts
   gradlew
   gradlew.bat
   settings.gradle.kts
```

### Build and run project
In a terminal go into the project folder and build it using Gradle:
```shell
./gradlew build
```

The build process should finish with the output `BUILD SUCCESSFUL`.

Now lets start the Spring Boot application using Gradle:
```shell
./gradlew bootRun
```

Once you see the application logging the following message open [http://localhost:8080](http://localhost:8080) in a
browser.
```shell
Started SpringOauth2WebappApplicationKt in X seconds
```

You should get the home page.

![initial-home-page](/images/posts/2024/spring-authorization-server/initial-home-page.png)

## Create an Authorization Server

### Using the Spring Initializr website
We will use the same method for creating the Authorization Server project as we did for the Webapp.

Go to the Spring Initializr website and select the needed options.

Important selections include:
* **Project type**: Gradle - Kotlin
* **Language**: Kotlin
* **Spring Boot version**: latest stable (currently 3.4.0)
* **Packaging**: Jar
* **Java version**: 21
* **Dependencies**:
  * **Spring Web**
  * **OAuth2 Authorization Server**

![spring-initializr-website-authorization-server](/images/posts/2024/spring-authorization-server/spring-initializr-website-authorization-server.png)

Click the `[GENERATE]` button to generate and download the project.

### Extract and clean up the project
Once downloaded we have a `.zip` file containing a template project. Unzip the archive into a suitable location and 
open it in your favorite editor or IDE.

Delete unnecessary files until your project look something like this:
```diff
▼ extended-spring-authorization-server
   ⯈ gradle
   ▼ src
      ▼ main
         ▼ kotlin
            ▼ com
               ▼ onlyteo
                  ▼ sandbox
                     ExtendedSpringAuthorizationServerApplication.kt
         ▼ resources
            application.properties
   .gitignore
   build.gradle.kts
   gradlew
   gradlew.bat
   settings.gradle.kts
```

Here as well we will swap the configuration file `application.properties` from a properties file to a yaml 
file `application.yaml`. The contents should be modified into YAML format:
```yaml
### SPRING ###
spring:
  # Application
  application:
    name: extended-spring-authorization-server

### LOGGING ###
logging:
  level:
    root: INFO
    com.onlyteo: DEBUG

### SERVER ###
server:
  port: 8888
```
We have changed the port from the default in order to avoid port collision with the Webapp during local development.

### Build and run project
In a terminal go into the project folder and build it using Gradle:
```shell
./gradlew build
```

The build process should finish with the output `BUILD SUCCESSFUL`.

Now lets start the Spring Boot application using Gradle:
```shell
./gradlew bootRun
```

Once you see the application logging the following message open [http://localhost:8888](http://localhost:8888) in a 
browser.
```shell
Started ExtendedSpringAuthorizationServerApplicationKt in X seconds
```

You should get a login webpage.

![login-webpage](/images/posts/2024/spring-authorization-server/login-webpage.png)

The application is not an Authorization Server yet, due to missing configuration. But since the application has Spring 
Security on the classpath Spring Boot has automatically secured it with a form login.

## Configure a minimal OAuth2 setup
Now that we have our skeleton versions of the Webapp and Authorization Server applications ready we can start to update 
them with OAuth2 configuration. We will turn the Webapp into an OAuth2 Client and the Authorization Server into an 
OAuth2 Authorization Server and an OpenID Connect Identity Provider.

The Webapp will be secured using the 
[Authorization Code](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow)
grant type, which is the recommended login flow for UI based web applications.

### Webapp configuration
The easiest way to configure Spring Security is by using the properties based configuration support. Since this is a 
Spring Boot enabled application it will automatically instantiate the necessary Spring beans in order bootstrap Spring 
Security features.

First we need to add the required Spring Security dependencies to the Webapp project. Open the `build.gradle.kts` 
file and add the last two dependencies:
```kotlin
dependencies {
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-thymeleaf")
    // Security
    implementation("org.thymeleaf.extras:thymeleaf-extras-springsecurity6")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
}
```

After that we can add OAuth2 Client related properties to the `application.yaml` file:
```yaml
### SPRING ###
spring:
  # Application
  application:
    name: spring-oauth2-webapp
  # Security
  security:
    oauth2:
      client:
        registration:
          spring-oauth2-webapp:
            provider: spring-authorization-server
            client-id: spring-oauth2-webapp
            client-secret: G4nd4lf
            scope:
              - openid
              - profile
              - roles
            authorization-grant-type: authorization_code
            client-authentication-method: client_secret_basic
        provider:
          spring-authorization-server:
            issuer-uri: http://127.0.0.1:8888 # Use IP to avoid session cookie collision

### LOGGING ###
logging:
  level:
    root: INFO
    com.onlyteo: DEBUG

### SERVER ###
server:
  port: 8080
```
The most important properties are under the `spring.security.oauth2.client` prefix. There is an entry for an OAuth2
Client `spring-oauth2-webapp`. This is known in Spring Security terms as a *Registered Client*.

Lastly we will add a tag to the `ìndex.html` home page to show the name of the logged-in user:
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity6">
<head>
    <title>Spring OAuth2 Webapp</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
    <link rel="stylesheet" type="text/css" th:href="@{~/css/style.css}"/>
</head>
<body>
<main>
    <h2>Spring OAuth2 Webapp</h2>
    <p>Logged in as <b sec:authentication="name"></b></p>
</main>
</body>
</html>
```

We have added a `sec` XML namespace that is used to show the username using the `sec:authentication="name"` parameter.

That is all the necessary changes required to secure the Webapp.

### Authorization Server configuration
Also with the Spring Authorization Server we will use the properties based configuration. Below is the required
contents of the `application.yaml` file.

```yaml
### SPRING ###
spring:
  # Application
  application:
    name: extended-spring-authorization-server
  # Security
  security:
    user:
      name: bilbo
      password: "{noop}G4nd4lf"
      roles:
        - USER
    oauth2:
      authorizationserver:
        client:
          spring-oauth2-webapp:
            registration:
              client-id: spring-oauth2-webapp
              client-secret: "{noop}G4nd4lf"
              scopes:
                - openid
                - profile
                - roles
              authorization-grant-types:
                - authorization_code
                - refresh_token
              client-authentication-methods:
                - client_secret_basic
              redirect-uris:
                - http://localhost:8080/login/oauth2/code/spring-oauth2-webapp

### LOGGING ###
logging:
  level:
    root: INFO
    com.onlyteo: DEBUG

### SERVER ###
server:
  port: 8888
```

The most important properties are under the `spring.security.oauth2.authorizationserver` prefix. There is an entry for 
one OAuth2 Client `spring-oauth2-webapp`. This mirrors the configuration in the Webapp.

A user has been defined under the `spring.security.user` prefix. These credentials are used for logging in and gain 
access to OAuth2 protected webapps. The `{noop}` prefix in the password signifies that it is specified in 
plaintext, which means it is supplied in an unencrypted format.

### Test OAuth2 login flow
The applications are now fully configured and we are ready to test the Authorization Code login flow.

Start both applications and open the Webapp home page [http://localhost:8080](http://localhost:8080) in a
browser. We see that we are quickly redirected to the Authorization Server address for login
[http://127.0.0.1:8888/login](http://127.0.0.1:8888/login).

![login-webpage](/images/posts/2024/spring-authorization-server/login-webpage.png)

We supply the user credentials in the login form, username: **bilbo**, password: **G4nd4lf**. When we then click the
`[Sign in]` button we are logged in and redirected back to the Webapp home page.

![updated-home-page](/images/posts/2024/spring-authorization-server/updated-home-page.png)

We see that the home page displays the username of the logged-in user.

And that is it! We have a working OAuth2 Client and OAuth2 Authorization Server. In the next part of this series we 
will extend the Authorization Server with the features listed in the intro. Happy coding!
