---
layout: post
title: "Build Advanced Token Security in a JavaScript SPA"
tags: [Tutorial, React, ForgeRock, Security, OAuth, Authentication]
featured_image: /assets/images/posts/2020/safe-by-jason-dent-unsplash.jpg
author: justin
---

Advanced development tutorial for implementing Token Vault plugin with the ForgeRock JavaScript SDK.

## First, Why Advanced Token Security?

In JavaScript Single Page Applications (or SPA), OAuth/OIDC Tokens (referred to from here-on as "tokens") are typically stored via the browser's [Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API): `localStorage` or `sessionStorage`. The security mechanism the browser uses to ensure data doesn't leak out to unintended actors is through the "[Same-Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)". In short, only JavaScript running on the exact same origin (scheme, domain and port) can access the stored data.

For example:

If a SPA running on `https://auth.example.com/login` stores data, JavaScript running on the following would be able to access it:

- `https://auth.example.com/users` (origins match, regardless of path)
- `https://auth.example.com?status=abc` (origins match, regardless or query parameters)

The following would NOT be able to access the data:

- `http://auth.example.com` (origin includes scheme, `http` v. `https`)
- `https://auth.examples.com` (origin includes domain; notice the plurality)
- `https://example.com` (origin includes sub-domains)
- `https://auth.example.com:8000` (origin includes port)

_For the majority of web applications, this security model can be sufficient_.

In most JavaScript applications, the code running on the app's origin can usually be trusted, so the browser's Web Storage API is sufficient, as long as good security practices are in place. But, in applications that are high-value targets; apps that are required to run untrusted, third-party code; or apps that have elevated scrutiny due to regulatory or compliance needs, the Same-Origin Policy may not be enough to protect the stored tokens.

Examples may be government agencies, financial organizations, or those that store sensitive data, like medical records. The web applications of these entities may have enough inherent risk that offsets the complexity of a more advanced token security implementation.

There are two solutions that can increase token security:

1. Backend for Front-End
2. Origin Isolation

## The Backend for Front-End Pattern

