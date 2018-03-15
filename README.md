# Apigee Edge Proxy demonstrating RFC 7523 Token Exchange

This API Proxy bundle demonstrates token exchange - JWT for an opaque OAuth token as decribed by [RFC 7523](https://tools.ietf.org/html/rfc7523).

This is the token exchange process employed [by Google for service-to-service invocation](https://developers.google.com/identity/protocols/OAuth2ServiceAccount) of various commercial APIs, such as Stackdriver, Drive, DLP, and so on.  [Google Assitant](https://developers.google.com/actions/identity/oauth2-assertion-flow) also uses this flow to obtain tokens from external systems.

This example shows you how you can implement this flow within your Apigee organization, to facilitate integration with Google Assistant, or to allow other third-party apps to authenticate securely.


## Disclaimer

This example is not an official Google product, nor is it part of an official Google product. It's an example.


## Dependencies

The runtime for example depends on the JWT policies for Apigee Edge, which were first available in Edge SaaS in January 2018.

The provisioning tool depends on the utilities {curl and openssl}, and also the bash shell, and the Java-based tool JwtTool, which is included here. To build the Java tool you need maven and JDK8.


## The jwt2token Proxy Endpoint

This endpoint performs the token exchange. The basepath is /rfc7523/jwt2token.

It accepts as input a POST /token

with a `x-www-form-urlencoded` payload that includes:

* grant_type = `urn:ietf:params:oauth:grant-type:jwt-bearer`
* assertion = a JWT

The JWT must:
* be signed via RS256 using the public key belonging to the developer.
* have an expiry of no greater than 300s.
* have never been previously used to obtain a token.
* have the correct audience and scope and issuer (==consumer key).

If all these checks  pass, then the proxy generates a client_credentials oauth token and returns it.

## Why

The reason to do token exchange is to allow fast server-side checking of
tokens. JWT are computationally expensive to parse and verify, because they use
public/private key signatures. On the other hand, an opaque OAuth token
generated by Apigee Edge, is really easy and cheap for Apigee Edge to
verify.

Exchanging a JWT for an opaque token allows a faster token check on the server side, during many many API requests. 

This token exchange - a JWT identifying the service for an opaque oauth token - is the pattern used by APIs for most public Google services.

## The Token Exchange Logic

Here's how the thing works:
* There must be a developer entity registered in Edge
* There must be an API product in Apigee Edge
* There must be an app for the developr, authorized on the API Product
* The app must have a custom attribute named "public_key", and its contents must be the PEM/PKCS8 encoding of a public key.
* The app (or client) must generate a JWT, which is signed with the corresponding private key
* The app sends in the JWT to request an opaque token
* Apigee Edge checks the JWT, and issues the token if everything is valid

The check for validity involves:
* The issuer must be a valid API Key registered in Apigee Edge. Not expired nor revoked.
* The total lifetime of the JWT must be no longer than 5 minutes.
* The issued-at time must be valid.  The not-before-time, if it exists, must be valid.
* The signature is correct.
* The JWT cannot have been used previously.  (Edge keeps a cache. )


## Provisioning the System

To provision the keys, the developer, the product, and the app, run the provisioning script, like the following:

```
tools/provisionKeysProxyProductDeveloperAndApp.sh   -o ORGNAME  -e ENVNAME
```

Specify your organization name and environment name as appropriate.

The output will finish something like:

```
private key file: private-pkcs8-20161214-201107.pem
public key file: public-20161214-201107.pem

consumer key: ayBZRVAGlmG8BnbQ0YllkCywvi3Ko9wI
consumer secret: m9cdAYxXtu3m7Nde
```

It will create a keypair, or re-use an existing keypair if there is a unique one.


If you want to skip import and deploy of the proxy, you can pass the `-S` option.

```
tools/provisionKeysProxyProductDeveloperAndApp.sh   -o ORGNAME  -e ENVNAME -S
```

The above will create a new app and upload a new public key.


## Generating the required JWT

1. First, build the Java tool that generates the JWT:

   ```
   cd tools/jwttool
   mvn clean install
   cd ../..
   ```

2. then, run the wrapper script to generate a JWT:

   ```
   tools/createJwt.sh -k PRIVATE_KEY_FILE  -i CONSUMER_KEY_HERE
   ```

The CONSUMER_KEY_HERE and the PRIVATE_KEY_FILE that you use here, must be the values shown by
the provisioning script.

You can then check the generated JWT like so:

```
tools/checkJwt.sh -k PUBLIC_KEY_FILE  -t TOKEN
```

You must pass the appropriate public key file and token here.


## Invoking the Proxy to Perform the Exchange

Use this command:

```
curl -X POST -H content-type:application/x-www-form-urlencoded \
  https://ORGNAME-ENVNAME.apigee.net/rfc7523/jwt2token/token \
 -d  'grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion=JWT_HERE'
```

You will need to insert the rather long JWT in the appropriate place.
Also insert your ORGNAME and ENVNAME as appropriate.


## Deleting the Provisioned artifacts from the System

To remove the developer, the product, and the app, the cache, and the proxy, run the provisioning script with the -r option, like the following:

```
tools/provisionKeysProxyProductDeveloperAndApp.sh   -o ORGNAME  -e ENVNAME -r

```

## Copyright and License

This material is Copyright (c) 2016-2018 Google, Inc.
and is licensed under the [Apache 2.0 License](LICENSE).
