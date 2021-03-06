---
layout: post
title: Building an OAuth 2 resource server with Spring Security 5
date: 2019-07-06
tags: [spring framework, spring boot, spring security, java, gradle, jwt, oauth]
---

This guide demonstrates how to secure HTTP/S service endpoints using JWT tokens within a Spring Boot microservice.

We will continue the discussion we started in the previous [guide][spring-security-oauth2-authorization-server], which presented a Spring Boot implementation of an authorization server.
Our authorization server allows authenticated clients to obtain digitally signed JWT access tokens, which can be independently verified by a resource server.

The code for this guide is available in [GitHub][spring-security-oauth2-resource-server.git].

### Create a new project
Create a new Spring Boot project using Gradle.

```bash
~$ mkdir spring-security-oauth2-resource-server && cd spring-security-oauth2-resource-server
~$ gradle init --project-name spring-security-oauth2-resource-server --type java-application --test-framework junit --package spring.security.oauth2.resource.server --dsl groovy
```

Replace the contents of the `build.gradle` file with the following:

```groovy
plugins {
  id 'java'
  id 'idea'
  id 'org.springframework.boot' version '2.1.4.RELEASE'
  id 'io.spring.dependency-management' version '1.0.7.RELEASE'
}

repositories {
  jcenter()
}

dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-security'
  implementation 'org.springframework.security.oauth.boot:spring-security-oauth2-autoconfigure:2.1.6.RELEASE'
  implementation 'org.glassfish.jaxb:jaxb-runtime'

  testImplementation('org.springframework.boot:spring-boot-starter-test') {
      exclude group: 'junit', module: 'junit'
  }
  testImplementation 'org.springframework.security:spring-security-test'
  testImplementation 'org.hamcrest:hamcrest:2.1'
  testImplementation 'org.junit.jupiter:junit-jupiter:5.4.2'
  testImplementation 'org.junit.platform:junit-platform-launcher:1.4.1'
}

sourceCompatibility = 11.0
targetCompatibility = 11.0

test {
  useJUnitPlatform()
}
```

### Spring Boot Application
Let's convert the vanila Java application auto-generated by the `gradle` script into a Spring Boot web application.
Replace the contents of the `App.java` class file with the following:

```java
package spring.security.oauth2.resource.server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;

@SpringBootApplication
@EnableResourceServer
public class App {

    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

While we are at it, also remove the `src/test/java/spring/security/oauth2/resource/server/AppTest.java`. This auto-generated test class uses JUnit 4, while we will be using JUnit 5 in this guide.

### Configure Spring Boot application as a Resource Server
Annotating your application with the `@EnableResourceServer` annotation is all the code that is required to enable this feature.
Additional configuration can be controlled via application properties.

In the previous [guide][spring-security-oauth2-authorization-server] we implemented an authorization server issuing JWT access tokens signed using `RS256` algorithm.
We can configure our resource server to use its public key to verify access token digital signatures.

We have two options of how we can configure our resource server.

#### Option 1: Set authorization server public key as application property

In the previous post we geneated the private/public key pair using the following OpenSSL commands:

```bash
~$ openssl genpkey -algorithm RSA -out signing-key.pem -pkeyopt rsa_keygen_bits:2048
~$ openssl rsa -pubout -in private_key.pem -out verifier-key.pem
```

We can simply copy the contents of the `verifier-key.pem` file into the `src/main/resources/application.yaml`:

```yaml
security:
  oauth2:
    resource:
      jwt:
        key-value: |
          -----BEGIN PUBLIC KEY-----
          MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA1vHuOHqm8nI8A1EulAyA
          BIr1JyIxy2eGKVpbDD8PX2Az0MHUauIGc1fKuXtpy/QxAun6DEP1Rcf91i4AhnWX
          aaGqKmVvepHc6g8DWGaS+2xwiyh2iidSLL2jdJPjFe6w7sH/E7s/TfA+vuT2HQqU
          gGpqoJI1Bhinmf0PPiHJ8VeD0rn1DK7dkc99m6YyEiHWXMHbXPUt4M+XDvx+GGyv
          VJy2RNIcUBf3rUfn64fFWKPxoC64dVmC+9KTXY1JiCwKCqns2LQg0aAQUqdb8Q6P
          DzqN69TbTloSfgj3CGHhAfK7tuJbSF8xVBu3H3dhS3HC1y0dZVMMr/6aPNdwUOo/
          2wIDAQAB
          -----END PUBLIC KEY-----
```

#### Option 2: Call authorization server API to dynamically load public key

Manually copying the public key into your application is not ideal. When the keys are rotated, which is a standard practice, you will have to update all resource servers that rely on it.
Instead, our authorization server exposes an endpoint which allows any resource server to dynamically request the latest public key from it.
To configure our resource server to use this endpoint, change the above configuration to this:

```yaml
security:
  oauth2:
    resource:
      jwt:
        key-uri: http://localhost:8080/oauth/token_key
```

Please do not confuse the above property with `security.oauth2.resource.jwk.key-set-uri`. While they do appear to perform the same function, there is a difference.
The JWK key set URI assumes an OpenID Connect (OIDC) compliant endpoint, which the `/oauth/token_key` endpoint is not.

Finally, to ensure that you can run both the authorization and the resource servers on the same machine, you will need to tweak the port number which one of them uses.
For example, modify the default port of the resource server by adding the following entry to the `application.yaml` file: `server.port: 8081`.

### Add a protected resource
Let's expose a simple endpoint to validate that the above configuration is correct and the authentication rules are enforced.

The following REST controller defines a single HTTP GET endpoint accessible via the `/users/me` path.
It will read the JWT claims and return the value of the `user_name` property back to the caller.

Any attempt to call this endpoint without or with invalid HTTP `Authorization` header attribute should result in HTTP `401 Unauthorized` reply.

A call with a valid HTTP `Authorization` header attribute (i.e. containing a valid JWT bearer token), should result in HTTP `200 OK` reply.

```java
package spring.security.oauth2.resource.server;

import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.security.Principal;

@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/me")
    public String me(@AuthenticationPrincipal Principal principal) {
        return principal.getName();
    }
}
```

### Running the application

Start your Spring Boot application via this gradle command:

```bash
~$ ./gradlew bootRun
```

Once the application starts, we can test the `/` endpoint using the `curl` command, e.g.

```bash
~$ curl -H "Authorization: Bearer JWT_GOES_HERE" http://localhost:8081/users/me --verbose
```

### Conclusion
This guide demonstrated how to protect HTTP service endpoints within a Spring Boot web application using JWT authentication tokens.

The complete, working solution is available in [GitHub][spring-security-oauth2-resource-server.git].

[spring-security-oauth2-authorization-server]: /2019/06/28/spring-security-oauth2-authorization-server
[spring-security-oauth2-resource-server.git]: https://github.com/academyhq/spring-security-oauth2-resource-server

