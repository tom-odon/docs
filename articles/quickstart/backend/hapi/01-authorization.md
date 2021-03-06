---
title: Authorization
description: This tutorial demonstrates how to add authorization to a Hapi.js API
---

<%= include('../../../_includes/_package', {
  org: 'auth0-samples',
  repo: 'auth0-hapi-api-samples',
  path: '01-Authenticate-RS256',
  requirements: [
    'Hapi.js 16.0.0',
    'hapi-auth-jwt2 7.2.4'
  ]
}) %>

<%= include('../_includes/_api_auth_preamble') %>

This sample demonstrates how to check for a JWT in the `Authorization` header of an incoming HTTP request and verify that it is valid. The validity check is done using the **hapi-auth-jwt2** plugin and can be applied to any endpoints you wish to protect. If the token is valid, the resources which are served by the endpoint can be released, otherwise a `401 Authorization` error will be returned.

## Install the Dependencies

The **hapi-auth-jwt2** plugin can be used to verify incoming JWTs. The **jwks-rsa** library can be used alongside it to fetch your Auth0 public key and complete the verification process. Install these libraries with npm.

```bash
npm install --save hapi-auth-jwt2 jwks-rsa
```

## Configure hapi-auth-jwt2

<%= include('../_includes/_api_jwks_description', { sampleLink: 'https://github.com/auth0-samples/auth0-hapi-api-samples/tree/master/02-Authenticate-HS256'}) %>

Set up the **hapi-auth-jwt2** plugin to fetch this public key through the **jwks-rsa** library.

```js
// server.js

const Hapi = require('hapi');
const jwt = require('hapi-auth-jwt2');
const jwksRsa = require('jwks-rsa');

const server = new Hapi.Server();

// ...

server.register(jwt, err => {
  if (err) throw err;
  server.auth.strategy('jwt', 'jwt', 'required', {
    complete: true,
    // verify the access token against the
    // remote Auth0 JWKS 
    key: jwksRsa.hapiJwt2Key({
      cache: true,
      rateLimit: true,
      jwksRequestsPerMinute: 5,
      jwksUri: `https://${account.namespace}/.well-known/jwks.json`
    }),
    verifyOptions: {
      audience: '{API_ID}',
      issuer: `https://${account.namespace}/`,
      algorithms: ['RS256']
    },
    validateFunc: validateUser
  });
  registerRoutes();
});
```

The **hapi-auth-jwt2** plugin does the work of actually verifying that the JWT is valid. However, the `validateFunc` key requires some function that can do further checking. This might simply be a check to ensure the JWT has a `sub` claim, but your needs may vary in your own application.

```js
// server.js

const validateUser = (decoded, request, callback) => {
  // This is a simple check that the `sub` claim
  // exists in the access token. Modify it to suit
  // the needs of your application
  if (decoded && decoded.sub) {
    return callback(null, true);
  }

  return callback(null, false);
}
```

## Protect Individual Endpoints

The configuration that is set up above for the **hapi-auth-jwt2** plugin specifies `required` as the third argument to the `strategy`. This means that all routes will require authentication by default. If you'd like to make a route public, you can simply pass `auth: false` to the route's `config`.

<%= include('../_includes/_api_scope_description') %>

Individual routes can be configured to look for a particular `scope` in the `access_token` using `auth.scope`.

```js
// server.js

// ...

server.route({
  method: 'GET',
  path: '/api/private',
  config: {
    auth: {
      scope: 'read:messages'
    },
    handler: (req, res) => {
      res({ message: "Hello from a private endpoint! You need to be authenticated and have a scope of read:messages to see this." });
    }
  }
});
```

With this configuration in place, only valid `access_token`s which have a scope of `read:messages` will be allowed to access this endpoint.