# general

## 中间件

Passport 用作 web 应用程序中验证请求的中间件。
中间件在“Node.js”中被 Express 和它更简约的兄弟 Connect 所普及。
考虑到它的流行程度，中间件很容易适应其他 web 框架。

下面的代码是一个使用用户名和密码验证用户的路由示例:

```js title="app.js"
app.post(
  "/login/password",
  passport.authenticate("local", {
    failureRedirect: "/login",
    failureMessage: true,
  }),
  function (req, res) {
    res.redirect("/~" + req.user.username);
  }
);
```

In this route, passport.authenticate() is middleware which will authenticate the request.
By default, when authentication succeeds, the req.user property is set to the authenticated user, a login session is established, and the next function in the stack is called.
This next function is typically application-specific logic which will process the request on behalf of the user.

When authentication fails, an HTTP 401 Unauthorized response will be sent and the request-response cycle will end.
Any additional functions in the stack will not be called.
This default behavior is suitable for APIs obeying representational state transfer (REST) constaints, and can be modified using options.

In traditional web applications, which interact with the user via HTML pages, forms, and redirects, the failureRedirect option is commonly used.
Instead of responding with 401 Unauthorized, the browser will be redirected to the given location with a 302 Found response.
This location is typically the login page, which gives the user another attempt to log in after an authentication failure.
This is often paired with the failureMessage option, which will add an informative message to the session about why authentication failed which can then be displayed to the user.

The mechanism used to authenticate the request is implemented by a strategy.
Authenticating a user with a username and password entails a different set of operations than authenticating a user via OpenID Connect.
As such, those two mechanisms are implemented by two different strategies.
In the route above, the local strategy is used to verify a username and password.

## 策略

策略负责对请求进行身份验证，它们通过实现身份验证机制来完成。
身份验证机制定义了如何在请求中编码凭证，例如密码或来自身份提供者(IdP)的断言。
它们还指定了验证该凭证所需的程序。
如果证书验证成功，则对请求进行身份验证。

有各种各样的身份验证机制，以及相应的各种策略。
策略分布在独立的包中，必须安装、配置和注册这些包。

### 安装

Strategies are published to the npm registry, and installed using a package manager.

For example, the following command will install passport-local, a package which provides a strategy for authenticating with a username and password:

```sh
$ npm install passport-local
```

And the following command will install passport-openidconnect, a package which implements support for OpenID Connect:

```sh
$ npm install passport-openidconnect
```

Developers only need to install the packages which provide authentication mechanisms required by the application.
These packages are then plugged into Passport.
This reduces overall application size by avoiding unnecessary dependencies.

### 配置

Once a package has been installed, the strategy needs to be configured.
The configuration varies with each authentication mechanism, so strategy-specific documentation should be consulted.
That being said, there are common patterns that are encountered across many strategies.

The following code is an example that configures the LocalStrategy:

```js
var LocalStrategy = require("passport-local");

var strategy = new LocalStrategy(function verify(username, password, cb) {
  db.get(
    "SELECT * FROM users WHERE username = ?",
    [username],
    function (err, user) {
      if (err) {
        return cb(err);
      }
      if (!user) {
        return cb(null, false, { message: "Incorrect username or password." });
      }

      crypto.pbkdf2(
        password,
        user.salt,
        310000,
        32,
        "sha256",
        function (err, hashedPassword) {
          if (err) {
            return cb(err);
          }
          if (!crypto.timingSafeEqual(user.hashed_password, hashedPassword)) {
            return cb(null, false, {
              message: "Incorrect username or password.",
            });
          }
          return cb(null, user);
        }
      );
    }
  );
});
```

#### 验证功能

The LocalStrategy constructor takes a function as an argument.
This function is known as a verify function, and is a common pattern in many strategies.
When authenticating a request, a strategy parses the credential contained in the request.
A verify function is then called, which is responsible for determining the user to which that credential belongs.
This allows data access to be delegated to the application.

In this particular example, the verify function is executing a SQL query to obtain a user record from the database and, after verifying the password, yielding the record back to the strategy, thus authenticating the user and establishing a login session.

Because a verify function is supplied by the application itself, access to persistent storage is not constrained in any way.
The application is free to use any data storage system, including relational databases, graph databases, or document stores, and structure data within that database according to any schema.

A verify function is strategy-specific, and the exact arguments it receives and parameters it yields will depend on the underlying authentication mechanism.
For authentication mechanisms involving shared secrets, such as a password, a verify function is responsible for verifying the credential and yielding a user.
For mechanisms that provide cryptographic authentication, a verify function will typically yield a user and a key, the later of which the strategy will use to cryptographically verify the credential.

