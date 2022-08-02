# OAuth 2.0

OAuth 2.0 是 OAuth 1.0 的继承者，旨在克服早期版本的缺陷。
身份验证流程本质上是相同的。
用户首先被重定向到服务提供者以授权访问。
授权授予后，用户被重定向回应用程序，其中的代码可以用来交换访问令牌。
请求访问的应用程序(称为客户机)由 ID 和秘密标识。

## 配置

当使用通用的 OAuth 2.0 策略时，客户端 ID、客户端机密和端点被指定为选项。

```js
var passport = require('passport'), OAuth2Strategy = require('passport-oauth').OAuth2Strategy;

passport.use('provider', new OAuth2Strategy(
    {
        authorizationURL: 'https://www.provider.com/oauth2/authorize',
        tokenURL: 'https://www.provider.com/oauth2/token',
        clientID: '123-456-789',
        clientSecret: 'shhh-its-a-secret'
        callbackURL: 'https://www.example.com/auth/provider/callback'
    },
    function(accessToken, refreshToken, profile, done) {
        User.findOrCreate('...', function(err, user) {
            done(err, user);
        });
    }
));
```

基于 OAuth 2.0 策略的验证回调接受 accessToken、refreshToken 和 profile 参数。
refreshToken 可用于获取新的访问令牌，如果提供者不发出刷新令牌，则可能是未定义的。
资料将包含服务提供商提供的用户资料信息;
更多信息请参考用户配置文件。

## 路由

OAuth 2.0 身份验证需要两条路由。

- 第一个路由将用户重定向到服务提供者。
- 第二个路由是 URL，用户通过提供者的身份验证后将被重定向到该 URL。

```js
// Redirect the user to the OAuth 2.0 provider for authentication.
// When complete, the provider will redirect the user back to the application at /auth/provider/callback
app.get("/auth/provider", passport.authenticate("provider"));

// The OAuth 2.0 provider has redirected the user back to the application.
// Finish the authentication process by attempting to obtain an access token.
// If authorization was granted, the user will be logged in.
// Otherwise, authentication has failed.
app.get(
  "/auth/provider/callback",
  passport.authenticate("provider", {
    successRedirect: "/",
    failureRedirect: "/login",
  })
);
```

## 作用域

当使用 OAuth 2.0 请求访问时，访问作用域由 `scope` 选项控制。

```js
app.get(
  "/auth/provider",
  passport.authenticate("provider", { scope: "email" })
);
```

可以将多个作用域指定为数组。

```js
app.get(
  "/auth/provider",
  passport.authenticate("provider", { scope: ["email", "sms"] })
);
```

作用域选项的值是特定于提供程序的。
有关受支持范围的详细信息，请参阅提供程序的文档。

## 链接

可以在网页上放置一个链接或按钮，点击后将启动认证过程。

```html
<a href="/auth/provider">使用OAuth 2.0提供程序登录</a>
```
