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

* Requiring [Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636)
* Storing user credentials in a database
* Storing OAuth2 registered clients in a database
* Storing OAuth2 authorizations in a database
* Storing OAuth2 authorization consents in a database
* Using external [RSA](https://datatracker.ietf.org/doc/html/rfc8017) keys for OAuth2 token signing

This is the second of two posts. In the first part we create two skeleton applications. In this post we will extend
upon what we did in part 1 and update the Authorization Server with the features described above. 

### Parts
* [Extending the Spring Authorization Server - Part 1](/posts/extending-the-spring-authorization-server-part-1)
* Extending the Spring Authorization Server - Part 2 (this post)

## Prerequisites

* Java Runtime - e.g. [Temurin JDK](https://adoptium.net) or [OpenJDK](https://openjdk.org)
* [Docker](https://www.docker.com)

We will implement the project using the [Kotlin](https://kotlinlang.org) programming language, configure the application
using the Spring Boot framework, and use the [Gradle](https://gradle.org) build and dependency management tool.

## Setting up database
We need a persistent storage for many of the features. For that we will use a Docker powered PostgreSQL database.

### Set up a PostgreSQL database with Docker
In the root of the Authorization Server project create a `docker-compose.yaml` file with the contents:
```yaml
### SERVICES ###
services:
  postgres:
    image: postgres
    container_name: postgres
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: G4nd4lf
    ports:
      - "5432:5432"
    volumes:
      - ./src/main/resources/db/init:/docker-entrypoint-initdb.d
      - postgres.data:/var/lib/postgresql/data
    networks:
      - postgres

### VOLUMES ###
volumes:
  postgres.data:
    name: postgres.data

### NETWORKS ###
networks:
  postgres:
    name: postgres
```

Add `authorization_server.sql` database initialization script to the `/src/main/resources/db/init` path with the
contents:
```postgresql
CREATE USER authorization_server WITH PASSWORD 'G4nd4lf';
CREATE DATABASE authorization_server WITH OWNER authorization_server;
```

Start the PostgreSQL Docker container by running the following command from the Authorization Server project root:
```shell
docker compose up -d
```

## Update Webapp with improved features
In order for the Webapp to support the improvements to security features that we will implement in the Authorization 
Server it must also be updated with a few features. Pure property driven configuration is not sufficient to enable 
these features. We must update the default Spring configuration by overriding a few Spring beans and configuration 
classes.

### Enable PKCE and OIDC logout support
The Authorization Server will be updated so that all OAuth2 clients are required to use PKCE together with the 
Authorization Code flow. Therefor PKCE must also be enabled in the Webapp. We must also enable a complete OIDC 
logout flow. This will ensure that logout will clear the session both in the Webapp and in the Authorization Server.

First we add a configuration bean to the `com.onlyteo.sandbox.config` package, that will hold all the necessary
configuration overrides:
```kotlin
@Configuration(proxyBeanMethods = false)
class WebSecurityConfig {
}
```

The project will now look like this:
```diff
▼ spring-oauth2-webapp
   ⯈ gradle
   ▼ src
      ▼ main
         ▼ kotlin
            ▼ com
               ▼ onlyteo
                  ▼ sandbox
                     ▼ config
                        WebSecurityConfig.kt
                     ▼ controller
                        ViewController.kt
                     SpringOauth2WebappApplication.kt
         ⯈ resources
   .gitignore
   build.gradle.kts
   gradlew
   gradlew.bat
   settings.gradle.kts
```

To enable the OIDC logout feature we add the following bean factory method to the `WebSecurityConfig` class:
```kotlin
@Bean
fun logoutSuccessHandler(
    clientRegistrationRepository: ClientRegistrationRepository
): LogoutSuccessHandler {
    return OidcClientInitiatedLogoutSuccessHandler(clientRegistrationRepository)
}
```

To enable the PKCE feature we add the following bean factory method to the `WebSecurityConfig` class:
```kotlin
@Bean
fun oAuth2AuthorizationRequestResolver(
    clientRegistrationRepository: ClientRegistrationRepository
): OAuth2AuthorizationRequestResolver {
    return DefaultOAuth2AuthorizationRequestResolver(
        clientRegistrationRepository,
        DEFAULT_AUTHORIZATION_REQUEST_BASE_URI
    ).apply {
        setAuthorizationRequestCustomizer(OAuth2AuthorizationRequestCustomizers.withPkce()) // Enables PKCE
    }
}
```

This all comes together in a `SecurityFilterChain` like this:
```kotlin
@Bean
@Throws(Exception::class)
fun webSecurityFilterChain(
    http: HttpSecurity,
    oAuth2AuthorizationRequestResolver: OAuth2AuthorizationRequestResolver,
    logoutSuccessHandler: LogoutSuccessHandler
): SecurityFilterChain {
    return http
        .authorizeHttpRequests { config ->
            config
                .requestMatchers("/assets/**", "/error").permitAll()
                .anyRequest().authenticated()
        }
        .oauth2Login { config ->
            config
                .authorizationEndpoint { auth ->
                    auth.authorizationRequestResolver(oAuth2AuthorizationRequestResolver)
                }
        }
        .logout { config ->
            config
                .logoutSuccessHandler(logoutSuccessHandler)
        }
        .build()
}
```

This concludes the necessary changes for the Webapp. Let's move on to the Authorization Server.

## Update Authorization Server with improved features
We are now ready to implement the improved security features in the Authorization Server.

### Enable PKCE for all OAuth2 clients
Enabling PKCE in the Authorization Server is very easy. It is a simple matter of adding the property 
`require-proof-key: true`:
```yaml
### SPRING ###
spring:
  # Security
  security:
    oauth2:
      authorizationserver:
        client:
          sandbox-frontend:
            require-proof-key: true # Enables PKCE
            require-authorization-consent: false
            registration:
              client-id: sandbox-frontend
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
```

### Add persistence dependencies
We need to add necessary dependencies to the Authorization Server `build.gradle.kts` file. We need Spring Boot 
JDBC support, PostgreSQL JDBC driver and Flyway database migration framework.
```kotlin
dependencies {
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-authorization-server")
    // Persistence
    implementation("org.springframework.boot:spring-boot-starter-jdbc")
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
}
```

Then we need the database model that will allow us to persist the necessary data to the database. The SQL 
scripts required to create the database model are supplied by Spring. We will download them from the GitHub project
for the Spring Authorization Server and save them to the Flyway migration folder `/src/main/resources/db/migration`.

We will download the following files:
* [oauth2-authorization-schema.sql](https://github.com/spring-projects/spring-authorization-server/blob/main/oauth2-authorization-server/src/main/resources/org/springframework/security/oauth2/server/authorization/oauth2-authorization-schema.sql)
* [oauth2-authorization-consent-schema.sql](https://github.com/spring-projects/spring-authorization-server/blob/main/oauth2-authorization-server/src/main/resources/org/springframework/security/oauth2/server/authorization/oauth2-authorization-consent-schema.sql)
* [oauth2-registered-client-schema.sql](https://github.com/spring-projects/spring-authorization-server/blob/main/oauth2-authorization-server/src/main/resources/org/springframework/security/oauth2/server/authorization/client/oauth2-registered-client-schema.sql)

If we look at the comment in the `oauth2-authorization-schema.sql` we see that we need to modify the file, replacing 
the use of `blob` with the `text` datatype since we are using PostgreSQL.
