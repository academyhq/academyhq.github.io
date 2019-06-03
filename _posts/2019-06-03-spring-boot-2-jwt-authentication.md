---
layout: post
title: "Securing Spring Boot services using JWT"
date: 2019-06-03
---

### What do we want to do?
Let's say we are building a microservice using Spring Boot and we want to protect our HTTP service endpoints using a JWT token (how the client can obtain a JWT token is a subject for another post).

### How can we do this?
We will need to add `spring-security-oauth2` and `spring-security-jwt` dependencies to our Spring Boot project.
Using these libraries we can configure our micrservice as a "Resource Server" capable of authenticating and authorising access to its HTTP endpoints by validating incoming JWT bearer tokens.

```groovy
plugins {
  id 'java'
  id 'idea'
  id 'org.springframework.boot' version '2.1.4.RELEASE'
  id 'io.spring.dependency-management' version '1.0.7.RELEASE'
}

repositories {
  mavenCentral()
}

dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-security'
  implementation 'org.springframework.security.oauth:spring-security-oauth2:2.3.5.RELEASE'
  implementation 'org.springframework.security:spring-security-jwt:1.0.10.RELEASE'
}
```

### Resource Service
We configuration the microservice as a Resource Server by annotating it with the `@EnableResourceServer` and defining a token service.
In the following example, symmetric key algorithm is assumed to have been used to generate a JWT token (e.g. `HS256` or any of its variants).
That is, the same key is used for both creating and verifying the token, as opposed to asymmtric key whereby a key for creating a token is different to the one used to verify it.

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Value("${security.jwt.signing.key}")
    private String signingKey;

    @Bean
    public JwtAccessTokenConverter accessTokenConverter() {
        var converter = new JwtAccessTokenConverter();
        converter.setSigningKey(signingKey);
        return converter;
    }

    @Bean
    public TokenStore tokenStore() {
        return new JwtTokenStore(accessTokenConverter());
    }

    @Bean
    @Primary
    public DefaultTokenServices tokenServices() {
        var defaultTokenServices = new DefaultTokenServices();
        defaultTokenServices.setTokenStore(tokenStore());
        return defaultTokenServices;
    }

    @Override
    public void configure(ResourceServerSecurityConfigurer config) {
        config.tokenServices(tokenServices());
    }
}
```

The signature verification key has been externalised into a standard Spring Boot configuration file.
Add the following `application.yaml` to the `src/main/resources` directory of your project.

```application.yaml
security.jwt.signing.key: R2XDCP83yB23awZkfzK/nErOV/0=
```

### Let's test our service
You can use https://jwt.io/ to quickly generate a JWT token for testing.
Ensure to pick `HS*` (e.g. `HS256`) algorithm from the dropdown and use the same key value as `security.jwt.signing.key` defined in `application.yaml`.

```bash
curl -H "Authorization: Bearer [TOKEN GOES HERE]" http://localhost:8080/[ENDPOINT]
```

