# Redirecting users from rules

[Rules](/rules) allow you to define arbitrary code which can be used to fulfill custom authentication/authorization requirements, log events, retrieve information from external services, and much more.
They can also be used to programatically redirect users before completing an authentication transaction.
This allows implementing custom authentication flows which require input on behalf of the user, such as:

* Requiring users to provide additional verification when logging from unknown locations
* Implementing custom verification mechanisms (e.g. proprietary multifactor authentication providers)
* Forcing users to change passwords

## How to use it

To redirect a user from a rule, set the `context.redirect` property as follows:

```js
function (user, context, callback) {
    context.redirect = {
        url: "https://example.com/foo"
    };
    return callback(null, user, context);
}
```

Once all rules have finished executing, the user will be redirected to the specified URL.

## What to do after redirecting

An authentication transaction that has been interrupted by setting `context.redirect` can be resumed by redirecting the user to the following URL:

```text
https://${account.namespace}/continue
```

When a user has been redirected to the `/continue` endpoint, all rules will be run again.
To distinguish between user-initiated logins and resumed login flows, the `context.protocol` property can be checked:

```js
function (user, context, callback) {
    if (context.protocol === "redirect-callback") {
        // User was redirected to the /continue endpoint
    } else {
        // User is logging in directly
    }
}
```

## Securely processing results after redirecting

Suppose we'd like to force users to change their passwords under a specific condition.
We can write a rule that would have the following behavior:

1. User attempts to log in and needs to change their password
2. User is redirected to an application-specific page with a JWT in the query string.
This JWT ensures that only this user's password can be changed, and **must be validated** by the application.
3. User changes their password in the application-specific page by having the application [call the Auth0 API](https://auth0.com/docs/api/v2#!/Users/patch_users_by_id)
4. Application redirects back to `/continue`, with a JWT in the query string.
This token must be issued for the same user that is attempting to log in, and must contain a `passwordChanged: true` claim.

```js
function(user, context, callback) {
  // Prerequisites:
  // 1. Implement a `mustChangePassword` function
  // 2. Set configuration variables for the following:
  // * CLIENT_ID
  // * CLIENT_SECRET
  // * ISSUER
  if (context.protocol !== "redirect-callback") {
    if (mustChangePassword(user)) {
      // User has initiated a login and is forced to change their password
      // Send user's information in a JWT to avoid tampering
      function createToken(clientId, clientSecret, issuer, user) {
        var options = {
          expiresInMinutes: 5,
          audience: clientId,
          issuer: issuer
        };
        return jwt.sign(user, new Buffer(clientSecret, "base64"), options);
      }
      var token = createToken(
        configuration.CLIENT_ID,
        configuration.CLIENT_SECRET,
        configuration.ISSUER, {
          sub: user.user_id,
          email: user.email
        }
      );
      context.redirect = {
        url: "https://example.com/change-pw?token=" + token
      };
    }
  } else {
    // User has been redirected to /continue?token=..., password change must be validated
    // The generated token must include a `passwordChanged` claim to confirm the password change
    function verifyToken(clientId, clientSecret, issuer, token, cb) {
      jwt.verify(
        token,
        new Buffer(clientSecret, "base64").toString("binary"), {
          audience: clientId,
          issuer: issuer
        },
        cb
      );
    }
    function postVerify(err, decoded) {
      if (err) {
        return callback(new UnauthorizedError("Password change failed"));
      } else if (decoded.sub !== user.user_id) {
        return callback(new UnauthorizedError("Token does not match the current user"));
      } else if (!decoded.passwordChanged) {
        return callback(new UnauthorizedError("Password change was not confirmed"));
      } else {
        // User's password has been changed successfully
        return callback(null, user, context);
      }
    }
    verifyToken(
      configuration.CLIENT_ID,
      configuration.CLIENT_SECRET,
      configuration.ISSUER,
      context.request.query.token,
      postVerify
    );
  }
}
```
