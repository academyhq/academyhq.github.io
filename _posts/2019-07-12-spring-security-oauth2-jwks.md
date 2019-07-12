---
layout: post
title: Adding JWK Set URI endpoint to Spring Security OAuth2 Authorization Server
date: 2019-07-12
tags: [spring framework, spring boot, spring security, java, gradle, jwt, jwk, oauth]
---

In the previous [guide][spring-boot-oauth2-authorization-server.post] we presented a simple implementation of an authorization server built using Spring Boot 2 and Spring Security 5.
However, we've discovered that the default `/oauth/token_key` endpoint exposed by our service did not conform to the OpenID Connect (OIDC) specification.
While this is not a problem for resource servers written in Spring Boot, mainly because of a workaround exposed via the `security.oauth2.resource.jwt.key-uri` property,
applications using other technologies would not be able to easily integrate with our authorization server.

The purpose of this guide is to demonstrate how we can extend our existing implementation by adding an OIDC compliant key set endpoint.
Same technique could then be used to expose additional OIDC endpoints thus bringing our implementation inline with the spec.

The code for this guide is available in [GitHub][spring-boot-oauth2-authorization-server-jwks.git].

### Configure Authorization Server to use JWK
Starting with our original implementation of the [authorization server][spring-boot-oauth2-authorization-server.git],
we need to add a new dependency `com.nimbusds:nimbus-jose-jwt:7.4` to the `build.gradle` script to get the necessary primitives required to implement the JWK endpoint.

Once this dependency is added to the classpath, we can implement a framework endpoint that exposes the authorization server's public key using JWK set URI.
As per specification, let's expose this endpoint via the `/.well-known/jwks.json` path.

```java
package spring.boot.oauth2.authorization.server;

import com.nimbusds.jose.jwk.JWKSet;
import com.nimbusds.jose.jwk.KeyUse;
import com.nimbusds.jose.jwk.RSAKey;
import net.minidev.json.JSONObject;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.oauth2.provider.endpoint.FrameworkEndpoint;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ResponseBody;

import java.security.KeyPair;
import java.security.interfaces.RSAPublicKey;

@FrameworkEndpoint
public class JwkSetEndpoint {

    @Autowired
    private KeyPair keyPair;

    @GetMapping("/.well-known/jwks.json")
    @ResponseBody
    public JSONObject getKey() {
        RSAPublicKey publicKey = (RSAPublicKey) this.keyPair.getPublic();
        RSAKey key = new RSAKey.Builder(publicKey).keyID("1").keyUse(KeyUse.SIGNATURE).build();
        return new JWKSet(key).toJSONObject();
    }
}
```

Notice how we are setting a `kid` property of the exported public key.
Since JWK endpoint supports multiple keys, for example, to facilitate key rotation, it is important to identify each key using a unique `kid` value.
This `kid` is added to the JWT header of the access token, so that the resource server knows which public key to use to verify the token signature.

It is likewise important to add the `KeyUse.SIGNATURE` to the resulting payload.
The resource server will expect the verifier key `use` flat to be set to `sig`, as it's the only purpose that this key should be used for, i.e. signing JWT access tokens!

### Quick Test
We are now ready to see the result of the above changes.
Start the application server again using this Spring Boot gradle command:

```bash
~$ ./gradlew bootRun
```

Once the authorization server starts, we can make a call to the `/.well-known/jwks.json` endpoint we've implemented, e.g.

```bash
~$ curl http://localhost:8080/.well-known/jwks.json
```

The response should look something like this:

```json
{
  "keys":[{
    "kid": "1",
    "use: "sig",
    "kty": "RSA",
    "e": "AQAB",
    "n": "1vHuOHqm8nI8A1EulAyABIr......6aPNdwUOo_2w"
  }]
}
```

### Configure Resource Server to use JWK
Re-configuring the resource server to use the JWK set URI endpoint, instead of the original `/oauth/token_key` endpoint is simple.
Starting with our original implementation of the [resource server][spring-boot-oauth2-resource-server.git],
we need to remove the security.oauth2.resource.jwt.key-uri` property and replace it with `security.oauth2.resource.jwk.key-set-uri`.

```yaml
server.port: 8081
security:
  oauth2:
    resource:
      jwk:
        key-set-uri: http://localhost:8080/.well-known/jwks.json
```

Unfortunately, a quick test of this configuration using the following commands returns an error.

```bash
// get the JWT access token
~$ curl -X POST client:password@localhost:8080/oauth/token -dgrant_type=client_credentials -dscope=any

