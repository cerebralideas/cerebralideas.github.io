---
layout: post
title: 'Build a Protected Web App with React JS'
tags: [Tutorial, React, ForgeRock, OAuth, Authentication, Development]
featured_image: assets/images/posts/2021/forgerock+react.jpg
author: justin
---

Development tutorial with a sample React.js SPA and Node.js REST API.

## What is a protected web app?

A protected web app is simply an app that uses some type of access artifact to ensure a user’s identity and permissions for viewing or managing a protected resource. This “access artifact” can be a session cookie, or it can be an OAuth-based access token. The protected resource could be a bank account, social media posts, music library, et cetera.

A protected web app will be responsible for providing a user a method of acquiring this access artifact, using the artifact for requesting the protected resource from an API, and removing the access artifact upon request.

## What you will learn

React JS (commonly referred to as just React) is a very popular, JavaScript view-library produced by Facebook’s engineering organization. As with most libraries and frameworks, it recommends or requires particular patterns and conventions for development. Since ForgeRock doesn’t (as of this writing) provide a React version of our SDK for use in React projects, we want to present this how-to guide to help you through a basic implementation our core JavaScript SDK using the React patterns and conventions commonly prescribed by the community.

The following application features will be implemented within this guide using the ForgeRock JavaScript SDK:

1. Dynamic authentication form for login and registration
2. OAuth/OIDC token acquisition through the Authorization Code Flow with PKCE
3. Protected client-side routing
4. Protected resource requests to a protected REST API
5. Logout

### This is not a guide to how to build a React app

We will not focus on how React apps are architected/constructed as that is outside of the scope of this guide. It’s also worth noting that there are dozens of React-based frameworks and generators for building web applications (e.g. Next.js, Create React App, Gatsby). What is “best” is highly contextual to the product and user requirements for any given project.

To demonstrate how these features are added within a React application, we will be using a simplified, non-production-ready, to-do app that is already constructed. This to-do app represents a Single Page Application with nothing other than React Router added to “the stack” for help with handling routing and redirection. This is to keep it as “bare-bones” as possible for the purpose of this simplicity.

### Pre-existing knowledge needed for this guide

We are assuming you have a basic understanding of the following:

1. JavaScript and its ecosystem of modules
2. The command line interface (e.g. Terminal, Shell, Bash)
3. Functional components: we will be using the functional-style, React components
4. Hooks: local state and side-effects will be managed with this concept
5. Context API: global state will be managed with this concept
6. React Router: our choice of handling “pages” and redirection

### Using this guide

This guide is intended to be “hands on”, so we are providing the web app and resource server for you. You can find the repo on Github. All you’ll need is your own ForgeRock ID Cloud or Access Manager. If you don’t have access to either, and you are interested in ForgeRock’s Identity and Access Management platform, reach out to a representative today, and we’ll be happy to get you started.

Project requirements:

- Node 14 or higher along with npm
- An ability to generate security certificates
- Admin access to an instance of ForgeRock ID Cloud or Access Manager

#### Two ways of using this guide:

1. Follow along by building the app yourself: continue to the “Setup” section below
2. Just want to read through the instructions: skip to “Configure the SDK to your ForgeRock server”

## ForgeRock Setup

### Configure CORS (Cross-Origin Resource Sharing)

Due to the fact that pieces of our system will be running on different origins (scheme, domain and port), we will need to configure CORS in the ForgeRock server to allow our web app to make requests. The following values are required:

- **Allowed Origins**: `https://react.example.com:8443`
- **Allowed Methods**: `GET` `POST`
- **Allowed headers**: `accept-api-version` `authorization` `content-type` `x-requested-with`
- **Allow credentials**: enable