A verify function yields under one of three conditions: success, failure, or an error.

If the verify function finds a user to which the credential belongs, and that credential is valid, it calls the callback with the authenticating user:

```js
return cb(null, user);
```

If the credential does not belong to a known user, or is not valid, the verify function calls the callback with false to indicate an authentication failure:

```js
return cb(null, false);
```

If an error occurs, such as the database not being available, the callback is called with an error, in idiomatic Node.js style:

```js
return cb(err);
```

It is important to distinguish between the two failure cases that can occur.
Authentication failures are expected conditions, in which the server is operating normally, even though invalid credentials are being received from the user (or a malicious adversary attempting to authenticate as the user).
Only when the server is operating abnormally should err be set, to indicate an internal error.

### 注册

配置好策略后，然后通过调用`.use()`来注册它:

```js
var passport = require("passport");

passport.use(strategy);
```

All strategies have a name which, by convention, corresponds to the package name according to the pattern passport-{name}.
For instance, the LocalStrategy configured above is named local as it is distributed in the passport-local package.

Once registered, the strategy can be employed to authenticate a request by passing the name of the strategy as the first argument to passport.authenticate() middleware:

```js
app.post(
  "/login/password",
  passport.authenticate("local", {
    failureRedirect: "/login",
    failureMessage: true,
  }),
  function (req, res) {
    res.redirect("/~" + req.user.username);
  }
);
```

In cases where there is a naming conflict, or the default name is not sufficiently descriptive, the name can be overridden when registering the strategy by passing a name as the first argument to .use():

```js
var passport = require('passport');

passport.use('password', strategy);
That name is then specified to passport.authenticate() middleware:

app.post('/login/password',
passport.authenticate('password', { failureRedirect: '/login', failureMessage: true }),
function(req, res) {
res.redirect('/~' + req.user.username);
});
```

For brevity, strategies are often configured and registered in a single statement:

```js
var passport = require('passport');
var LocalStrategy = require('passport-local');

passport.use(new LocalStrategy(function verify(username, password, cb) {
// ...
});
```

## 会话

web 应用程序需要在用户浏览页面时识别用户的能力。
这一系列请求和响应(每个请求和响应都与同一用户关联)称为会话。

HTTP is a stateless protocol, meaning that each request to an application can be understood in isolation - without any context from previous requests.
This poses a challenge for web applications with logged in users, as the authenticated user needs to be remembered across subsequent requests as they navigate the application.

To solve this challenge, web applications make use of sessions, which allow state to be maintained between the application server and the user's browser.
A session is established by setting an HTTP cookie in the browser, which the browser then transmits to the server on every request.
The server uses the value of the cookie to retrieve information it needs across multiple requests.
In effect, this creates a stateful protocol on top of HTTP.

While sessions are used to maintain authentication state, they can also be used by applications to maintain other state unrelated to authentication.
Passport is carefully designed to isolate authentication state, referred to as a login session, from other state.

Applications must initialize session support in order to make use of login sessions.
In an Express app, session support is added by using express-session middleware.

```js
var session = require("express-session");

app.use(
  session({
    secret: "keyboard cat",
    resave: false,
    saveUninitialized: false,
    cookie: { secure: true },
  })
);
```

To maintain a login session, Passport serializes and deserializes user information to and from the session.
The information that is stored is determined by the application, which supplies a serializeUser and a deserializeUser function.

```js
passport.serializeUser(function (user, cb) {
  process.nextTick(function () {
    return cb(null, {
      id: user.id,
      username: user.username,
      picture: user.picture,
    });
  });
});

passport.deserializeUser(function (user, cb) {
  process.nextTick(function () {
    return cb(null, user);
  });
});
```

A login session is established upon a user successfully authenticating using a credential.
The following route will authenticate a user using a username and password.
If successfully verified, Passport will call the serializeUser function, which in the above example is storing the user's ID, username, and picture.
Any other properties of the user, such as an address or birthday, are not stored.

```js
app.post(
  "/login/password",
  passport.authenticate("local", {
    failureRedirect: "/login",
    failureMessage: true,
  }),
  function (req, res) {
    res.redirect("/~" + req.user.username);
  }
);
```

As the user navigates from page to page, the session itself can be authenticated using the built-in session strategy.
Because an authenticated session is typically needed for the majority of routes in an application, it is common to use this as application-level middleware, after session middleware.

```js
app.use(session(/* ... */);
app.use(passport.authenticate('session'));
This can also be accomplished, more succinctly, using the passport.session() alias.

app.use(session(/* ... */);
app.use(passport.session());
```

