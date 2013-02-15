# ember-auth

`ember-auth` provides token authentication support to
[ember.js](http://emberjs.com/).

# Overview

`ember-auth` expects your server to implement token authentication.
It will then query for this token upon sign in; append this token for every
API request generated by the model query and persistence; and destroy this
token upon sign out.

# Status

## Is `ember-auth` production-ready?

No. The module isn't even finished; and in any case, I haven't learnt how to
write proper unit tests for ember modules yet.
Perhaps [you can help](#contributing)?

# Installation

`ember-auth` distribution files lives in `lib/`.

Coffeescript files are in `src/`.

# Pre-req

`ember-auth` expects your server to provide an API interface with
token authentication support.
*An example of setting up token authentication in rails and devise will
be posted soon in the wiki.*

The token authentication API should expose two end points:
* a `POST` route for token creation
  * This should return the authentication token and the user model ID
    upon successful token creation in a JSON hash.
* a `DELETE` route for token destruction

Your server should also use proper HTTP status codes, i.e. the 2xx series for
successful actions, and the 4xx series for errors. In particular, the following
will be relevant:
* `200 OK`:           general pages, or returning protected content upon
                      successful authentication
* `201 Created`:      suggested response code for successful token creation
* `400 Bad Request`:  suggested response code for wrong parameters when
                      consuming an API end point
* `401 Unauthorized`: trying to access protected content without proper
                      authentication
* `404 Not Found`:    ah, good old 404.

At present `ember-auth` only supports `DS.RESTAdapter`.

# Usage

## Config

Let's say your server exposes a token authentication interface as follows:
* `POST /users/sign_in` for token creation (sign in)
  * expects `email` and `password` as params
  * sample response: `{user_id: 1, auth_token: "jL3hbrhni82yxIHUD"}`
* `DELETE /users/sign_out` for token destruction (sign out)
  * expects `auth_token` as param
  * (no response requirement)

```coffeescript
Auth.Config.reopen
  tokenCreateUrl: '/users/sign_in'
  tokenDestroyUrl: '/users/sign_out'
  tokenKey: 'auth_token'
  idKey: 'user_id'
```

## Persistence adapter

Persistence adapter setup: you will use `Auth.RESTAdapeter`; it is an
extension of `DS.RESTAdapter`.

```coffeescript
App.Store = DS.Store.extend
  revision: 11 # or whatever suitable
  adapter: Auth.RESTAdapter.create()
```

## Sign in/out views and templates

### 1. Widget style

"Widget" style sign in/out forms, for example at the end of a navigation bar.
The distinctive characteristic is that they do not have dedicated routes;
and that they are contained in a small view area within the app.

Make a `view` and a `template` for the authorization form area:

```coffeescript
App.AuthView = Em.View.extend
  templateName: 'auth'
```

```html
<script type="text/x-handlebars" data-template-name="auth">
  {{#if Auth.authToken}}
    {{view App.SignOutView}}
  {{else}}
    {{view App.SignInView}}
  {{/if}}
</script>
```

Note the use of `Auth.authToken` as the condition. `ember-auth` will store
the authentication token here when "signed in", and it will set it to `null`
when "signed out".

The sign in form:

```coffeescript
App.SignInView = Em.View.extend
  templateName: 'sign_in'

  email:    null
  password: null

  submit: (evt, view) ->
    evt.preventDefault()
    evt.stopPropagation()
    Auth.signIn
      email:    @get 'email'
      password: @get 'password'
```

```html
<script type="text/x-handlebars" data-template-name="sign_in">
  <form>
    <label>Email</label>
    {{view Ember.TextField valueBinding="email" valueBinding="view.email"}}
    <label>Password</label>
    {{view Ember.TextField valueBinding="password" valueBinding="view.password"}}
    <button>Sign In</button>
  </form>
</script>
```

Here we use the `Auth.signIn` helper. It accepts a hash of params that will
be passed to the API call.

The sign out form:

```coffeescript
App.SignOutView = Em.View.extend
  templateName: 'sign_out'

  submit: (evt, view) ->
    evt.preventDefault()
    evt.stopPropagation()
    Auth.signOut()
```

```html
<script type="text/x-handlebars" data-template-name="sign_out">
  <form>
    <button>Sign Out</button>
  </form>
</script>
```

The `Auth.signOut` helper has the same signature as the `Auth.signIn` helper,
except that it will pass the authentication token as a parameter by default,
at the key specified in `Auth.Config.tokenKey`.
The [FAQ](https://github.com/heartsentwined/ember-auth/wiki/FAQ) has an
explanation of this default behavior.

### 2. Full page style

"Full page" style, e.g. a "sign_in" route where the sign in form itself
is the main content of the whole page.
The distinctive characteristic is that sign in / out pages have their own routes.

Make a `route`, a `controller` and a `template` for the sign in page:

```coffeescript
App.Router.map ->
  @route 'sign_in'
```

```coffeescript
App.SignInRoute = Ember.Route.extend()
```

```coffeescript
App.SignInController = Ember.ObjectController.extend
  email: null
  password: null

  signIn: ->
    Auth.signIn
      email:    @get 'email'
      password: @get 'password'
```

```html
<script type="text/x-handlebars" data-template-name="sign_in">
  <form>
    <label>Email</label>
    {{view Ember.TextField valueBinding="email" valueBinding="email"}}
    <label>Password</label>
    {{view Ember.TextField valueBinding="password" valueBinding="password"}}
    <button {{action "signIn"}}>Sign In</button>
  </form>
</script>
```

We register a `signIn` action to our Sign In button, and then use the
`Auth.signIn` helper to sign the user in. The `Auth.signIn` helper is
explained in the [Widget style section](#1-widget-style).

The sign out page:

```coffeescript
App.Router.map ->
  @route 'sign_out'
```

```coffeescript
App.SignOutRoute = Ember.Route.extend()
```

```coffeescript
App.SignOutController = Ember.ObjectController.extend
  signOut: ->
    Auth.signOut()
```

```html
<script type="text/x-handlebars" data-template-name="sign_in">
  <form>
    <button {{action "signOut"}}>Sign Out</button>
  </form>
</script>
```

Again, we register a `signOut` action on the button; the `Auth.signOut` helper
is explained in the [Widget style section](#1-widget-style).

## Authenticated-only routes

Authenticated-only routes setup: you will use `Auth.Route`; it is an
extension of `Ember.Route`.

The route `panel` - let's say, pointing to the user control panel - should
be a protected route:

```coffeescript
App.PanelRoute = Auth.Route.extend()
```

`Auth.Route` does nothing by default. It is there to provide a center place for
implement any `route`- and authentication-related logic:

```coffeescript
Auth.Route.reopen
  # do something
```

However, see Redirects section right below for built-in redirection support.

## Redirects

`ember-auth` provides five kinds of redirects to assist in building your UI.
All these require a `route` to redirect to, so they won't make sense if you
are using only the widget-style UI. ([Why?](https://github.com/heartsentwined/ember-auth/wiki/FAQ))

### Authenticated-only routes

You can have non-authenticated ("not signed in") users redirected to a
sign in route - let's say, named `sign_in` - when they visit an `Auth.Route`:

```coffeescript
Auth.Config.reopen
  signInRoute: 'sign_in'
  authRedirect: true
```

It is a good idea to make your sign out route authenticated-only with redirection:
non-authenticated users should not try to sign out; and in any case your
server API should reject sign out request from non-authenticated users anyway.

```coffeescript
App.SignOutRoute = Auth.Route.extend()
```

### Post- sign in redirect: fixed route

You can have the user redirected to a specified route - let's say, 'account' -
after signing in.

```coffeescript
Auth.Config.reopen
  signInRedirectFallbackRoute: 'account' # defaults to 'index'
```

You will need to modify your controller. Extend from `Auth.SignInController`,
and call its `registerRedirect` method from your sign in action.

```coffeescript
App.SignInController = Auth.SignInController.extend # changed here
  email: null
  password: null

  signIn: ->
    @registerRedirect() # and here
    Auth.signIn
      email:    @get 'email'
      password: @get 'password'
```

### Post- sign in redirect: smart mode

"Smart" redirect. After sign in, the user is redirected to:
* one's previous route, unless one comes from the `signInRoute`
* the fallback route otherwise

Let's say your sign in route is called `sign_in`, and you want the fallback
route to be `account`

```coffeescript
Auth.Config.reopen
  signInRoute: 'sign_in'
  smartSignInRedirect: true
  signInRedirectFallbackRoute: 'account' # defaults to 'index'
```

Same modification to `controller` as the
[post- sign in fixed route redirect](#post--sign-in-redirect-fixed-route).

### Post- sign out redirect: fixed route

You can have the user redirected to a specified route - let's say, 'home' -
after signing out.

```coffeescript
Auth.Config.reopen
  signOutRedirectFallbackRoute: 'home' # defaults to 'index'
```

You will need to modify your controller. Extend from `Auth.SignOutController`,
and call its `registerRedirect` method from your sign in action.

```coffeescript
App.LogOutController = Auth.SignOutController.extend # changed here
  signOut: ->
    @registerRedirect() # and here
    Auth.signOut()
```

### Post- sign out redirect: smart mode

This is rather awkward.
Do you really have a use case, where you will implement a logic that
auto-redirects the user to your sign out route in the first place,
such that "smart" redirecting the user back from one's previous route will
make sense? Anyway, here it is:

"Smart" redirect. After sign out, the user is redirected to:
* one's previous route, unless one comes from the `signOutRoute`
* the fallback route otherwise

Let's say your sign out route is called `sign_out`, and you want the fallback
route to be `home`

```coffeescript
Auth.Config.reopen
  signOutRoute: 'sign_out'
  smartSignOutRedirect: true
  signOutRedirectFallbackRoute: 'home' # defaults to 'index'
```

Same modification to `controller` as the
[post- sign out fixed route redirect](#post--sign-out-redirect-fixed-route).

## Further use cases

The source code at `src/auth.coffee` is a comprehensive list of public API
and helper methods; `src/config.coffee` contains an exhaustive list of
configurable options.

# Contributing

You are welcome! As usual:

1. Fork
2. Branch
3. Hack
4. Commit
5. Pull request

## Todo

* a full-blown rails + devise + ember-auth tutorial

# License

GPL 3.0