One solution that is quite common is to avoid storing high-value secrets within the browser in the first place. This can be done with a dedicated [Backend for Front-End, or BFF](https://samnewman.io/patterns/architectural/bff/) (yeah, it's a silly initialism). This is increasingly becoming a common pattern for apps made with common meta-frameworks, like Next, Nuxt, SvelteKit, etc. The server component of these frameworks can store the tokens on the front-end's behalf using in-memory stores or just writing the token into an HTTPOnly cookie, and requests that require authorization can be proxied through the accompanying server where the tokens are accessible. Therefore, no token needs to be stored on the front-end, eliminating the risk.

You can read more about the arguments in favor of, and against, this design in the [Hacker News discussion on the blog post titled "SPAs are Dead"](https://news.ycombinator.com/item?id=26728596). If, on the other hand, you are not in the position to develop, deploy and maintain a full backend, you have an alternative choice for securing your tokens: Origin Isolation.

## Origin Isolation

Origin Isolation is a concept that introduces an alternative to the BFF, and provides a more advanced mechanism for token security. The concept is to store tokens in a _different_ and dedicated origin related to the main application. To do this, two web servers are needed to respond to two different origins: one is your main app, and the second is an "iframed" app dedicated to managing tokens. You can [read more about the details of this in ForgeRock's patent](https://patents.justia.com/patent/11689528).

This particular design means that if your main application gets compromised, the malicious code running in your main app's origin still has no access to the tokens stored in the alternative origin. You'll still need a Web Server of some kind to serve the files necessary to handle requests to this alternative origin, but being that only static files are served, the options are much simpler and lightweight.

This solution will be the focus of this rest of this tutorial.

## What is Token Vault?

The Token Vault is a codified implementation of Origin Isolation that [you can read more about in our docs](https://backstage.forgerock.com/docs/sdks/latest/javascript/token-vault.html). It is a plugin available to our customers that use the ForgeRock JavaScript SDK to enable OAuth/OIDC token request and management. These apps can remain largely unmodified to take advantage of Token Vault.

It's equally important to note that even though your main app doesn't need much modification, additional build and server requirements are necessary, which introduces complexity and added maintenance to your system. It is recommended to only introduce Token Vault to an app if heightened security measures are a requirement of your system.

## What You Will Learn

We will use an existing React JS, to-do sample application similar to what [we built in a previous tutorial](https://backstage.forgerock.com/docs/sdks/latest/blog/build-protected-reactjs-web-app.html) as a starting point for implementing Token Vault. This will represent a "realistic", web application that has an existing implementation of ForgeRock's JavaScript SDK. Unlike the previous tutorial, we'll start with a fully working app, and focus on just adding Token Vault.

This tutorial is going to focus on OAuth/OIDC authorization and token management, so authentication related concerns like login and registration journeys are going to be handled outside the app. This is referred to as Centralized Login or the Authorization Code Flow (with PKCE).

### This is Not a Guide on How to Build a React App

How React apps are architected or constructed is outside the scope of this guide. It’s also worth noting that there are many React-based frameworks and generators for building web applications, such as Next.js, Remix, and Gatsby. What is "best" is highly contextual to the product and user requirements for any given project.

To demonstrate our Token Vault integration, we will be using a simplified, non-production-ready, to-do app. This to-do app represents a Single Page Application (SPA) with nothing more than React Router added to "the stack" for routing and redirection.

### Using This Guide

This is a "hands-on" guide. We are providing the web app and resource server for you. You can find the repo on GitHub to follow along. All you’ll need is your own ForgeRock Identity Cloud or Access Manager. If you don’t have access to either, and you are interested in ForgeRock’s Identity and Access Management platform, reach out to a representative today, and we’ll be happy to get you started.

Two ways of using this guide

1. Follow along by building portions of the app yourself: continue by ensuring you can [meet the requirements](https://backstage.forgerock.com/docs/sdks/latest/blog/build-protected-reactjs-web-app.html#react-requirements) below.
2. Just curious about the code implementation details: skip to [Implementing the Token Vault](https://backstage.forgerock.com/docs/sdks/latest/blog/build-protected-reactjs-web-app.html#implementing-the-token-vault).

## Requirements

### Knowledge Requirements

1. JavaScript and the npm ecosystem of modules
2. The command line interface (such as Shell or Bash)
3. Core Git commands (including `clone` and `checkout`)
4. React and basic React conventions
5. Context API: global state will be managed with this concept
6. Build systems/bundlers and development servers: We will use basic Vite configuration and commands

### Technical Requirements

- Admin access to an instance of ForgeRock Identity Cloud or Access Manager (AM)
- Node.js >= 16 & npm >= 8 (please check your version with `node -v` and `npm -v`)

## ForgeRock Setup

NOTE: If you've already completed the previous tutorial for React JS or Angular, then you may already have most of this setup within your ForgeRock server. We'll call out the newly introduced data points to ensure you don't miss the configuration.

### Step 1. Configure CORS (Cross-Origin Resource Sharing)

Due to the fact that pieces of our system will be running on different origins (scheme, domain and port), we will need to configure CORS in the ForgeRock server to allow our web app to make requests. Use the following values:

- **Allowed Origins**: `http://localhost:5173` `http://localhost:5175`
- **Allowed Methods**: `GET` `POST`
- **Allowed headers**: `accept-api-version` `authorization` `content-type` `x-requested-with`
- **Allow credentials**: enable

_New: Rather than domain aliases and self-signed certificates, we will use `localhost` as that is a trusted domain by default. The main application will be on the `5173` port, and the proxy will be on the 5175 port. Because the proxy will also make calls to the AM server, its origin will need to be allowed as well._

Note: Service Workers are not compatible with self-signed certificates, regardless of tool or creation method. Certificates from a system-trusted, valid Certificate Authority (CA) are required or direct use of `localhost`. Self-signed certificates will lead to a `fetch` error similar to the following:

```text
Failed to register a ServiceWorker for scope ('https://example.com:8000/') with script ('https://example.com:8000/sw.js'):
An unknown error occurred when fetching the script
```

{% include image-wide.html imageurl="/assets/images/posts/2021/react-todo-app-screenshots/cors-configuration.png" title="Example CORS from ForgeRock ID Cloud" caption="Screenshot of CORS configuration within ID Cloud" %}

For more information about CORS configuration, [visit our ForgeRock documentation for CORS on AM](https://backstage.forgerock.com/docs/am/7/security-guide/enable-cors-support.html) or [for CORS configuration on ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/tenants/configure-cors.html).

### Step 2. Create Two OAuth Clients

Within the ForgeRock server, create two OAuth clients: one for the React web app, and one for the Node.js resource server.

Why two? It's conventional to have one OAuth client per app in the system. For this case, a public OAuth client for the React app will provide our app with OAuth/OIDC tokens. The Node.js server will validate the user's Access Token shared via the React app using its own confidential OAuth client.

#### Public OAuth Client Settings

- **Client name/ID**: `CentralLoginOAuthClient`
- **Client type**: `Public`
- **Secret**: `<leave empty>`
- **Scopes**: `openid` `profile` `email`
- **Grant types**: `Authorization Code` `Refresh Token`
- **Implicit consent**: enabled
- **Redirection URLs/Sign-in URLs**: `http://localhost:5173/login`
- Response types: `code` `id_token` `refresh_token`
- **Token authentication endpoint method**: `none`

_New: The client name and redirection/sign-in URL has changed from previous tutorial as well as the Refresh Token grant and response type._

#### Confidential OAuth Client Settings

- **Client name/ID**: `RestOAuthClient`
- **Client type**: `Confidential`
- **Secret**: `<alphanumeric string>` (treat it like a password)
- **Default scope**: `am-introspect-all-tokens`
- **Grant types**: `Authorization Code`
- **Token authentication endpoint method**: `client_secret_basic`

{% include image-wide.html imageurl="/assets/images/posts/2021/react-todo-app-screenshots/oauth-client-configuration.png" title="Example OAuth client from ForgeRock ID Cloud" caption="Screenshot of an OAuth client configuration in ID Cloud" %}

For more information about configuring OAuth clients, [visit our ForgeRock documentation for OAuth 2.0 client configuration on AM](https://backstage.forgerock.com/docs/am/7/oauth2-guide/oauth2-register-client.html) or [for OAuth client/application configuration on ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/realms/applications.html).

### Step 3. Create a Test User

Create a simple test user (identity) in your ForgeRock server within the realm you will be using.

If using ForgeRock ID Cloud, [follow our ForgeRock documentation for creating a user in ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/identities/manage-identities.html#create_a_user_profile).

If you are using ForgeRock’s AM, click on **Identities** item in the left navigation. Use the following instructions to create a user:

1. Click on **Add Identity** to view the create user form
2. In a separate browser tab, [visit this UUID v4 generator copy the UUID](https://www.uuidgenerator.net/version4)
3. Switch back to AM and paste the UUID into the input
4. Provide a password and email address

**Note: You will use this UUID as the username for logging into the React app.**

## Local Project Setup

### Step 1. Installing the Project

First, clone the [JavaScript SDK project](https://github.com/ForgeRock/forgerock-javascript-sdk) to your local computer, `cd` (change directory) into the project folder, checkout the branch for this guide, and install the needed dependencies:

```bash
git clone https://github.com/cerebrl/forgerock-sample-web-react-ts
cd forgerock-sample-web-react-ts
git checkout blog/token-vault-tutorial/start
npm i
```

NOTE: There’s also a branch that represents the completion of this guide. If you get stuck, you can visit `blog/token-vault-tutorial/complete` branch in Github.

### Step 2. Create the `.env` File

First, open the `.env.example` file in the root directory. Copy this file and name it `.env`. Add your relevant values to this new file as it will provide all the important configuration settings to your applications.

Here’s a hypothetical example:

```properties
# .env

# System settings
VITE_AM_URL=https://auth.example.com/am/ # Needs to be your ForeRock server
VITE_APP_URL=http://localhost:5173
VITE_API_URL=http://localhost:5174
VITE_PROXY_URL=http://localhost:5175 # This will be our Token Vault Proxy URL

# AM settings
VITE_AM_JOURNEY_LOGIN=Login # Not used with Centralized Login
VITE_AM_JOURNEY_REGISTER=Registration # Not used with Centralized Login
VITE_AM_TIMEOUT=50000
VITE_AM_REALM_PATH=alpha
VITE_AM_WEB_OAUTH_CLIENT=CentralLoginOAuthClient
VITE_AM_WEB_OAUTH_SCOPE=openid email profile

# AM settings for your API (todos) server
# (does not need VITE prefix)
AM_REST_OAUTH_CLIENT=RestOAuthClient
AM_REST_OAUTH_SECRET=6MWE8hs46k68g9s7fHOJd2LEfv # Don't use; this is just an example
```

We are using Vite for our client apps' build system and development server, so by using the `VITE_` prefix, Vite will automatically include these environment variables in our source code. Here are descriptions for some of the values:

- `VITE_AM_URL`: this should be your ForgeRock server and the URL almost always end with `/am/`
- `VITE_APP_URL`, `VITE_API_URL` & `VITE_PROXY_URL`: these will be the URLs you use for your locally running apps
- `VITE_AM_REALM_PATH`: the realm of your ForgeRock server (likely `alpha` if using Identity Cloud, or `root` if using standalone AM)
- `VITE_REST_OAUTH_CLIENT & VITE_REST_OAUTH_SECRET`: this will be the OAuth client you configure in your ForgeRock server to support the REST API server

## Build and Run the Project

Now that everything is setup, build and run the to-do app project. Open two terminal windows and use the following commands at the root directory of the SDK repo:

```bash
# In one terminal window, start the React app
npm run dev:react

# In the other terminal window, start the Rest API
npm run dev:api
```

_Note: we use npm workspaces to manage our multiple sample apps, but understanding how it works is not relevant to this tutorial._

The `dev:react` command above uses Vite and will restart on any change to a dependent file. This will also apply to the `dev:proxy` we will build shortly. The `dev:api` command runs a basic Node server with no "watchers". This should not be relevant as you won't have to modify any of its code. If a change is made within the `todo-api-server` workspace, or an environment variable it relies on, a restart would be required.

## View the App in Browser

In a _different_ browser than the one you are using to administer the ForgeRock server, visit the following URL: `http://localhost:5173` (I typically use Edge for the app development and Chrome for the ForgeRock server administration). A home page should be rendered explaining the purpose of the project. It should look like the below (it may be dark if you have the dark theme/mode set in your OS):

{% include image-wide.html imageurl="/assets/images/posts/2021/react-todo-app-screenshots/home-page.png" title="To-do app home page" caption="Screenshot of the home page of the sample app" %}

If you encounter errors, here are a few tips:

- Visit `http://localhost:5174/health-check` in the same browser you use for the React app; ensure it responds with "OK"
- Check the terminal that has the `watch` command running for error output
- Ensure you are not logged into the ForgeRock server within the same browser as the sample app; logout from ForgeRock if you are and use a different browser

## Install Token Vault Module

Install the Token Vault npm module within the root of the app:

```sh
npm install @forgerock/token-vault
```

This npm module will be used throughout multiple applications in our project, so installing it at the root, rather than at the app or workspace level, is a benefit.

## Implement the Token Vault Proxy

### Step 1. Scaffold the Proxy

Next, we'll need to create a third application, the Token Vault Proxy. Follow this structure when creating the new directory and its files.

```diff-text
  root
  ├── todo-api-server/
  ├── todo-react-app/
+ └─┬ token-vault-proxy/
+   ├─┬ src/
+   │ └── index.ts
+   ├── index.html
+   ├── package.json
+   └── vite.config.ts
```

### Step 2. Add the npm Workspace

To ease some of the dependency management and script running, add this new "workspace" to our root `package.json`, and then add a new script to our `scripts`:

```diff-json
@@ package.json @@

@@ collapsed @@

  "scripts": {
    "clean": "git clean -fdX -e \"!.env\"",
    "build:react": "npm run build --workspace todo-react-app",
+   "build:proxy": "npm run build --workspace token-vault-proxy",
    "build:server": "npm run build --workspace todo-api-server",
    "dev:react": "npm run dev --workspace todo-react-app",
+   "dev:proxy": "npm run dev --workspace token-vault-proxy",
    "dev:server": "npm run dev --workspace todo-api-server",
    "lint": "eslint --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "serve:react": "npm run serve --workspace todo-react-app",
+   "serve:proxy": "npm run serve --workspace token-vault-proxy",
    "serve:server": "npm run serve --workspace todo-api-server"
  },

@@ collapsed @@

  "workspaces": [
    "todo-api-server"
    "todo-react-app",
+   "token-vault-proxy",
  ]
```

### Step 3. Setup the Supporting Files

Create a new directory at the root named `token-vault-proxy`, then create `package.json`.

```diff-json
@@ token-vault-proxy/package.json @@

+ {
+   "name": "token-vault-proxy",
+   "private": true,
+   "version": "1.0.0",
+   "description": "The proxy for Token Vault",
+   "main": "index.js",
+   "scripts": {
+     "dev": "vite",
+     "build": "vite build",
+     "serve": "vite preview --port 7443"
+   },
+   "dependencies": {},
+   "devDependencies": {},
+   "license": "MIT"
+ }
```

Now, create the Vite config file.

```diff-ts
@@ token-vault-proxy/vite.config.ts @@

+ import { defineConfig } from "vite";
+
+ // https://vitejs.dev/config/
+ export default defineConfig(() => {
+   return {
+     envDir: "../", // Points to the `.env` created in the root dir
+     root: process.cwd(),
+     server: {
+       port: 7443,
+       strictPort: true,
+     },
+   };
+ });
```

Lastly, create the `index.html` file. This file can be overly simple as all you need is the inclusion of the JavaScript file that will be our Proxy.

```diff-html
@@ token-vault-proxy/index.html @@

+ <!DOCTYPE html>
+ <html>
+   <p>Proxy is OK</p>
+   <script type="module" src="src/index.ts"></script>
+ </html>
```

If you're not familiar with how Vite works, seeing the `.ts` extension might look at bit weird in an HTML file. But don't worry, Vite uses this to find entry files and it will rewrite the actual `.js` reference for us.

### Step 4. Create & Configure the Proxy

Let's create and configure the Token Vault Proxy according to our system needs. First, create the `src` directory and the `index.ts` file within it.'

```diff-ts
@@ src/index.ts @@

+ import { proxy } from "@forgerock/token-vault";
+
+ // Initialize the token vault proxy
+ proxy({
+   app: {
+     origin: import.meta.env.VITE_TOKEN_VAULT_APP_ORIGIN,
+   },
+   forgerock: {
+     clientId: import.meta.env.VITE_AM_WEB_OAUTH_CLIENT,
+     scope: import.meta.env.VITE_AM_WEB_OAUTH_SCOPE,
+     serverConfig: {
+       baseUrl: import.meta.env.VITE_AM_URL,
+     },
+     realmPath: import.meta.env.VITE_AM_REALM_PATH,
+   },
+   proxy: {
+     urls: [`${import.meta.env.VITE_API_URL}/*`],
+   }
+ });
```

The configuration above represents the minimum needed to create the Proxy. We need to declare the app's origin, as that's the only source to which the Proxy will respond. We have the ForgeRock configuration, in order for the Proxy to effectively callout to the ForgeRock server for token lifecycle management. Finally, there's the Proxy's URL array that acts as an allow-list to ensure that only valid URLs are proxied with the appropriate tokens.

### Step 5. Build & Verify the Proxy

With everything setup, build the proxy app and verify it's being served correctly.

```sh
npm run dev:proxy
```

Once the script finishes its initial build and runs the server, you can now check the app and ensure its running. Go to `http://localhost:5175` in your browser. You should see an "Proxy is OK" printed to screen, and there should be no errors in the Console or Network tab of your browser's dev tools.

```
<screenshot here>
```

## Implement the Token Vault Interceptor

### Step 1. Create the Interceptor's Build Config

Since the Token Vault Interceptor is a Service Worker, it needs to be bundled separately from your main application code. To do this, write a _new_ Vite config file within the `todo-react-app` directory/workspace named `vite.interceptor.config.ts`. We do not recommend trying to use the same configuration file for both your app and Interceptor.

```diff-txt
  root
  ├── ...
  ├─┬ todo-react-app/
  │ ├── ...
  │ ├── vite.config.ts
+ │ └── vite.interceptor.config.ts
```

Now that you have the new Vite config for the Interceptor, import the `defineConfig` method and pass it the appropriate configuration.

```diff-ts
@@ todo-react-app/vite.interceptor.config.ts @@

+ import { defineConfig } from "vite";
+
+ // https://vitejs.dev/config/
+ export default defineConfig({
+   build: {
+     emptyOutDir: false,
+     rollupOptions: {
+       input: "src/interceptor/index.ts",
+       output: {
+         dir: "public", // Treating this like a static asset is important
+         entryFileNames: "interceptor.js",
+         format: "iife", // Very important for better browser support
+       },
+     },
+   },
+   envDir: "../", // Points to the `.env` created in the root dir
+ });
```

Notice above we provide the Interceptor source file as the input, and then explicitly tell Vite to bundle it as an IIFE ([Immediately Invoked Function Expression](https://developer.mozilla.org/en-US/docs/Glossary/IIFE)) and save the output to this app's `public` directory. This means the Interceptor will be available as a static asset at the root of our web server.

It is important to note that bundling it as an IIFE and configuring the output to the public directory is intentional and important. Bundling as an IIFE removes any module system from the file, which is vital to supporting all major browsers within the Service Worker context. Outputting it to the public directory like a static asset is also important. It allows the `scope` of the Service Worker to also be available at the root. You can [read more about Service Worker scopes on MDN](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register).

### Step 2. Create the New Interceptor File

Let's create the new interceptor file that is expected as the entry file to our new Vite config.

```diff-txt
  root
  ├─┬ todo-react-app
  │ ├─┬ src/
+ │ │ ├─┬ interceptor/
+ │ │ │ └── index.ts
  │ │ └── ...
  │ ├── ...
  │ └── vite.interceptor.config.ts
```

### Step 3. Import & Initialize the `interceptor` Module

Configure your Interceptor with the following variables from your `.env` file.

```diff-ts
@@ todo-react-app/src/interceptor/index.ts @@

+ import { interceptor } from "@forgerock/token-vault";
+
+ interceptor({
+   interceptor: {
+     urls: [`${import.meta.env.VITE_API_URL}/*`],
+   },
+   forgerock: {
+     serverConfig: {
+       baseUrl: import.meta.env.VITE_AM_URL,
+       timeout: import.meta.env.VITE_AM_TIMEOUT,
+     },
+     realmPath: import.meta.env.VITE_AM_REALM_PATH,
+   },
+ });
```

The above only covers the minimal configuration needed, but it's enough to get a basic Interceptor started. The `urls` array represents all the URLs you'd like the intercepted and proxied through the Token Vault Proxy in order for the Access Token to be added to the outbound request. This should only be for requesting your "protected resources".

The wildcard (`*`) can be used if you want a catch-all for endpoints of a certain origin or root path. Full glob patterns are _not_ supported, so a URL value can only end with `*`.

### Step 4. Build the Interceptor

Now that we have the dedicated Vite config and the Interceptor entry file created, add a dedicated build command to the `package.json` within the `todo-react-app` workspace.

```diff-json
@@ todo-react-app/package.json @@

@@ collapsed @@

    "scripts": {
-     "dev": "npm run build:interceptor && vite",
+     "dev": "npm run build:interceptor && vite",
-     "build": "npm run build:interceptor && vite build",
+     "build": "npm run build:interceptor && vite build",
+     "build:interceptor": "vite build -c ./vite.interceptor.config.ts",
      "serve": "vite preview --port 5173"
    },

@@ collapsed @@
```

It's worth noting that the Interceptor will only be rebuilt at the start of the command, and not rebuilt after any change thereafter as there's no "watch" command used here for the Interceptor itself. Once this portion of code is correctly setup, it should rarely change, so this should be fine. Your main app will still be rebuilt and "hot-reloading" will take place.

Run `npm run build:interceptor`, and you should be able to see the resulting `interceptor.js` built and placed into your `public` directory.

```diff-txt
  root
  ├─┬ todo-react-app
  │ ├─┬ public/
  │ │ ├── ...
+ │ │ ├── interceptor.js

```

### Step 5. Ensure `interceptor.js` is Accessible

Since we haven't implemented the Interceptor yet in the main app, we can't really test it. But, we can at least make sure the file is accessible in the browser as we expect. To do this, run the following command:

```sh
npm run dev:react
```

Once the command starts the server, using your browser, visit `http://localhost:5173/interceptor.js`. You should plainly see the fully built JavaScript file. Ensure it does not have any `import` statements and looks complete (it should contain more code than just the original source file you wrote above).

## Implement the Token Vault Client

Now that we have all the separate pieces set up, wire it all together with the Token Vault Client plugin.

### Step 1. Add HTML Element to `index.html`

When we initiate the Proxy, it will need a real DOM element to mount to. The easiest way to ensure we have a proper element, we'll just directly add it to our `index.html`.

```diff-html
@@ todo-react-app/index.html @@

@@ collapsed @@

    <body>
      <!-Root div for mounting React app -->
      <div id="root" class="cstm_root"></div>
+
+     <!-Root div for mounting Token Vault Proxy (iframe) -->
+     <div id="token-vault"></div>

      <!-Import React app -->
      <script type="module" src="/src/index.tsx"></script>
    </body>
  </html>
```

### Step 2. Import & Initialize the `client` Module

First, import the `client` module and remove the `TokenStorage` module from the SDK import. Second, call the `client` function with the below minimal configuration. This is how we "glue" the three entities together within your main app. This function returns an object that we use to register and instantiate each entity.

```diff-tsx
@@ todo-react-app/src/index.tsx @@

- import { Config, TokenStorage } from '@forgerock/javascript-sdk';
+ import { Config } from '@forgerock/javascript-sdk';
+ import { client } from "@forgerock/token-vault";
  import ReactDOM from 'react-dom/client';

@@ collapsed @@

+ const register = client({
+   app: {
+     origin: import.meta.env.TOKEN_VAULT_APP_ORIGIN,
+   },
+   interceptor: {
+     file: "/interceptor.js", // references public/interceptor.js
+   },
+   proxy: {
+     origin: import.meta.env.TOKEN_VAULT_PROXY_ORIGIN,
+   },
+ });

@@ collapsed @@
```

Remember, the `file` reference within the `interceptor` object needs to point to the _built_ Interceptor file (which will be located in `public` directory as a static file, but served from the root), not the source file itself.

This function ensures the app, Interceptor and Proxy are appropriately configured.

### Step 2. Register the Interceptor, Proxy & Token Store

Now that we've initialized and configured the client, we now register the Interceptor, the Proxy and the Token Vault Store.

```diff-tsx
@@ todo-react-app/src/index.tsx @@

@@ collapsed @@

+ // Register the Token Vault Interceptor
+ await register.interceptor();
+
+ // Register the Token Vault Proxy
+ await register.proxy(
+   // This must be a live DOM element; it cannot be a Virtual DOM element
+   // `token-vault` is the element added in Step 1 above to `todo-react-app/index.html`
+   document.getElementById("token-vault") as HTMLElement
+ );
+
+ // Register the Token Vault Store
+ const tokenStore = register.store();

@@ collapsed @@
```

Registering the Interceptor is what requests and registers the Service Worker. Calling `register.interceptor` returns [the `ServiceWorkerRegistration` object that can be used to unregister the Service Worker](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration) as well as other functions, if that's needed. We won't be implementing that in this tutorial.

Registering the Proxy constructs the `iframe` component and mounts it do the DOM element passed into the method. It's important to note that this must be a real, available DOM element; not a Virtual DOM element. This results in the Proxy being "registered" as a child frame, and therefore accessible, to your main app. Calling `register.proxy` also returns an optional reference to the DOM element of the `iframe` that can be used to manually destroy the element and the Proxy, if needed.

Finally, registering the Store will provide us the object that will be used to replace the default token store within the SDK. There are some additional convenience methods on this `store` object that we'll take advantage of later in the tutorial.

### Step 3. Replace the SDK's Default Token Store

Within the existing SDK configuration, pass the `tokenStore` object we created in the previous step to the `set` method to override the SDK's internal token store.

```diff-tsx
@@ todo-react-app/src/index.tsx @@

@@ collapsed @@

  // Configure the SDK
  Config.set({
    clientId: import.meta.env.WEB_OAUTH_CLIENT,
    redirectUri: import.meta.env.REDIRECT_URI,
    scope: import.meta.env.OAUTH_SCOPE,
    serverConfig: {
      baseUrl: import.meta.env.AM_URL,
      timeout: import.meta.env.TIMEOUT,
    },
    realmPath: import.meta.env.REALM_PATH,
+   tokenStore, // this references the Token Vault Store we created above
  });

@@ collapsed @@
```

This configures the SDK to use the Token Vault Store, which will be within the Proxy, and that it does need to internally manage the tokens.

### Step 4. Check for existing tokens

Currently in our application, we check for the existence of stored tokens to provide a hint if our user is authorized. Now that the main app doesn't have access to the tokens, we have to ask the Proxy if it has tokens. To do this, replace the SDK method of `TokenStorage.get` with the Proxy's `has` method.

```diff-tsx
@@ todo-react-app/src/index.tsx @@

@@ collapsed @@

   let isAuthenticated = false;
    try {
 -    isAuthenticated = !((await TokenStorage.get()) == null);
 +    isAuthenticated = !!(await tokenStore?.has())?.hasTokens;
    } catch (err) {
      console.error(`Error: token retrieval for hydration; ${err}`);
    }

@@ collapsed @@
```

Note that this doesn't return the tokens as that would violate the security of keeping them in another origin, but the Proxy will inform you of their existence. This is enough to hint to our UI that the user is likely authorized.

## Build and Run the Apps

At this point, all the necessary entities are setup, we will now run all the needed servers and test out our new application with Token Vault enabled. Open three different terminal windows all from within the root of this project. Enter each command in its own window:

```sh
npm run dev:react
```

```sh
npm run dev:api
```

```sh
npm run dev:proxy
```

Allow all the commands to complete the build and start the development servers. Then, visit `http://localhost:5173` in your browser of choice. The to-do application should look and behave no different from before.

Open the dev tools of your browser, and proceed to log into the app. You will be redirected to the ForgeRock login page, and then redirected back after successfully authenticating. You may notice some additional redirection within the React app itself; this is normal. Once you land on the home page, you should see the "logged in experience" with your username in the success alert.

To test whether Token Vault is successfully implemented, go to the Application or Storage tab of your dev tools and inspect the `localStorage` section. You should see two origins: `http://localhost:5173` (our main app) and `http://localhost:5175` (our Proxy). Your user's tokens should be stored under the Proxy's origin (port `5175`), not under the React app's origin (port `5173`). If you observe that behavior, then you have successfully implemented Token Vault. Congratulations, your tokens are now more securely stored!

If you don't see the tokens in the Proxy origin's `localStorage`, then follow the troubleshooting section below:

## Troubleshooting

### Getting failure in the Service Worker registration?

Make sure you're dedicated Vite config is correct, and check the actual file output for the `interceptor.js`. If the built file has ES Module syntax in it, or it looks incomplete, then that will cause the issue. Service Workers in some browsers, even the latest versions, don't support the same ES version or features as the main browser context.

### Tokens are not being saved under any origin

Open up your browser's dev tools, and ensure the following (enabling "Preserve Log" for both the Console and the Network tabs is very helpful):

1. Your Interceptor is running under your main app's origin
2. Do you have an `/access_token` request (it comes after the `/authorize` request and redirect)
3. Your Interceptor is intercepting the `/access_token` request (if it is, you should see two outgoing requests: one for main app, and one for proxy)
4. Your Proxy is running within the `iframe` and forwarding the request
5. There are no network errors
6. There are no console errors

### I'm getting a CORS failure

Make sure you have both origins listed in your ForgeRock CORS configuration. Additionally, it's best if you use the ForgeRock SDK template when creating a new CORS config in your ForgeRock server.

If both origins are listed, make sure you have no typos in the allowed methods (method, like `GET`, `POST` are case-sensitive) and headers (headers are NOT case-sensitive). Allowed Credentials also has to be enabled.