When the session is authenticated, Passport will call the deserializeUser function, which in the above example is yielding the previously stored user ID, username, and picture.
The req.user property is then set to the yielded information.

There is an inherent tradeoff between the amount of data stored in a session and database load incurred when authenticating a session.
This tradeoff is particularly pertinent when session data is stored on the client, rather than the server, using a package such as cookie-session.
Storing less data in the session will require heavier queries to a database to obtain that information.
Conversely, storing more data in the session reduces database queries while potentially exceeding the maximum amount of data that can be stored in a cookie.

This tradeoff is controlled by the application and the serializeUser and deserializeUser functions it supplies.
In contrast to the above example, the following example minimizes the data stored in the session at the expense of querying the database for every request in which the session is authenticated.

```js
passport.serializeUser(function (user, cb) {
  process.nextTick(function () {
    return cb(null, user.id);
  });
});

passport.deserializeUser(function (id, cb) {
  db.get("SELECT * FROM users WHERE id = ?", [id], function (err, user) {
    if (err) {
      return cb(err);
    }
    return cb(null, user);
  });
});
```

To balance this tradeoff, it is recommended that any user information needed on every request to the application be stored in the session.
For example, if the application displays a user element containing the user's name, email address, and photo on every page, that information should be stored in the session to eliminate what would otherwise be frequent database queries.
Specific routes, such as a checkout page, that need additional information such as a shipping address, can query the database for that data.

## 用户名和密码

A username and password is the traditional, and still most widely used, way for users to authenticate to a website.
Support for this mechanism is provided by the passport-local package.

### 安装

To install passport-local, execute the following command:

```sh
$ npm install passport-local
```

### 配置

The following code is an example that configures and registers the LocalStrategy:

```js
var passport = require('passport');
var LocalStrategy = require('passport-local');
var crypto = require('crypto');

passport.use(new LocalStrategy(function verify(username, password, cb) {
  db.get('SELECT * FROM users WHERE username = ?', [ username ], function(err, user) {
    if (err) { return cb(err); }
    if (!user) { return cb(null, false, { message: 'Incorrect username or password.' }); }

    crypto.pbkdf2(password, user.salt, 310000, 32, 'sha256', function(err, hashedPassword) {
      if (err) { return cb(err); }
      if (!crypto.timingSafeEqual(user.hashed_password, hashedPassword)) {
        return cb(null, false, { message: 'Incorrect username or password.' });
      }
      return cb(null, user);
    });
  });
});
```

The LocalStrategy constructor takes a verify function as an argument, which accepts username and password as arguments.
When authenticating a request, the strategy parses a username and password, which are submitted via an HTML form to the web application.
The strategy then calls the verify function with those credentials.

The verify function is responsible for determining the user to which the username belongs, as well as verifying the password.
Because the verify function is supplied by the application, the application is free to use a database and schema of its choosing.
The example above illustrates usage of a SQL database.

Similarly, the application is free to determine its password storage format.
The example above illustrates usage of PBKDF2 when comparing the user-supplied password with the hashed password stored in the database.

In case of authentication failure, the verify callback supplies a message, via the message option, describing why authentication failed.
This will be displayed to the user when they are re-prompted to sign in, informing them of what went wrong.

### 提示

通过呈现一个表单，提示用户使用用户名和密码登录。
这是通过定义一个路由来完成的:

```js
app.get("/login", function (req, res, next) {
  res.render("login");
});
```

The following form is an example which uses best practices:

```html
<form action="/login/password" method="post">
  <div>
    <label for="username">Username</label>
    <input
      id="username"
      name="username"
      type="text"
      autocomplete="username"
      required
    />
  </div>
  <div>
    <label for="current-password">Password</label>
    <input
      id="current-password"
      name="password"
      type="password"
      autocomplete="current-password"
      required
    />
  </div>
  <div>
    <button type="submit">Sign in</button>
  </div>
</form>
```

### 进行身份验证

当用户提交表单时，它由一个使用用户输入的用户名和密码对用户进行身份验证的路由进行处理。

```js
app.post(
  "/login/password",
  passport.authenticate("local", {
    failureRedirect: "/login",
    failureMessage: true,
  }),
  function (req, res) {
    res.redirect("/~" + req.user.username);
  }
);
```

If authentication succeeds, passport.authenticate() middleware calls the next function in the stack.
In this example, the function is redirecting the authenticated user to their profile page.

When authentication fails, the user is re-prompted to sign in and informed that their initial attempt was not successful.
This is accomplished by using the failureRedirect option, which will redirect the user to the login page, along with the failureMessage option which will add the message to req.session.messages.
