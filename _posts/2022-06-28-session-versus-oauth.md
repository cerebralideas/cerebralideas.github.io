---
layout: post
title: "Session-Based Versus OAuth-Based Access"
tags: [Architecture, Authentication, Session, OAuth, ForgeRock]
author: justin
featured_image: /assets/images/posts/2022/hand-holding-coin-by-zsun-fi-unsplash.webp
---

There are many factors in designing a protected system, but one of the most important decisions that need to be made lies in the choice of the Access Artifact that your system will use to represent a verified entity. The two most common Access Artifacts are the Session Token and the OAuth Access Token. Although both artifacts _can_ be used together within a system, we will mostly discuss these artifacts as an either-or for the sake of simplicity.

Before we dive into the differences between these two artifacts, let’s discuss some of their similarities:

1. Both are string values (ForgeRock often uses [JWT or JWT-like formats](https://jwt.io/))
2. Both are passed around the system via HTTP headers (though, different headers)
3. Both have lifetimes and lifecycles
4. Both have a flow that has to be completed in order to acquire the token
5. Both can be acquired, validated and removed via REST calls to the ForgeRock platform

With that said, let’s dive into the details of each artifact, help differentiate the two and their respective use-cases.

## Session-Based Access

A “session” within a system represents a temporal verification of an identity within the system. The most basic form of this is through the username and password (aka. user credentials) being given to the system. The system validates the credentials (along with other optional factors) and provides a Session Token as proof of this verification.

### Session Token

The Session Token is acquired after the successful completion of an authentication flow. [Within ForgeRock, this can be an authentication or registration journey](https://backstage.forgerock.com/docs/idcloud/latest/realms/journeys.html); it can be a simple journey request the user’s username and password, or a complex journey of credentials, push notifications and/or device profiling to ensure the integrity of the information.

It’s important to think of a Session Token as an artifact of authentication (aka. identification). That is, the session token itself represents _who_ the user is. This is in contrast to an artifact that represents _what_ the user can do. This is an important distinction to remember and will come up again when we discuss OAuth.

> Generally, a session token itself is quite binary: Is this a recognized and valid identity within our system or not?

Of course, ForgeRock and other IdPs can provide very powerful tools that go beyond such a binary concept. With ForgeRock, we provide a concept such as [Authorization Policies](https://backstage.forgerock.com/docs/am/7/authorization-guide/what-is-authz-decision.html), but this addition is beyond the scope of this article.

_**NOTE**: A Session Token requires network requests to the Identity Provider for validation and additional data. The token itself cannot be used independently, which is an important limitation of the artifact._

### Cookies

This Session Token is sent to the browser as a `Set-Cookie` header, which instructs the browser to store the token as a cookie.

```txt
==> REQUEST TO AUTHENTICATE WITH IDP
POST https://example.com/auth/session

Headers
origin: https://dashboard.example.com

Request Body
{ username: "...", password: "..." }

<== RESPONSE FROM IDP
200 OK

Headers
set-cookie: session=<session-token>; Domain=.example.com; HTTPOnly; Secure;
```

```txt
==> REQUEST TO PROTECTED RESOURCES
GET https://api.example.com/photos

Headers
cookie: session=<session-token>; Domain=.example.com; HTTPOnly; Secure;
origin: https://dashboard.example.com

<== RESPONSE FROM RESOURCE SERVER
200 OK

Response Body
{ ... }
```

Cookies are a decades-old [mechanism that allows the persistence of simple data on the browser](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies). How cookies are stored, managed, and attached to network requests are all “built-in” to the browser. If the cookie has the attribute `HTTPOnly` (which ForgeRock enables by default, along with the `Secure` attribute), it means the cookie will be inaccessible to the JavaScript runtime for security/privacy. The end result is the browser will manage 100% of the cookie’s lifecycle, and therefore the Session Token, on your behalf. This has its pros and cons.

_**NOTE**: For native mobile applications, the cookie is not a “native” mechanism, so the token is handled and stored manually within secure storage._

It’s important to understand the pros and cons of cookies within browsers. When a cookie is stored in the browser, it is intrinsically linked to the domain (technically hostname) of the _server_. If the client (web application) is running on the same hostname of the server, then the cookie is “first-party”. If the client and server are on different hostnames, the cookie is considered “third-party”.

> Third-party cookies, and how they are treated by browsers, is a very complex and critical topic to understand. For more information, read about [Apple Safari’s anti-tracking](https://www.theverge.com/2020/3/24/21192830/apple-safari-intelligent-tracking-privacy-full-third-party-cookie-blocking), [Firefox's Total Cookie Protection](https://forgerock.slack.com/archives/C020T976DL6/p1655936639673749), and [Google's eventual blocking of 3rd party cookies](https://www.theverge.com/2021/6/24/22547339/google-chrome-cookiepocalypse-delayed-2023).

For your protected, resource servers to also receive this Session Cookie, they too must be on the same hostname or a “suffix” of the hostname. For example: `.example.com` is a suffix of `auth.example.com` and `photos.example.com`. If the servers are not on the same hostname or a suffix, the cookie will not be sent. This means the design of your system and its hostnames are a very important factor.

### Embedded Login

The use of Session Tokens generally requires that authentication happens locally within the app. This is in contrast to Centralized Login, which means the user is redirected to a separate web application that’s dedicated to authenticating a user. This is an important factor for native mobile applications. If the user is shifted to a browser to authenticate, the Session Token will be contained within the browser and will not be shared with the native application (due to browser security protocols).

## OAuth-Based Access

OAuth is a complex specification that provides details regarding how a user acquires an Access Artifact known as the Access Token. It’s worth noting that when the term OAuth is used herein, we are referring to the [OAuth 2.0 specification](https://oauth.net/2/).

OAuth was originally designed to allow third-party applications limited, granular access to a user’s account and/or resources. Because the application was typically third-party, providing a session token would generally provide complete and total access to the user’s account/resources, which would not be acceptable. Therefore, the Access Token was designed to be used in place of a Session Token.

> Although OAuth was originally designed for third-party use, it is not limited to it.

There are different types of OAuth flows that are used for acquiring an Access Token. In this article, we will focus on the [Authorization Code Flow](https://oauth.net/2/grant-types/authorization-code/) but will keep it quite simple for brevity. What the user typically does is get redirected from the client application (aka Relying Party) to the Identity Provider. The user authenticates into the system and is then presented with a consent page. This consent page is asking the user for explicit consent to share certain aspects of their account with the Relying Party (or “client application”). If consent is given, the Identity Provider generates the Access Token and sends it to the Relying Party.

_**NOTE**: First-party applications (the Relying Parties and the Identity Provider are of the same company) can have what’s considered “implied consent” and therefore this explicit consent is not a requirement._

Once the application has the Access Token, it sends it anytime it needs to access protected resources, just like the Session Token. But, let’s inspect how the Access Token is acquired and determine the differences from a Session Token.

The below diagram is a very simplified Authorization Code Flow

```txt
==> REDIRECT TO IDP
GET https://example.com/oauth/authorize?client_id=...

<== REDIRECT BACK TO RELYING PARTY
GET https://dashboard.example.com/authorize/success?code=...&state=...

==> REQUEST FOR TOKENS FROM IDP
POST https://example.com/oauth/access_token

Request Body
{ code: ... }

<== Response FROM IDP
200 OK

Response Body
{ access_token: ... }
```

```txt
==> REQUEST TO PROTECTED RESOURCE SERVER
GET https://api.example.com/photos

Headers
authorization: Bearer <access_token>
origin:        https://dashboard.example.com

<== RESPONSE FROM RESOURCE SERVER
200 OK

Response Body
{ ... }
```

### Access Token

Before we get into the technical details of the Access Token, let’s first cover the intentions. Unlike the Session Token, the Access Token is an artifact representing Authorization (aka. permissions). Essentially, it represents _what_ the user can do, more than _who_ the user is. This absence of identification is intentional and is one of the reasons for the creation of [Open ID Connect](https://openid.net/connect/), but that too is beyond the scope of this article.

To represent _what_ the user can do, we have the idea of “scopes”. Scopes are arbitrary values that are intended to represent resources and actions that are allowed if the token is presented to the system. Examples of scopes can be `photos.read` or `profile_email`. Since this is inherent to the token, that means the Access Token has a granularity by default. Let’s look at how this information is embedded into the token itself.

It's common for Access Tokens to use the [JWT standard for structuring embedded data and encoding it](https://jwt.io/introduction) for transport. Example of JWT Access Token:

```txt
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

When the string is decoded, you get a few things:

1. Header
2. Payload
3. Signature

The payload is where you will find information regarding the user, the identity provider, and the scopes allowed by the token. The signature allows applications to independently verify the token’s validity, expiration and information. This means tokens can be validated and used independent of the Identity Provider. This can mean less “coupling” and higher performance due to reduced network calls.

### Token Lifecycle

After the client completes the OAuth flow, the Identity Provider passes the token in the response body from a REST endpoint call. Being that the token is just a property on a JSON response, it’s up to the client to store and manage the token appropriately. There are no native mechanisms on which the client (Relying Party) can rely, this is in stark contrast to the browser cookie.

This means the client is responsible for storing, renewing, removing and sending (token lifecycle) the Access Token. Luckily, we have the ForgeRock SDKs, so this token lifecycle is handled for you automatically.

### Embedded Login and Centralized Login

With the flexibility of Access Tokens, there’s more freedom to how the user is authenticated in the system. Either type, embedded or centralized, will work with Access Tokens.

## Which Is Better?

As with many things in life, the real answer is “it depends”. There is no right or wrong answer here, just what fits your system better given the constraints of your product requirements.

If your system is entirely first party and all under the same domain/hostname, a session-based access system may fit your system perfectly fine. If, on the other hand, you have a diverse system of third-parties and multiple domain names, OAuth-based access may be the better solution.

If we take what we discussed above, we can extrapolate the following about Session Tokens/Cookies:

- They have built-in mechanisms that “just work” with web applications
- They are very secure transport mechanisms because of the attributes `HTTPOnly` and `Secure`
- They are highly coupled to the hostname of the Identity Provider
- They have no built-in mechanism for granularity
- If leaked, they have potentially more damage potential
- They require network requests to the Identity Provider to validate the token
- Generally requires Embedded Login

Here’s what we can extrapolate about Access Tokens:

- They have no built-in mechanisms for storing and managing with any platform
- They require more security considerations because storage is up to the client
- They provide more freedom in diverse systems and are not constrained to hostnames
- They have built-in granularity for authorization or permission to resources and actions
- They can be used and validated independently of the Identity Provider
- If leaked, due to the constrained nature of Access Tokens, they can have limited damage potential
- Allows for either Embedded or Centralized Login

With that said, if you’re still unsure, speak with a ForgeRock representative or someone from your preferred Identity Provider.

> Access Management is hard, and getting it right means a lot. Don't underestimate the importance of this decision.
