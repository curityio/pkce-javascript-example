# A simple JavaScript PKCE example

## Introduction

The `OAuth Code Flow` is one of the more advanced and flexible token flows and with that also very popular. The details of this flow is not covered by this article but can be found here, [Code Flow Overview](https://curity.io/resources/develop/oauth/oauth-code-flow/index.html).

Proof Key for Code Exchange (PKCE) is a technique described in [RFC7636](https://tools.ietf.org/html/rfc7636) that is used to mitigate the risk of the authorization code being hijacked. More details on how to configure the Curity Identity Server to enable PKCE can be found here, [Configure PKCE in Curity](https://curity.io/resources/operate/tutorials/advanced/pkce/) and further details on PKCE is available here, [Proof Key for Code Exchange (PKCE)](https://curity.io/resources/architect/oauth/oauth-pkce/).

## Configuration

### Client
The client -- the HTML page -- needs to be configured with the client ID. By default, this is `public-test-client`. If certain scopes are desired, these should be configured as well.

```JavaScript
const clientId = "public-test-client";
```

#### Creating the secret
This function generates a random string (the secret) that is later signed before its sent to the authorization server.

```JavaScript
function generateRandomString(length) {
  var text = "";
  var possible = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";

  for (var i = 0; i < length; i++) {
    text += possible.charAt(Math.floor(Math.random() * possible.length));
  }

  return text;
}
```

#### Signing the secret
The Web crypto API is used to sign the key using SHA-256

```JavaScript
async function generateCodeChallenge(codeVerifier) {
  var digest = await crypto.subtle.digest("SHA-256",
    new TextEncoder().encode(codeVerifier));

  return btoa(String.fromCharCode(...new Uint8Array(digest)))
    .replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_')
}
```

#### Storing the key
Store the signed key between requests (using session storage)

```JavaScript
window.sessionStorage.setItem("code_verifier", codeVerifier);
```

#### Sending the key with the authorization requests
The signed secret (codeChallenge) is passed to the authorization server as part of the authorization request. The method (`S256` in our case) used for signing the secret is also passed with the request.

```JavaScript
var redirectUri = window.location.href.split('?')[0];
var args = new URLSearchParams({
  response_type: "code",
  client_id: clientId,
  code_challenge_method: "S256",
  code_challenge: codeChallenge,
  redirect_uri: redirectUri
});
window.location = authorizeEndpoint + "/?" + args;
```

#### Call the token endpoint with the code and key
The authorization code is passed in the POST request to the token endpoint along with the secret key (retrieved from the session storage).
```JavaScript
xhr.responseType = 'json';
xhr.open("POST", tokenEndpoint, true);
xhr.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');
xhr.send(new URLSearchParams({
  client_id: clientId,
  code_verifier: window.sessionStorage.getItem("code_verifier"),
  grant_type: "authorization_code",
  redirect_uri: location.href.replace(location.search, ''),
  code: code
}));
```

### OAuth Server
The OAuth server needs to be configured with a client that matches the one configured [above](#Client). Also, the redirect should be set. When using `npx` (described below), this will be `http://localhost:8080` by default if no port is provided. Additionally, scopes may be configured.

This can be created in the Curity Identity Server by merging this XML with the current configuration:

```xml
<config xmlns="http://tail-f.com/ns/config/1.0">
  <profiles xmlns="https://curity.se/ns/conf/base">
  <profile>
    <id>my-good-oauth-profile</id> <!-- Replace with the ID of your OAuth profile -->
    <type xmlns:as="https://curity.se/ns/conf/profile/oauth">as:oauth-service</type>
      <settings>
      <authorization-server xmlns="https://curity.se/ns/conf/profile/oauth">
      <client-store>
      <config-backed>
      <client>
        <id>public-test-client</id>
        <no-authentication>true</no-authentication>
        <redirect-uris>http://localhost:8080/</redirect-uris> <!-- Update with your URL -->
        <capabilities>
          <code/>
        </capabilities>      
        <validate-port-on-loopback-interfaces>false</validate-port-on-loopback-interfaces>
      </client>
      </config-backed>
      </client-store>
      </authorization-server>
      </settings>
  </profile>
  </profiles>
</config>
```

## Serving the file

The HTML needs to be served somehow from a Web server. Because the client is just a static HTML page, this can be done with a trivial server configuration. These are a couple of different ways to very easily server the static HTML page:

```sh
$ npx http-server -p <port>
```

```sh
$ php -S <host>:<port>
```

```sh
$ python -m SimpleHTTPServer <port>
```

These will not use TLS, but are fast and easy ways to serve the HTML file without setting up any infrastructure.
