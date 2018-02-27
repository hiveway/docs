Authorization
=============

Hiveway builds around the idea of being a server first, rather than a client itself. Similarly to how a XMPP chat server communicates with others and with its own clients, Hiveway takes care of federation to other networks, like other Hiveway or GNU Social instances. So Hiveway provides a REST API, and a 3rd-party app system for using it via OAuth2.

You can get a client ID and client secret required for OAuth [via an API end-point](API.md#apps).

From these two, you will need to acquire an access token. It is possible to do using your account's e-mail and password like this:

    curl -X POST -d "client_id=CLIENT_ID_HERE&client_secret=CLIENT_SECRET_HERE&grant_type=password&username=YOUR_EMAIL&password=YOUR_PASSWORD" -Ss https://hiveway.net/oauth/token

The `/oauth/token` path will attempt to login with the given credentials, and then retrieve the access token for the current user. If the login failed the response will be a 302 redirect to `/auth/sign_in`. Otherwise the response will be a JSON object containing the key `access_token`.

Use that token in any API requests by setting a header like this:

    curl --header "Authorization: Bearer ACCESS_TOKEN_HERE" -sS https://hiveway.net/api/v1/timelines/home

Please note that the password-based approach is not recommended especially if you're dealing with other user's accounts and not just your own. Usually you would use the authorization grant approach where you redirect the user to a web page on the original site where they can login and authorize the application and are then redirected back to your application with an access code.

OAuth
=====

We use the [Doorkeeper gem for OAuth](https://github.com/doorkeeper-gem/doorkeeper/wiki), so you can refer to their docs on specifics of the end-points.

The API is divided up into access scopes:

- `read`: Read data
- `write`: Post statuses and upload media for statuses
- `follow`: Follow, unfollow, block, unblock

Multiple scopes can be requested during the authorization phase with the `scope` query param (space-separate the scopes). If you do not specify a `scope` in your authorization request, the resulting access token will default to `read` access.

One possible way this might work for you:

1. Get `client_key` and `client_secret` from your local cache. If you don't have the two, you need to [register the application](https://github.com/hiveway/docs/blob/master/API/API.md#registering-an-application). Store `client_key` and `client_secret` in your local cache for next time. We actually don't need the `id` returned from this call.
1. Get the `refresh_token` from your local cache. If you have one, any access token you might have has probably expired anyway. Refresh by visiting `/oauth/token` using a POST request with the parameters `client_id=<client_key>` (yes, this is confusing!), `client_secret=<client_secret>`, `grant_type=refresh_token`, `refresh_token=<refresh_token>`. Save the `access_token` you get back in your local cache.
1. Get the `access_token` from your local cache. Try to connect. One simple way to do this is to [verify credentials](https://github.com/hiveway/docs/blob/master/API/API.md#getting-the-current-user). Whatever API call you make, add the HTTP header `Authorization: Bearer <access_token>`.
1. If you have neither refresh nor access token, tell the user to visit `/oauth/authorize` with parameters `scope=read write follow` (URL encode the space to `%20`, of course), `response_type=code`, `redirect_uri=urn:ietf:wg:oauth:2.0:oob` (assuming you don't want to redirect your users elsewhere), `client_id=<client_key>` (yes, this is confusing). The user clicks on the URL and gets shown a page asking them to authorize your app for reading, writing, and following (assuming this is the scope you requested, feel free to request less). If the user clicks on the right button, they get back a code. Store this code as your `refresh_token`. Go back to step #2.
