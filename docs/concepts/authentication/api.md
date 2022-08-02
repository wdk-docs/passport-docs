# API

## OAuth 2.0

OAuth 2.0(正式由 RFC 6749 指定)提供了一个授权框架，允许用户授权访问第三方应用程序。
获得授权后，向应用程序颁发一个令牌，以用作身份验证凭据。
这有两个主要的安全好处:

- 应用程序不需要存储用户的用户名和密码。
- 令牌可以有一个受限制的范围(例如:只读访问)。

这些好处对于确保 web 应用程序的安全性特别重要，使 OAuth 2.0 成为 API 认证的主要标准。

当使用 OAuth 2.0 来保护 API 端点时，必须执行三个不同的步骤:

- 应用程序向用户请求访问受保护资源的权限。
- 如果用户授予权限，则向应用程序颁发令牌。
- 应用程序使用令牌进行身份验证以访问受保护的资源。

### 发行令牌

OAuth2orize 是 Passport 的兄弟项目，提供了一个用于实现 OAuth 2.0 授权服务器的工具包。

授权过程是一个复杂的序列，包括对请求应用程序和用户进行身份验证，并提示用户获得许可，确保为用户提供足够的细节以做出明智的决定。

此外，由实现者决定在访问范围方面可以对应用程序施加哪些限制，以及随后实施这些限制。

作为一个工具包，OAuth2orize 不打算做出实现决策。
本指南没有涉及这些问题，但是强烈建议部署 OAuth 2.0 的服务对涉及的安全考虑有一个完整的理解。

### 验证令牌

OAuth 2.0 提供了一个框架，在这个框架中可以发布任意扩展的令牌类型集。
在实践中，只有特定的令牌类型得到了广泛使用。

### 不记名的令牌

无记名令牌是 OAuth 2.0 中发行最广泛的令牌类型。
事实上，许多实现都假定无记名令牌是唯一发行的令牌类型。

不记名令牌可以使用 `passport-http-bearer` 模块进行身份验证。

### 安装

```sh
$ npm install passport-http-bearer
```

### 配置

```js
passport.use(
  new BearerStrategy(function (token, done) {
    User.findOne({ token: token }, function (err, user) {
      if (err) {
        return done(err);
      }
      if (!user) {
        return done(null, false);
      }
      return done(null, user, { scope: "read" });
    });
  })
);
```

不记名令牌的验证回调接受令牌作为参数。
当调用 done 时，可选信息可以被传递，它将由 Passport 在 req.authInfo 设置。
这通常用于传递令牌的范围，并可在进行访问控制检查时使用。

### 保护端点

```js
app.get(
  "/api/me",
  passport.authenticate("bearer", { session: false }),
  function (req, res) {
    res.json(req.user);
  }
);
```

使用承载策略指定 passport.authenticate()以保护 API 端点。
api 通常不需要会话，因此可以禁用会话。

## OAuth

OAuth(由 RFC 5849 正式指定)为用户提供了一种向第三方应用程序授予对其数据访问权的方法，而无需将其密码暴露给这些应用程序。

该协议极大地提高了 web 应用程序的安全性，尤其是，OAuth 在引起人们对将密码暴露给外部服务的潜在危险的注意方面发挥了重要作用。

虽然 OAuth 1.0 仍然被广泛使用，但它已经被 OAuth 2.0 所取代。
建议新的实现基于 OAuth 2.0。

当使用 OAuth 来保护 API 端点时，必须执行三个不同的步骤:

应用程序向用户请求访问受保护资源的权限。
如果用户授予权限，则向应用程序颁发令牌。
应用程序使用令牌进行身份验证以访问受保护的资源。

### 发行令牌

OAuthorize 是 Passport 的兄弟项目，它提供了一个用于实现 OAuth 服务提供商的工具包。

授权过程是一个复杂的序列，包括对请求应用程序和用户进行身份验证，并提示用户获得许可，确保为用户提供足够的细节以做出明智的决定。

此外，由实现者决定在访问范围方面可以对应用程序施加哪些限制，以及随后实施这些限制。

作为一个工具包，OAuthorize 不试图做出实现决策。
本指南没有涉及这些问题，但是强烈建议部署 OAuth 的服务对涉及的安全考虑有一个完整的理解。

### 验证令牌

签发后，可以使用 passport-http-oauth 模块对 OAuth 令牌进行身份验证。

### 安装

```sh
$ npm install passport-http-oauth
```

### 配置