For more information on how to configure CORS, [visit our ForgeRock documentation for CORS on AM](https://backstage.forgerock.com/docs/am/7/security-guide/enable-cors-support.html) or [for CORS configuration on ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/tenants/configure-cors.html).

### Create simple login journey/tree

You will need a simple username and password login journey/tree for this guide. You’ll need one Page Node connected to the start of the journey/tree, and added inside of that node the Username Collector and Password Collector nodes. Then, add the Data Store Decision node and connect that to the Page Node. Finally, connect the `True` outcome of the decision node to the Success node, and add the `False` outcome to the Failure node.

![Image of username-password journey in ID Cloud](https://share.getcloudapp.com/bLuqDlZW)

For more information on how to build journeys/trees, [visit our ForgeRock documentation for tree configuration on AM](https://backstage.forgerock.com/docs/am/7/authentication-guide/about-authentication-trees.html) or [for journey configuration on ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/realms/journeys.html).

### Create two OAuth clients

You will need to create two OAuth clients configured in the ForgeRock server: one for the React app, and one for the REST API server. They each server a different purpose. The OAuth client for the React app (acting as a Relying Party) will provide our app with OAuth/OIDC tokens for requesting resources from our Resource Server (REST API). The Resource Server will then validate the user’s Access Token via its own OAuth client.

For more information on how to configure OAuth clients, [visit our ForgeRock documentation for OAuth 2.0 client configuration on AM](https://backstage.forgerock.com/docs/am/7/oauth2-guide/oauth2-register-client.html) or [for OAuth client/application configuration on ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/realms/applications.html).

#### Public OAuth client for React app

Create a public (for Single Page Application) OAuth client for the web app. Here are the relevant settings:

- **Client name/ID**: `WebOAuthClient`
- **Client type**: `Public`
- **Secret**: `<leave empty>`
- **Scopes**: `openid` `profile` `email`
- **Grant types**: `Authorization Code`
- **Implicit consent**: enabled
- **Redirection URLs/Sign-in URLs**: `https://react.example.com:8443/callback`
- **Token authentication endpoint method**: `none`

#### Confidential OAuth client for our resource server

Create a confidential (Node.js) OAuth client for the API server Below are the relevant settings:

- **Client name/ID**: `RestOAuthClient`
- **Client type**: `Confidential`
- **Secret**: `<alphanumeric string>` (treat it like a password)
- **Default scope**: `am-introspect-all-tokens`
- **Grant types**: `Authorization Code`
- **Token authentication endpoint method**: `client_secret_basic`

### Create a test user

Create a test user within your ForgeRock server. No special attributes or privileges, just a basic user in the realm you will be using. You’ll use this user when testing your login form.

For more information on how to build journeys/trees, [visit our ForgeRock documentation for identities/user management on ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/identities/manage-identities.html). If you are using ForgeRock’s AM, you will need to login as an administrator and then click on **Identities**. Now, click on **Add Identity**.

The **User ID** value should be a UUID, so just [go to this UUID v4 generator](https://www.uuidgenerator.net/version4) to copy and paste into the input. You will use this UUID as the username for logging in in the React app. Complete user creation by providing a password and email address.

## Local project setup

### Installing the project

First, clone the to-do app project to your local computer, `cd` (change directory) into the project folder, checkout the branch for this guide and install the needed dependencies:

```sh
git clone <repo>
cd <directory>
git checkout blog/build-protected-app/start
npm i
```

While that is installing the dependencies, let’s get your ForgeRock ID Cloud or Access Manager setup. Login into your ForgeRock server as an administrator.

NOTE: The branch that will be used in this guide will be an overly-simplified version of the sample app you see in the `main` Git branch. There’s also a branch that represents the completion of this guide, and if you are at anytime stuck, you can visit `blog/build-protected-app/complete` branch in Github. `<TODO: needs link>`

### Create your security certificates

Any apps that deal with authentication/authorization require HTTPS (secure protocol) which means security (SSL/TLS) certificates are necessary. For local testing and development, it’s common to generate your own self-signed certificates.

You’re free to use any method to do this, but if you need assistance in generating your own certs, [`mkcert` can help simplify the process](https://github.com/FiloSottile/mkcert). After following `mkcert`’s installation guide and simple example of creating certs, you should have two files: `example.com+5.pem` & `example.com+5-key.pem`. _Ensure these two files are in the root of this project._

> **Warning: Self-signed certificates or certificates not from an industry-recognized, certificate authority (CA) should never be used in production.**

### Set local domain aliasing

Edit your `hosts` file to point special domains to your `localhost` . If you’re on a Mac, the file can be found here: `/etc/hosts`. If you are on Windows, it will be found here (or similar): `System32\Drivers\etc\hosts`. Open the file as an administrator or as `sudo`, and add the following line:

```
127.0.0.1 react.example.com api.example.com
```

This allows for easier operation of the app locally and keeps it consistent with how it will operate on the Web.

### Create the `.env` file

This file provides all the important configuration values to the system. We provide an example file found in the root of the project called `.env.example`. Copy this file and rename it to `.env` within the root of your project. Add your relevant values to this new file, so everything is gets built and properly configured.

Here’s a hypothetical example:

```
AM_URL=https://auth.forgerock.com/am
APP_URL=https://react.example.com:8443
API_URL=https://api.example.com:9443
DEBUGGER_OFF=false
DEVELOPMENT=true
JOURNEY_LOGIN=Login
JOURNEY_REGISTER=Registration
SEC_KEY_FILE=key-file.pem
SEC_CERT_FILE=cert-file.pem
REALM_PATH=alpha
REST_OAUTH_CLIENT=RestOAuthClient
REST_OAUTH_SECRET=6MWE8hs46k68g9s7fHOJd2LEfv
WEB_OAUTH_CLIENT=WebOAuthClient
```

Most of the values should be self-explanatory, but below are some explanations for the potentially less clear:

- `DEBUGGER_OFF`: when `true`, this disables the `debugger` statements in the app. These can be enabled for learning the integration points at runtime of the app in your browser. When the browser’s developer tools are open, the app will pause at each integration point. Code comments will be placed above them explaining its use.
- `DEVELOPMENT`: when `true`, this provides better debugging for development purposes
- `JOURNEY_LOGIN`: the name of your simple login journey/tree (created above)
- `JOURNEY_REGISTER`: the name of your registration journey/tree (not needed for this guide)
- `REALM_PATH`: realm of your ForgeRock server (likely `root`, `alpha` or `beta`)

## Build and run the project

Now that everything is setup, let’s build and run the to-do app project. First, you’ll need two terminal windows open:

```sh
# In one terminal window
$ npm run watch

# In the other terminal window
$ npm start
```

The `watch` command will watch for file changes and rebuild the project when necessary. The `start` command will run the two web servers needed for the client app and resource server. Neither of these commands need to be restarted, unless you do one of the following:

- Modify your `.env` file (restart `watch` command)
- Modify the `webpack.config.js` file (restart `watch` command)
- Modify any file within the `/server` directory (restart `start` command)

## View the app in browser

Open your preferred browser and use the following URL: `https://react.example.com:8443`. A home page should be rendered explaining the purpose of the project. It should look like the below (it may be dark if you have a dark theme set in your OS):

![Image of web app in browser](https://share.getcloudapp.com/Apu98pkn)

If there are errors, here are a few tips:

- Check all the URLs and ports (the web app uses 8443) of the system
- Are all apps running on HTTPS and were the related security certificates created for the used domains
- Make sure your `hosts` file has the right aliases
- Are the dependencies installed? You should have a `node_modules` directory in your project
- `404` errors related to resources? Check the terminal that has the `watch` command running

## Configure the SDK to your ForgeRock server

Now that we have our environment and servers setup, let’s jump into the application! Within your IDE of choice, navigate to the `/client` directory. This is where you will spend the rest of your time.

First, open up the `index.js` file and import the `Config` object from the JavaScript SDK and call the `set` method on this `Config` object:

```js
import { Config } from '@forgerock/javascript-sdk';
// rest of imports ...

Config.set();
// rest of application code ...
```

This will always be the initial interaction with the SDK and is frequently done at the application’s top-level file. To setup the SDK to communicate with the appropriate ForgeRock server and its journeys/trees, OAuth clients and realms, pass in a configuration object with the appropriate values.

Using the ForgeRock server that you setup earlier in the article, set those values in this configuration object. To reduce redundancy and risk of human error, we are going to pull most of these values out of our `.env` variables which are mapped to constants within our `constants.js` file:

```js
Config.set({
  clientId: WEB_OAUTH_CLIENT,
  redirectUri: `${APP_URL}/callback`,
  scope: 'openid profile email',
  serverConfig: {
    baseUrl: AM_URL,
    timeout: '5000',
  },
  realmPath: REALM_PATH,
  tree: JOURNEY_LOGIN,
});
```

Go back to your browser and refresh the home page. There should be no change to what’s rendered, and no errors in the console. Now that the app is configured to our ForgeRock server, let’s wire up the simple login page!

## Building a login page

First, let’s review how the application renders the home page:

`index.js` > `router.js` > `view/home.js` > inline code + components (`components/`)

For the login page, the same pattern applies, except the fact that it will having much less inline code:

`index.js` > `router.js` > `view/login.js` > `components/journey/form.js`

Go ahead and navigate to the app’s login page within your browser. You should see a loading spinner and message that’s persistent. To ensure the right form inputs are rendered to the user, the right data needs to be retrieved from the ForgeRock server. That will be the first task.

Since most of the action is taking place in `components/journey/form.js`, open it and add the JavaScript SDK import:

```js
import { FRAuth } from '@forgerock/javascript-sdk';
```

`FRAuth` will be the first object used as it provides the necessary methods for authenticating a user against the Login **Journey**/**Tree**. Our first use of `FRAuth` will be the use of its `start` method. Since this is an asynchronous method, we will use the `useEffect` hook, so we’ll need to add that to our import from React. This `start` method will return data that we will want to save as local state, so let’s also use the `useState` hook.

```js
import React, { useEffect } from 'react';
// other imports ...

// .. Login function
const [step, setStep] = useState(null);
console.log(step);

useEffect(() => {
  const nextStep = FRAuth.start();
  setStep(nextStep);
}, []);

// return statement ...
```

NOTE: We are passing in an empty array as the second argument into `useEffect`. This essentially instructs the `useEffect` to only run once after the component mounts, and that’s exactly what we want.

The above code will result in two logs to your console: one that will be `null` and the other should be an object with a few properties. The one property to focus on is the `callbacks` property. This property contains the instructions for what needs to be rendered to the user for input collection. Currently, the `Login` component doesn’t react (pun intended) to this new data. To change that, a few things need to be done within this `Login` function:

1. Import our a few, existing, form-input components
2. Create a function to map over the `callbacks` and assign the appropriate component to render
3. Within the JSX, use a ternary to conditionally render the `callbacks` depending their presence
4. If we do have callbacks, use our mapping function to render them

Import the `Password`, `Text` and `Unknown` components.

```js
// other imports ...

import Password from './password';
import Text from './text';
import Unknown from './unknown';
```

Now that we have the needed components imported, within the `Login` function body, create the function that we will use to map these components to their appropriate callback.

```js
// within Login's function body ...
function mapCallbacksToComponents(cb, idx) {
  switch (cb.getType()) {
    case 'NameCallback':
      return <Text callback={cb} inputName={name} key='username' />;
    case 'PasswordCallback':
      return <Password callback={cb} inputName={name} key='password' />;
    default:
      // If current callback is not supported, render a warning message
      return <Unknown callback={cb} key={`unknown-${idx}`} />;
  }
}
```

Finally, let’s check for the presence of the `step.callbacks`, and if they exist, map over them with the function from above. Replace the single return of `<Loading message="Checking your session ..." />` with the below:

```js
if (!step) {
  return <Loading message='Checking your session ...' />;
} else if (step?.callbacks?.length) {
  return (
    <form className='cstm_form'>
      {step.callbacks.map(mapCallbacksToComponents)}
      <button className='btn btn-primary w-100' type='submit'>
        Sign In
      </button>
    </form>
  );
} else {
  return <Error message={renderStep.payload.message} />;
}
```

Refresh the page, and you should now have a dynamic form that reacts to the callbacks returned from our initial call to ForgeRock.

## Handling the login form submission

Of course, a form that can’t submit anything is not very usable, so the next step will be handling the submission. First, let’s edit the current form element, `<form className="cstm_form">`, and add an `onSubmit` handler with a simple function.

```js
<form
  className="cstm_form"
  onSubmit={(event) => {
    event.preventDefault();
    const nextStep = FRAuth.next(step);
    setStep(nextStep);
  }}
>
```

You may be asking, “How did the user’s input values get added to the `step` object?” Let’s take a look at the component for rendering the username input. Open up the `Text` component: `components/journey/text.js`. Notice how special methods are being used on the callback object. These are provided as convenience methods for getting and setting data.

```js
export default function Text({ callback, inputName }) {
  const [state] = useContext(AppContext);
  const existingValue = callback.getInputValue();

  function setValue(event) {
    callback.setInputValue(event.target.value);
  }

  return (
    <div className={`cstm_form-floating form-floating mb-3`}>
      <input
        className={`cstm_form-control form-control ${validationClass} bg-transparent ${state.theme.textClass} ${state.theme.borderClass}`}
        defaultValue={existingValue}
        id={inputName}
        name={inputName}
        onChange={setValue}
        placeholder={textInputLabel}
      />
      <label htmlFor={inputName}>{textInputLabel}</label>
    </div>
  );
}
```

The two important items to focus on is the `callback.getInputValue()` and the `callback.setInputValue()`. The `getInputValue` is used to display any existing value that may be provided by ForgeRock, and the `setInputValue` is used to set the user’s input on the callback while they are typing (`onChange`). The reason this works, even though we are not communicating this value to the `Form` component, is due to us passing the `callback` to the components by “reference” (not by “value”). So, any mutation of the callback object within the `Text` (or `Password`) component is also contained within the `step` object in `Form`.

NOTE: You may be thinking, “That’s not very idiomatic React! Shared, mutable state is bad.” And, yes, you are correct, but we are taking advantage of this to keep everything simple (and this guide from being too long), so I hope you can excuse the pattern.

Each type of callback type has its own collection of methods for getting and setting data in addition to a base set methods that the SDK automatically adds to response from the ForgeRock server. For more information about these callback methods, [see our API documentation](https://sdks.forgerock.com/javascript/api-reference-core/index.html), or [the source code in Github](https://github.com/ForgeRock/forgerock-javascript-sdk/tree/master/src/fr-auth/callbacks), for more details.

That that the form is rendering and submitting, we need to add an additional condition to our `Form` component. This condition will handle the success result of the authentication journey.

```js
if (!step) {
  // Loading ...
} else if (step.type === 'LoginSuccess') {
  return <Alert message="Success! You're logged in." type='success' />;
} else if (step.type === 'Step') {
  // Form rendering
} else {
  // Error message
}
```

Once the success condition is handled, you can return back to the browser, refresh the page and login with your test user created in the Setup section above. You should see your “Success!” alert message rendered. Congratulations, you are now able to authenticate users!

## Continuing to the OAuth 2.0 flow

You could stop at the above and consider the user authenticated. After successful login, the session has been created and a session cookie has been written to the browser. This would be session-based authentication, and is viable when your system (apps and services) can rely on cookies as the access artifact to validate. But, [there are limitations to cookies (and these limitations will increase)](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/), so many decide to add an additional step to their authentication process, and that’s the “OAuth” or “OIDC flow”.

The goal of this flow is to attain tokens that can be used for access related purposes, replacing the need to rely on cookies being shared with the system. The two common tokens are the Access Token and the ID Token. The Access Token is what we will focus on in this post. With that said, import a new object from the ForgeRock SDK for requesting these tokens.

```
import { FRAuth, TokenManager } from '@forgerock/javascript-sdk';
```

Only an authenticated user that has a valid session can successfully request OAuth/OIDC tokens, so we have to make sure we make this token request after we get a `’LoginSuccess’` back from the authentication journey. We also have to consider that this is an asynchronous call to the ForgeRock server. There are multiple ways to handle this, but we’ll use the `useEffect` and `useState`.

Go back to `components/journey/form.js` and add a `useState` to the top of the function body to create a simple boolean flag of the user’s authentication state.

```js
// ... const [step, setStep] = useState(null);
const [isAuthenticated, setAuthentication] = useState(false);
```

Now, add a new `useEffect` hook to allow us to work with another asynchronous request, but unlike our first `useEffect`, we want this one to be dependent on the state of `isAuthenticated`. To do this, add `isAuthenticated` to the array passed in as the second argument.

```js
useEffect(() => {
  async function oauthFlow() {
    const tokens = await TokenManager.getTokens();
    console.log(tokens);
  }
  if (isAuthenticated) {
    oauthFlow();
  }
}, [isAuthenticated]);
```

Finally, we need to conditionally set this authentication flag when we have a success response from our authentication journey. In your form’s `onSubmit` handler add a simple conditional and set the flag to `true`.

```js
// event.preventDefault();
// const nextStep = FRAuth.next(step);
if (nextStep.type === 'LoginSuccess') {
  setAuthentication(true);
}
```

Once the changes are made, return back to your browser and remove all cookies created from any previous logins. When you refresh the page, you want to make sure the login form is rendered (if success message is displayed make sure “third-party cookies” are also removed).

Login with your test user, and you should get a success message like you did before, but now check your browser’s console log. You should see an additional entry of an object that contains your `idToken` and `accessToken`. Don’t worry about storing these tokens. The SDK handles that for you, and if you’re curious, they are in `localStorage`.

And with that, you have completed a full login and OAuth/OIDC flow.

## Using the Access Token

Now that the user’s identity is validated and an access token is attained, let’s now review what can be done with this information. First, collect basic user information, so the user can be addressed by name and email.

### Requesting user information

The SDK provides a convenience method for calling the `/userinfo` endpoint, a standard OAuth endpoint for requesting details about the current user. Within the `form.js` file, add the `UserManager` object to our ForgeRock SDK import statement.

```js
import import { FRAuth, TokenManager, UserManager } from '@forgerock/javascript-sdk';
```

On this new object, the `getCurrentUser` method will request the user data, and also validate if the existing access token is valid. Just after the `TokenManager.getTokens` method call, within the `oauthFlow` function from above, add this new method.

```js
// const tokens = await TokenManager.getTokens();
// console.log(tokens);
const user = await UserManager.getCurrentUser();
console.log(user);
```

If the access token is valid, the user information will be logged to the console just after the tokens. The remaining tasks before we move on from the `form.js` file is setting a small portion of this state to the global store in order for a whole application to use it. Add the remaining imports for setting the state and redirecting back to the home page: `useContext`, `AppContext` and `useHistory`.

```js
import React, { useEffect, useContext } from 'react';
import { useHistory } from 'react-router-dom';

import { AppContext } from '../../global-state';
```

At the top of the `Form` function body, use the `useContext` method to get the app’s global state and methods and call `useHistory` to get the `history` object.

```js
const [state, methods] = useContext(AppContext);
const history = useHistory();
```

Now, use these methods after the `UserManager.getCurrentUser` method call to set the new user information to the global state along and redirect to the home page.

```js
// const user = await UserManager.getCurrentUser();
// console.log(user);

methods.setUser(user.name);
methods.setEmail(user.email);
methods.setAuthentication(true);

history.push('/');
```

Once the above is written, revisit the browser, clear out all cookies, storage and cache, and log in with you test user. If you landed on the home page and the logs in the console show tokens and user data, you have successfully used the access token for retrieving use data. You may also notice that the home page looks a bit different. That’s due to the app “reacting” to the global state that we set just before redirecting.

### Reacting to the presence of the access token

Ensuring your app provides good UX, it’s important to have a recognizable, authenticated experience. This makes it clear to them that they are logged in. We do that by setting the global state, but there are time when the user may close the app and reopen it or refresh the browser. Since we’ve lost that state, we can easily verify their authentication status by checking the presence of stored tokens.

This can be done with the `TokenStorage.get` method. Adding this to the `index.js` file will provide what we need to rehydrate our state. First, import `TokenStorage` into the file and add it within the `initAndHydrate` function along with grabbing some of the user data the global state module adds to `localStorage`. Second, add these values to the `useGlobalStateMgmt` function call.

```js
import { Config, TokenStorage } from '@forgerock/javascript-sdk';

// (async function initAndHydrate() {
const isAuthenticated = !!(await TokenStorage.get());
const email = window.sessionStorage.getItem('sdk_email');
const username = window.sessionStorage.getItem('sdk_username');

// function Init() {
const stateMgmt = useGlobalStateMgmt({
  email,
  isAuthenticated,
  prefersDarkTheme,
  username,
});
```

Since we have a global state API we can use throughout our app, different components can pull this state in and use it to conditionally render one set of UI elements versus another. Navigation elements and the displaying of profile data are good examples of such conditional rendering. Examples of this can be found by reviewing `components/layout/header.js` and `views/home.js`.

### Validating the access token

The presence of the access token can be a good _hint_ for authentication, but it doesn’t mean the token is actually valid. Tokens can expire or be revoked on the server-side.

To ensure the token is still indeed valid, the use of the same `getCurrentUser` can be used. This is optional, depending on your product requirements, but you can protect routes with a token validation check before rendering portions of your application. This can prevent partial rendering of the application to then be ejected out due to invalid token, which can be a jarring user experience.

To validate a token for protecting a route, open the `routes.js` file, import the `ProtectedRoute` module and replace the regular `<Route path="todos">` with the new `ProtectedRoute` wrapper.

```js
import { ProtectedRoute } from './utilities/route';

// <Route path="/register">
//   <Register />
// </Route>
<ProtectedRoute path='/todos'>
  <Header />
  <Todos />
  <Footer />
</ProtectedRoute>;
```

Let’s take a look at what this wrapper does. Open `utilities/route.js` file. Let’s focus just the `validateAccessToken` function within the `useEffect` function. Currently, it’s just checking for the existence of the tokens with `TokenStorage.get`, which may be fine for some situations. We can optionally call the `UserManager.getCurrentUser` method to be sure that we not only have tokens, but that they are valid.

To do this, import `UserManager` into the file, and then replace `TokenStorage.get` with `UserManager.getCurrentUser`.

```js
import { UserManager } from '@forgerock/javascript-sdk';

// try {
await UserManager.getCurrentUser();
```

In the above code, we are reusing the `getCurrentUser` method to validate our token. If it succeeds, we can be sure our token is valid and call `setValid` to `'valid'` . If it fails, we know it is not valid and call `setValid` to `'invalid'`. We set that outcome with our `setValid` state method and our routing knows exactly where to redirect the user.

### Request protected resources with an access token

To make resource requests to an access-token-protected endpoint, we have an `HttpClient` module that provides a simple wrapper around the native `fetch` method of the browser. When you call the `request` method, it will retrieve the user’s access token, and attach it as a Bearer Token to the request as an `Authorization` header. This is what the resource server will use to to make its own request to the ForgeRock server to validate the user’s access token.

Open `utilities/request.js` and import the `HttpClient` from the ForgeRock SDK. Then, replace the native fetch method with the `HttpClient.request` method.

```js
import { HttpClient } from '@forgerock/javascript-sdk';

//   try {
const response = await HttpClient.request({
  url: `${API_URL}/${resource}`,
  init: {
    body: data && JSON.stringify(data),
    headers: {
      'Content-Type': 'application/json',
    },
    method: method,
  },
});
```

It’s worth noting that the `init` object in the above maps directly the `init` options object seen in the [official `Request` documentation in the Mozilla Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/Request/Request). The interface of the response from the request will also map directly to [the official `Response` object seen in the Mozilla Web Doc](https://developer.mozilla.org/en-US/docs/Web/API/Response).

Now that the user can login, request access tokens and access the page for the protected resources, the todos, revisit the browser and clear out all cookies, storage and cache. Keeping the developer tools open and on the network tab, log in with you test user. Once you have been redirected to the home page, click on the “Todos” item in the nav. There’s a lot of activity that will happen, but one of them will be the network call to `https://api.example.com:9443/todos`. Click on that network request and view the request headers. Notice the `Authorization` header with the bearer token. That’s the `HttpClient` in action.

## Handle logout request

Of course, you can’t have a protected app without providing the ability to log out. Luckily, this is a fairly easy task.

Open up the `views/logout.js` file and import `FRUser` from the ForgeRock SDK, `useEffect` and `useContext` from React, `useHistory` from React Router, and import `AppContext` from the global state module.

```js
import { FRUser } from '@forgerock/javascript-sdk';
import React, { useContext, useEffect } from 'react';
import { useHistory } from 'react-router-dom';

import { AppContext } from '../global-state';
```

Since this requires a network request, write call `useEffect` and pass in a callback function with the following functionality:

```js
useEffect(() => {
  async function logout() {
    try {
      await FRUser.logout();

      setAuthentication(false);
      setEmail('');
      setUser('');

      history.push('/');
    } catch (error) {
      console.error(error);
    }
  }
  logout();
}, []);
```

Since we only want to call this method once, after the component mounts, we will pass in an empty array as a second argument for `useEffect`. After `FRUser.logout` completes, we just empty or falsify the global state to clean up and redirect back to the home page.

Once all the above is complete, return to your browser, empty the cache, storage and cache, and reload the page. You should now be able to log the test user in, navigate to the todos page, add and edit some todos, and logout. Congratulations, you just built a ForgeRock protected app with React JS.
