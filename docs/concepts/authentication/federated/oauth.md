# OAuth

OAuth is a standard protocol that allows users to authorize API access to web and desktop or mobile applications. Once access has been granted, the authorized application can utilize the API on behalf of the user. OAuth has also emerged as a popular mechanism for delegated authentication.

OAuth comes in two primary flavors, both of which are widely deployed.

The initial version of OAuth was developed as an open standard by a loosely organized collective of web developers. Their work resulted in OAuth 1.0, which was superseded by OAuth 1.0a. This work has now been standardized by the IETF as RFC 5849.

Recent efforts undertaken by the Web Authorization Protocol Working Group have focused on defining OAuth 2.0. Due to the lengthy standardization effort, providers have proceeded to deploy implementations conforming to various drafts, each with slightly different semantics.

Thankfully, Passport shields an application from the complexities of dealing with OAuth variants. In many cases, a provider-specific strategy can be used instead of the generic OAuth strategies described below. This cuts down on the necessary configuration, and accommodates any provider-specific quirks. See Facebook, Twitter or the list of providers for preferred usage.

Support for OAuth is provided by the passport-oauth module.

## Install

```sh
$ npm install passport-oauth
```

OAuth 1.0
OAuth 1.0 is a delegated authentication strategy that involves multiple steps. First, a request token must be obtained. Next, the user is redirected to the service provider to authorize access. Finally, after authorization has been granted, the user is redirected back to the application and the request token can be exchanged for an access token. The application requesting access, known as a consumer, is identified by a consumer key and consumer secret.

## Configuration

When using the generic OAuth strategy, the key, secret, and endpoints are specified as options.

```js
var passport = require('passport')
, OAuthStrategy = require('passport-oauth').OAuthStrategy;

passport.use('provider', new OAuthStrategy({
requestTokenURL: 'https://www.provider.com/oauth/request_token',
accessTokenURL: 'https://www.provider.com/oauth/access_token',
userAuthorizationURL: 'https://www.provider.com/oauth/authorize',
consumerKey: '123-456-789',
consumerSecret: 'shhh-its-a-secret',
callbackURL: 'https://www.example.com/auth/provider/callback'
},
function(token, tokenSecret, profile, done) {
User.findOrCreate(..., function(err, user) {
done(err, user);
});
}
));
```

The verify callback for OAuth-based strategies accepts token, tokenSecret, and profile arguments. token is the access token and tokenSecret is its corresponding secret. profile will contain user profile information provided by the service provider; refer to User Profile for additional information.

## Routes

Two routes are required for OAuth authentication. The first route initiates an OAuth transaction and redirects the user to the service provider. The second route is the URL to which the user will be redirected after authenticating with the provider.

```js
// Redirect the user to the OAuth provider for authentication. When
// complete, the provider will redirect the user back to the application at
// /auth/provider/callback
app.get("/auth/provider", passport.authenticate("provider"));

// The OAuth provider has redirected the user back to the application.
// Finish the authentication process by attempting to obtain an access
// token. If authorization was granted, the user will be logged in.
// Otherwise, authentication has failed.
app.get(
  "/auth/provider/callback",
  passport.authenticate("provider", {
    successRedirect: "/",
    failureRedirect: "/login",
  })
);
```

Link
A link or button can be placed on a web page, which will start the authentication process when clicked.

```html
<a href="/auth/provider">Log In with OAuth Provider</a>
```