// use the JWT access token to access a protected resource
~$ curl -H "Authorization: Bearer JWT_GOES_HERE" http://localhost:8081/users/me --verbose
```

```json
{
  "error": "invalid_token",
  "error_description": "Invalid JWT/JWS: kid is a required JOSE Header"
}
```

The issue is related to the `kid` attribute that was mentioned earlier in this guide.
Given that the JWKS endpoint returns a list of public keys, the `kid` attribute in the JWT header is required to identify, which of the keys the resource server should use to verify the token.
Unfortunately, the default implementation of the `JwtAccessTokenConverter` we've defined in the `AuthorizationServerConfigurer` does not add a `kid` attribute to the JWT header.

Decoding the JWT token issued by the authorization server, reveals the following header:
```json
{
  "alg": "RS256",
  "typ": "JWT",
}
```

What we want the header to look like is this (setting `kid` to 1 because this is how we have identified our public key, i.e. `new RSAKey.Builder(publicKey).keyID("1")...`):
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "1"
}
```

At present, it is not possible to easily extend `JwtAccessTokenConverter` to provide customer header attributes.
The only workaround is to extend the entire class and override the protected `encode` method responsible for creating the header.

```java
package spring.boot.oauth2.authorization.server;

import org.springframework.security.jwt.JwtHelper;
import org.springframework.security.jwt.crypto.sign.RsaSigner;
import org.springframework.security.jwt.crypto.sign.Signer;
import org.springframework.security.oauth2.common.OAuth2AccessToken;
import org.springframework.security.oauth2.common.util.JsonParser;
import org.springframework.security.oauth2.common.util.JsonParserFactory;
import org.springframework.security.oauth2.provider.OAuth2Authentication;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
import org.springframework.util.Assert;

import java.security.KeyPair;
import java.security.PrivateKey;
import java.security.interfaces.RSAPrivateKey;
import java.util.Map;

public class JwtAccessTokenWithKidConverter extends JwtAccessTokenConverter {

    private JsonParser objectMapper = JsonParserFactory.create();
    private Signer signer;

    @Override
    public void setKeyPair(KeyPair keyPair) {
        super.setKeyPair(keyPair);
        PrivateKey privateKey = keyPair.getPrivate();
        Assert.state(privateKey instanceof RSAPrivateKey, "KeyPair must be an RSA ");
        signer = new RsaSigner((RSAPrivateKey) privateKey);
    }

    @Override
    public void setSigningKey(String key) {
        super.setSigningKey(key);
        if (key.startsWith("-----BEGIN")) {
            signer = new RsaSigner(key);
        }
    }

    @Override
    protected String encode(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
        String content;
        try {
            content = objectMapper.formatMap(getAccessTokenConverter().convertAccessToken(accessToken, authentication));
        }
        catch (Exception e) {
            throw new IllegalStateException("Cannot convert access token to JSON", e);
        }
        return JwtHelper.encode(content, signer, Map.of("kid", "1")).getEncoded();
    }
}
```

The `JwtAccessTokenConverter.encode` method uses several private attributes that we need to somehow redefine to be able to adequately reproduce its functionality.

The key attribute is the `signer`, which as the name suggests is responsible for encapsulating the authorization server's private key used for digitally signing access tokens.

To keep the functionality as closely as possible to the parent class to ensure that we do not inadvertantly break its usage,
we need to identify all points within the `JwtAccessTokenConverter` where the `signer` attribute is set.
At each point, we need to ensure that we get access to the signer within our custom implementation.
The only way to do this, is to create a new object, in the same way as the original methods: `setKeyPair` and `setSigningKey`.

> In the sample above, the `kid` value is hardcoded, just like in the `JwkSetEndpoint` class.
> To productionise this, you will want to add it to the `application.yaml` properties file along with the private key.

Once this is done, we need to re-wire the `AuthorizationServerConfigurer` to use our new converter, e.g.

```java
@Bean
public JwtAccessTokenConverter accessTokenConverter() {
    var converter = new JwtAccessTokenWithKidConverter();
    converter.setKeyPair(getKeyPair());
    return converter;
}
```

### Conclusion

In this guide, we have extended an OAuth2 authorization server to support OIDC compliant JWKs URI endpoint.

Subsequently, we have changed the resource server we built in the previous [guide][spring-boot-oauth2-resource-server.post] to use the new endpoint to retrieve authorization server's public key.

Finally, we have worked through existing limitations of the `JwtAccessTokenConverter` class and discussed a workaround for adding a `kid` attribute to the JWT access token header.

The complete, working solution is available in [GitHub][spring-boot-oauth2-authorization-server-jwks.git].

[spring-boot-oauth2-authorization-server.post]: /2019/06/28/spring-boot-2-oauth2-authorization-server
[spring-boot-oauth2-resource-server.post]: 2019/07/06/spring-boot-2-oauth2-resource-server
[spring-boot-oauth2-resource-server.git]: https://github.com/academyhq/spring-boot-oauth2-resource-server
[spring-boot-oauth2-resource-server-jwks.git]: https://github.com/academyhq/spring-boot-oauth2-resource-server/tree/jwks
[spring-boot-oauth2-authorization-server.git]: https://github.com/academyhq/spring-boot-oauth2-authorization-server
[spring-boot-oauth2-authorization-server-jwks.git]: https://github.com/academyhq/spring-boot-oauth2-authorization-server/tree/jwks