```js
passport.use(
  "token",
  new TokenStrategy(
    function (consumerKey, done) {
      Consumer.findOne({ key: consumerKey }, function (err, consumer) {
        if (err) {
          return done(err);
        }
        if (!consumer) {
          return done(null, false);
        }
        return done(null, consumer, consumer.secret);
      });
    },
    function (accessToken, done) {
      AccessToken.findOne({ token: accessToken }, function (err, token) {
        if (err) {
          return done(err);
        }
        if (!token) {
          return done(null, false);
        }
        Users.findById(token.userId, function (err, user) {
          if (err) {
            return done(err);
          }
          if (!user) {
            return done(null, false);
          }
          // fourth argument is optional info.typically used to pass details needed to authorize the request (ex: `scope`)
          return done(null, user, token.secret, { scope: token.scope });
        });
      });
    },
    function (timestamp, nonce, done) {
      // validate the timestamp and nonce as necessary
      done(null, true);
    }
  )
);
```

In contrast to other strategies, there are two callbacks required by OAuth.
In OAuth, both an identifier for the requesting application and the user-specific token are encoded as credentials.

The first callback is known as the "consumer callback", and is used to find the application making the request, including the secret assigned to it.
The second callback is the "token callback", which is used to identify the user as well as the token's corresponding secret.
The secrets supplied by the consumer and token callbacks are used to compute a signature, and authentication fails if it does not match the request signature.

A final "validate callback" is optional, which can be used to prevent replay attacks by checking the timestamp and nonce used in the request.

### 保护断点

```js
app.get(
  "/api/me",
  passport.authenticate("token", { session: false }),
  function (req, res) {
    res.json(req.user);
  }
);
```

Specify passport.authenticate() with the token strategy to protect API endpoints.
Sessions are not typically needed by APIs, so they can be disabled.

## Basic & Digest

除了定义 HTTP 的身份验证框架外，RFC 2617 还定义了基本和摘要身份验证方案。
这两种方案都使用用户名和密码作为认证用户的凭证，通常用于保护 API 端点。

应该注意的是，依赖用户名和密码凭据可能会产生不利的安全影响，特别是在服务器和客户机之间没有高度信任的情况下。
在这些情况下，建议使用 OAuth 2.0 这样的授权框架。

passport-http 模块提供了对基本和摘要模式的支持。

### 安装

```sh
$ npm install passport-http
```

### Basic

The Basic scheme uses a username and password to authenticate a user.
These credentials are transported in plain text, so it is advised to use HTTPS when implementing this scheme.

##### 配置

```js
passport.use(
  new BasicStrategy(function (username, password, done) {
    User.findOne({ username: username }, function (err, user) {
      if (err) {
        return done(err);
      }
      if (!user) {
        return done(null, false);
      }
      if (!user.validPassword(password)) {
        return done(null, false);
      }
      return done(null, user);
    });
  })
);
```

The verify callback for Basic authentication accepts username and password arguments.

#### 保护端点

```js
app.get(
  "/api/me",
  passport.authenticate("basic", { session: false }),
  function (req, res) {
    res.json(req.user);
  }
);
```

Specify passport.authenticate() with the basic strategy to protect API endpoints.
Sessions are not typically needed by APIs, so they can be disabled.

### Digest

The Digest scheme uses a username and password to authenticate a user.
Its primary benefit over Basic is that it uses a challenge-response paradigm to avoid sending the password in the clear.

#### 配置

```js
passport.use(
  new DigestStrategy(
    { qop: "auth" },
    function (username, done) {
      User.findOne({ username: username }, function (err, user) {
        if (err) {
          return done(err);
        }
        if (!user) {
          return done(null, false);
        }
        return done(null, user, user.password);
      });
    },
    function (params, done) {
      // validate nonces as necessary
      done(null, true);
    }
  )
);
```

The Digest strategy utilizes two callbacks, the second of which is optional.

The first callback, known as the "secret callback" accepts the username and calls done supplying a user and the corresponding secret password.
The password is used to compute a hash, and authentication fails if it does not match that contained in the request.

The second "validate callback" accepts nonce related params, which can be checked to avoid replay attacks.

#### 保护端点

```js
app.get(
  "/api/me",
  passport.authenticate("digest", { session: false }),
  function (req, res) {
    res.json(req.user);
  }
);
```

Specify passport.authenticate() with the digest strategy to protect API endpoints.
Sessions are not typically needed by APIs, so they can be disabled.
