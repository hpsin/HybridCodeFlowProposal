
# Hybrid Application authorization code flow

## Overview

An application development pattern we've seen is one of combined front- and back-end authentication.  When the implicit flow was a recommended and feasible solution, applications would use the authorization code flow to authenticate the back=end, and then use the implicit flow to authenticate the front-end silently.  This resulted in a single auth prompt for users, a desired user experience.  

As privacy-conscious features in browsers block the use of 3rd party cookies, this pattern is no longer supported, and popular opinion on the implicit flow is turning.  Applications could do two separate code flows to authenticate the front- and back-end separately, but this set of redirects is unappealing for users. Instead, we propose an extension to the confidential client authorization code flow to allow the back-end to request a new authorization code suitable for redemption on the front-end.  

## Assumptions inherent in this proposal

1. The Microsoft application model allows application registrations to function as both public and confidential clients, on a per-redirect-URI model. This allows e.g. an email client to have a confidential client website, a public client native app, and an API with various scopes registered. While this document requires the client IDs of the front- and back-end apps to match, other platforms could authorize code redemption based on a family of client IDs.
1. Microsoft's implementation of the browser-bound authorization code flow returns time-bound refresh tokens to the single-page app. This pattern allows the SPA to retrieve multiple tokens without interaction.  Application models that rely on fetching a new code per-token may consider extending this flow to allow repeated code requests using the confidential client refresh token to reduce interactive redirects to the login page.
1. This document assumes that the STS and RP must exist on separate domains, so that 3rd-party cookie usage cannot be eliminated by reducing the number of domains in use.

## 1 - Authorize

Confidential client `authorize` call per standard OAuth 2.0 code flow specification.  There are no requirements around the `scope` requested. 

## 2 - Server-Side Token Call

The server backing the web application recieves the code and redeems it using the authorization code flow in conjunction with client authentication per OpenID Connect, section 9 (`client_secret_post`,`private_key_jwt`,`client_secret_basic`). Native applications per [RFC 8252](https://tools.ietf.org/html/rfc8252) cannot use this flow. 

### Request

```http
    POST /token 
    Host: https://sts.example.com
    Content-Type: application/x-www-form-urlencoded

    grant_type=authorization_code
    &client_id=2d4d11a2-f814-46a7-890a-274a72a7309e
    &code=AwABAAAAvPM1KaPlrEqd...
    &redirect_uri=https%3A%2F%2FRelyingParty.example%2Ftoken
    &return_public_code=1
```

| Parameter | Description |
|----------:|:------------|
| `grant_type` | Must be `authorization_code` for the authorization code flow. |
| `client_id` | Application ID. |
| `code` | The authorization_code acquired during the Authorize call. |
| `redirect_uri` | A `Confidential` redirect_uri registered on the client application. |
| `return_public_code` | Flag to indicate that a public code is required. |

### Success Response

```http
    HTTP/1.1 200 OK
    -- snip header fields --

    {
        "access_token": "eyJ0eXAi...",
        "token_type": "Bearer",
        "expires_in": 3600,
        "scope": "https://api.service.example/data.read",
        "refresh_token": "AwABAAAAvPM1KaPl...",
        "public_code": "Asdfgjdfgdfib..."
        "id_token": "eyJ0eXAi...",
    }
```

| Parameter | Description |
|----------:|:------------|
| `access_token` | The requested access token. |
| `token_type` | Indicates the token type value. |
| `expires_in` | How long the access token is valid (in seconds). |
| `scope` | The permissions associated with the returned access token. |
| `refresh_token` | An OAuth 2.0 refresh token. |
| `public_code` | An OAuth 2.0 authorization code to be redeemed for client-side tokens by the user-agent. Authorization code will be redeemable only by the same application. |
| `id_token` | If the Authorize call in 1 used the `openid` scope, an ID token is returned pursuant OpenID Connect. |

### Error Response

Per OAuth 2.0 specification.

## 3 - Client-Side Token Call

The `public_code` field included in the initial token response is included in the initial rendering of the front-end application.  The front-end then redeems that code via XHR.  
### Request

```http
    POST /token 
    Host: https://sts.example.com
    Content-Type: application/x-www-form-urlencoded
    Origin: https://relyingparty.example

    grant_type=authorization_code
    &client_id=2d4d11a2-f814-46a7-890a-274a72a7309e
    &code=Asdfgjdfgdfib...
```

| Parameter | Description |
|----------:|:------------|
| `grant_type` | Must be authorization_code for the authorization code flow. |
| `client_id` | Application ID. |
| `code` | The authorization_code acquired during the server-side `token` call. |

Remark: no `redirect_uri` needs to be provided for this request, as the client-side authorization code is obtained as part of a server-side redemption of a previous code. If, however, a `redirect_uri` is provided the server will perform a consistency check to ensure that it is of a type allowed to redeem codes as a public client.

### Success Response

```http
    HTTP/1.1 200 OK
    -- snip header fields --
    Access-Control-Allow-Origin: [reflected origin header]
    Access-Control-Allow-Credentials: true
    Access-Control-Allow-Methods: POST, OPTIONS

    {
        "access_token": "eyJ0eXAi...",
        "token_type": "Bearer",
        "expires_in": 3600,
        "scope": "https://api.service.example/data.read",
        "refresh_token": "AwABAAAAvPM1KaPl...",
        "id_token": "eyJ0eXAi...",
    }
```

| Parameter | Description |
|----------:|:------------|
| `access_token` | The requested access token. |
| `token_type` | Indicates the token type value. |
| `expires_in` | How long the access token is valid (in seconds). |
| `scope` | The permissions associated with the returned access token. |
| `refresh_token` | An OAuth 2.0 refresh token.|
| `id_token` | If the Authorize call in 1 used the `openid` scope, an ID token is returned pursuant OpenID Connect. |

### Error Response

Per OAuth 2.0 specification.
