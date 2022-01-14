---
layout: post
title: 'Build a Protected React Native Mobile App: Part One – iOS'
tags: [Tutorial, React Native, ForgeRock, OAuth, Authentication]
featured_image: /assets/images/posts/2021/forgerock+react.jpg
author: justin
---

This is part one of a multi-part series covering the basics of developing a ForgeRock-protected, mobile app with React Native. This part focuses on developing the iOS bridge code along with a minimal React UI to authenticate a user.

## Table of contents

- [First, what is a protected mobile app?](#first-what-is-a-protected-mobile-app)
- [What you will learn](#what-you-will-learn)
  - [This is not a guide on how to build a React Native app](#this-is-not-a-guide-on-how-to-build-a-react-native-app)
  - [Using this guide](#using-this-guide)
- [Requirements](#requirements)
  - [Knowledge requirements](#knowledge-requirements)
  - [Technical requirements](#technical-requirements)
- [ForgeRock setup](#forgerock-setup)
  - [Step 1. Create a simple login journey/tree](#step-1-create-a-simple-login-journeytree)
  - [Step 2. Create an OAuth client](#step-2-create-an-oauth-client)
  - [Step 3. Create a test user](#step-3-create-a-test-user)
- [Local project setup](#local-project-setup)
  - [Step 1. Clone the project](#step-1-clone-the-project)
  - [Step 2. Install the project dependencies](#step-2-install-the-project-dependencies)
  - [Step 3. Create the `.env.js` file](#step-3-create-the-envjs-file)
- [Build and run the project](#build-and-run-the-project)
  - [If the app doesn't build](#if-the-app-doesnt-build)
- [Using Xcode, iOS Simulator and Safari dev tools](#using-xcode-ios-simulator-and-safari-dev-tools)
  - [Tips if the home screen doesn’t render](#tips-if-the-home-screen-doesnt-render)
- [Implement the iOS bridge code for authentication](#implement-the-ios-bridge-code-for-authentication)
  - [Step 1. Review the iOS related files](#step-1-review-the-ios-related-files)
  - [Step 2. Configure your `.plist` file](#step-2-configure-your-plist-file)
  - [Step 3. Write the `start` method](#step-3-write-the-start-method)
  - [Step 4. Write the `login` method](#step-4-write-the-login-method)
  - [Step 5. Write the `next` method](#step-5-write-the-next-method)
  - [Step 6. Write the `logout` bridge method](#step-6-write-the-logout-bridge-method)
- [Building a React Native form for simple login](#building-a-react-native-form-for-simple-login)
  - [Step 1. Initializing the SDK](#step-1-initializing-the-sdk)
  - [Step 2. Building the login view](#step-2-building-the-login-view)
  - [Step 3. Handling the login form submission](#step-3-handling-the-login-form-submission)
  - [Step 4. Requesting user info and redirecting to home screen](#step-4-requesting-user-info-and-redirecting-to-home-screen)
- [Adding logout functionality to our bridge and React Native code](#adding-logout-functionality-to-our-bridge-and-react-native-code)
  - [Handle logout request in React Native](#handle-logout-request-in-react-native)

## First, what is a protected mobile app?

A protected app (client or server) is simply an app that uses some type of access artifact to verify a user’s identity (authentication) and permissions (authorization) prior to giving access to a resource. This "access artifact" can be a session cookie, an access token, or an assertion, and is shared between the entities of a system.

Additionally, a protected mobile app (client) is responsible for providing a user the methods for acquiring, using, and removing this access artifact upon request. The focus of this guide is implementing these capabilities using the ForgeRock iOS SDK within a React Native app.

## What you will learn

"Bridge code" is a concept common to mobile app development within hybrid applications. This is when a portion of the mobile app is written in a language that is not native to the platform (Android and Java or iOS and Swift). React Native, a cross-platform, view-library from Facebook, requires this bridging code to provide the React (JavaScript) layer access to native (Swift in this case) APIs or dependencies. We also touch on some of the concepts and patterns popularized by the React library. Since ForgeRock doesn’t (as of this writing) provide a React Native version of our SDK, we present this how-to as a guide to basic development of "bridge code" for connecting our iOS SDK to the React Native layer.

This guide covers how to implement the following application features using version 3 of the ForgeRock iOS and JavaScript SDK:

1. Authentication through a simple journey/tree.
2. Requesting OAuth/OIDC tokens.
3. Requesting user information.
4. Logging a user out.

### This is not a guide on how to build a React Native app

How React Native apps are architected or constructed is outside of the scope of this guide. It’s also worth noting that there are many React Native libraries and frameworks for building mobile applications. What is "best" is highly contextual to the product and user requirements for any given project.

To simply demonstrate our SDK integration, we use a few libraries to simplify the development of the application. We chose [React Navigation as the navigation library](https://reactnavigation.org/) and [NativeBase as the UI library](https://nativebase.io/), but neither is featured heavily in this part of the multi-part guide.

### Using this guide

This is a "hands on" guide. We are providing the mobile app and resource server for you. You can [find the repo on Github](https://github.com/ForgeRock/forgerock-react-native-sample) to follow along. All you’ll need is your own [ForgeRock Identity Cloud](https://www.forgerock.com/platform/identity-cloud) or [Access Management](https://www.forgerock.com/platform/access-management). If you don’t have access to either, and you are interested in ForgeRock’s Identity and Access Management platform, [reach out to a representative today](https://www.forgerock.com/contact), and we’ll be happy to get you started.

#### Two ways of using this guide

1. Follow along by building portions of the app yourself: continue by ensuring you can [meet the requirements](#requirements) below
2. Just curious about the bridge code and React Native implementation details: skip to [Develop the iOS bridge code for the ForgeRock SDK](#implement-the-ios-bridge-code-for-authentication)

## Requirements

### Knowledge requirements

1. Xcode, the iOS Simulator and related tools.
2. Swift and CocoaPods.
3. JavaScript and the npm ecosystem of modules.
4. The command line interface (e.g. Terminal, Shell or Bash).
5. Core Git commands (e.g. `clone`, `checkout`).
6. Functional components: we use the functional-style, React components.
7. Hooks: local state and side-effects are managed with this concept.
8. Context API: global state is managed with this concept.

### Technical requirements

- Admin access to an instance of ForgeRock Identity Cloud or Access Management (referred to as AM).
- Xcode 12 or higher with the developer tools installed.
- Node.js 14 or higher & npm 7 or higher (please check your version with `node -v` and `npm -v`).
- A tool or service to generate a security certificate and key (self-signed is fine).

## ForgeRock setup

### Step 1. Create a simple login journey/tree

You need a simple username and password authentication journey/tree (referred to as "journey") for this guide.

1. Create a Page Node and connect it to the start of the journey.
2. Add the Username Collector and Password Collector nodes within the Page Node.
3. Add the Data Store Decision node and connect it to the Page Node.
4. Connect the `True` outcome of the decision node to the Success node.
5. Connect the `False` outcome of the decision node to the Failure node.

{% include image-wide.html imageurl="/assets/images/posts/2021/react-todo-app-screenshots/username-password-journey.png" title="Example journey from ForgeRock ID Cloud" caption="Screenshot of username-password journey in ID Cloud" %}

For more information about building journeys/trees, [visit our ForgeRock documentation for tree configuration on AM](https://backstage.forgerock.com/docs/am/7/authentication-guide/about-authentication-trees.html) or [for journey configuration on ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/realms/journeys.html).

### Step 2. Create an OAuth client

Within the ForgeRock server, create a public, "Native", OAuth client for the mobile client app with the following values:

- **Client name/ID**: `ReactNativeOAuthClient`
- **Client type**: `Public`
- **Secret**: `<leave empty>`
- **Scopes**: `openid` `profile` `email`
- **Grant types**: `Authorization Code` `Refresh Token`
- **Implicit consent**: enabled
- **Redirection URLs/Sign-in URLs**: `https://com.example.reactnative.todo/callback`
- **Token authentication endpoint method**: `none`

Each of the above is required, so double check you set each of the items correctly. If you don't, you'll likely get an "invalid OAuth client" error in Xcode logs.

{% include image-wide.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/oauth-client-configuration.png" title="Example OAuth client from ForgeRock ID Cloud" caption="Screenshot of an OAuth client configuration in ID Cloud" %}

For more information about configuring OAuth clients, [visit our ForgeRock documentation for OAuth 2.0 client configuration on AM](https://backstage.forgerock.com/docs/am/7/oauth2-guide/oauth2-register-client.html) or [for OAuth client/application configuration on ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/realms/applications.html).

### Step 3. Create a test user

Create a test user (identity) in your ForgeRock server within the realm you will be using.

If using ForgeRock ID Cloud, [follow our ForgeRock documentation for creating a user in ID Cloud](https://backstage.forgerock.com/docs/idcloud/latest/identities/manage-identities.html#create_a_user_profile).

Or, if you are using ForgeRock’s AM, click on **Identities** in the left navigation. Use the following instructions to create a user:

1. Click on **Add Identity** to view the create user form.
2. In a separate browser tab, [visit this UUID v4 generator copy the UUID](https://www.uuidgenerator.net/version4).
3. Switch back to AM and paste the UUID into the input.
4. Provide a password and email address.

**Note: You will use this UUID as the username for logging into the app.**

## Local project setup

### Step 1. Clone the project

First, clone the [React Native Sample project](https://github.com/cerebrl/forgerock-react-native-sample) to your local computer, `cd` (change directory) into the project folder, and `checkout` the branch for this guide:

```bash
git clone git@github.com:cerebrl/forgerock-react-native-sample.git
cd forgerock-react-native-sample
git checkout blog/build-protected-app/part-one/start
```

In addition, there’s a branch that represents the completion of this guide. If you get stuck, you can [visit `blog/build-protected-app/part-one/complete` branch in Github](https://github.com/cerebrl/forgerock-react-native-sample/tree/blog/build-protected-app%2Fpart-one%2Fcomplete) and use it as a reference.

NOTE: The branch used in this guide is an overly-simplified version of the sample app found in the `main` Git branch within the `reactnative-todo` directory. This "full version" is a to-do application that communicates with a protected REST API.

{% include image-caption.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-todos-screen.png" title="The to-do sample app" caption="Screenshot of the to-do screen from the full sample app" %}

### Step 2. Install the project dependencies

React Native requires two types of dependencies: JavaScript and Swift (CocoaPods) modules. First, let’s install the JavaScript dependencies. Within the project directory: `reactnative-todo/` (file and directory references are from this location), use the following command:

```bash
npm install
```

Once the above command finishes, `cd` into the `ios` directory and install the needed CocoaPods. When done, you can return to the project directory.

```bash
cd ios
pod install
cd ..
```

### Step 3. Create the `.env.js` file

Using the ForgeRock server settings from above, create the `.env.js` file from the `.env.js.template` file within the project. Add your relevant values to configure all the important server settings to your project. Not all variables will need values at this time.

A hypothetical example (required variables commented):

```js
/**
 * Avoid trailing slashes in the URL string values below
 */
const AM_URL = 'https://auth.forgerock.com/am'; // Required
const API_PORT = 9443; // Required; default port
const API_BASE_URL = 'http://localhost'; // Required; default domain
const DEBUGGER_OFF = true; // Required
const REALM_PATH = 'alpha'; // Required
```

Descriptions of relevant values:

- `AM_URL`: The URL that references AM itself (for Identity Cloud, the URL is likely be `https://<tenant-name>.forgeblocks.com/am`)
- `API_PORT` and `API_BASE_URL` just need to be "truthy" right now to avoid errors and will be used in a future part of this series
- `DEBUGGER_OFF`: when `true`, this disables the `debugger` statements in the JavaScript layer. These debugger statements are for learning the integration points at runtime in your browser. When the browser’s developer tools are open, the app pauses at each integration point. Code comments above each integration point explain its use.
- `REALM_PATH`: the realm of your ForgeRock server (likely `root`, `alpha` or `bravo`)

## Build and run the project

Now that everything is setup, build and run the to-do app project.

1. Open your Finder application and find the following file `ios/reactnativetodo.xcworkspace`.
2. Double click this file to open and load the project within Xcode.
3. Once Xcode is ready, select iPhone 11 or higher as the target for the device simulator on which to run the app.
4. Now, click the build/play button to build and run this application in the target simulator.

With everything up and running, you will need to rebuild the project with Xcode when you modify the bridge code (Swift files). But, when modifying the React Native code, it will use "hot module reloading" to automatically reflect the changes in the app without having to manually rebuild the project.

### If the app doesn't build

1. Make sure `libFRAuth.a` is added to your Target's "Frameworks, Libraries, and Embedded Content" under the "General" tab.
2. Make sure the Metro server is running; `npx react-native start` if you want to run it manually.
3. Bridge code has been altered, so be aware of name changes.
4. If you get `[!] CocoaPods could not find compatible versions for pod "FRAuth"`, run `pod repo update` then `pod install`.

## Using Xcode, iOS Simulator and Safari dev tools

We recommend the use of iPhone 11 or higher as the target for the iOS Simulator. When you first run the build command in Xcode (clicking the "play" button), it takes a while for the app to build, the OS to load, and app to launch within the Simulator. Once the app is launched, rebuilding it is much faster if the changes are not automatically "hot reloaded" when made in the React layer.

{% include image-caption.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-home-screen.png" title="To-do app home screen" caption="Screenshot of the home screen from the starter code" %}

_Note: only the home screen will render successfully at this moment. If you click on the Sign In button, it won’t be fully functional. This is **intended** as you will develop this functionality throughout this tutorial._

Once the app is built and running, you will have access to all the logs within Xcode's output console. Both the native and JavaScript logs display here. Because of this, there's quite a lot of output, so you may want to use it only when the Safari console does not provide enough information for debugging purposes.

{% include image-caption.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-xcode-login-response.png" title="Xcode log output" caption="Screenshot of Xcode console output" %}

For additional tooling, click "Device" within the top menu, and then select "Shake". This triggers the React Native dev tools, allowing you to reload the app, inspect the UI, as well as other actions.

_Note: Due to [a particular confusing bug within React Native](https://github.com/facebook/react-native/issues/28531), we do not recommend using the Chrome debugger, but recommend using Safari for debugging. To use Safari for debugging the React code, [follow the instructions found within the React Native docs](https://reactnative.dev/docs/debugging#safari-developer-tools)._

{% include image-wide.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-safari-devtools.png" title="Safari dev tools" caption="Screenshot of Safari dev tools with breakpoint in form component" %}

### Tips if the home screen doesn’t render

1. Restart the app (in Xcode) and Metro (in terminal).
2. Didn't work? Using Xcode, clean the build folder and rebuild/rerun the app.
3. If that doesn't work, remove the the following from the `reactnative-todo` directory: `node_modules`, `package-lock.json`, `ios/.Pods`, `ios/Podfile.lock` and then reinstall dependencies with `npm i` and within the `ios/` directory `pod install`
4. If you're still having issues, within the simulator, click the **Home** button and long press the React Todo application to .remove it. Then, restart from the project Xcode.
5. You can also use "Device" > "Erase All Content and Settings" if the problem persists.

## Implement the iOS bridge code for authentication

### Step 1. Review the iOS related files

We first need to review the files that allow for the "bridging" between our React Native project and our native SDKs. Within Xcode, navigate to the `ios/reactnativetodo` directory, and you will see a few important files:

- `reactnativetodo-Bridging-Header.h`: this the header file that exposes the React Native bridging module and the FRAuth module .into the Swift context
- `FRAuthSampleBridge.m`: The module file that defines the exported interfaces of our bridging code.
- `FRAuthSampleBridge.swift`: The main Swift bridging code that provides the callable methods for the React Native layer.
- `FRAuthSampleStructs.swift`: Provides the structs for the Swift bridging code.
- `FRAuthSampleHelpers.swift`: Provides the extensions to often used objects within the bridge code.
- `FRAuthConfig`: A `.plist` file that configures the iOS SDK to the appropriate ForgeRock server.

_Note: The remainder of the files within the workspace are automatically generated when you create a React Native project with the CLI command, so you can ignore them._

We provide the header file as is. The file's creation, naming and use requires very specific conventions that are outside the scope of this tutorial. You will not need to modify it.

### Step 2. Configure your `.plist` file

Within Xcode's directory/file list section (aka Project Navigator), find the `FRAuthConfig.plist.template` file in the `ios/reactnativetodos` directory and right click on it. Select "Open As" > "Source Code". Copy all of its contents.

Within the same directory, create a new `.plist` or Properties file called `FRAuthConfig.plist`. Opening this file into the same view with "Open As Source Code", paste the copied contents into the new file. Finally, add the relevant values for your ForgeRock server configuration for the application.

A hypothetical example (your values may vary):

```plist
<dict>
  <key>forgerock_cookie_name</key>
  <string>a1baeb394ea5300</string>
  <key>forgerock_oauth_client_id</key>
  <string>MobileOAuthClient</string>
  <key>forgerock_oauth_redirect_uri</key>
  <string>https://com.example.reactnative.todo/callback</string>
  <key>forgerock_oauth_scope</key>
  <string>openid profile email</string>
  <key>forgerock_oauth_threshold</key>
  <string>60</string>
  <key>forgerock_url</key>
  <string>https://auth.example.com/am</string>
  <key>forgerock_realm</key>
  <string>alpha</string>
  <key>forgerock_timeout</key>
  <string>60</string>
  <key>forgerock_auth_service_name</key>
  <string>Login</string>
  <key>forgerock_registration_service_name</key>
  <string>Registration</string>
</dict>
```

Descriptions of relevant values:

- `forgerock_cookie_name`: If you have ForgeRock Identity Cloud, you can find this random string value under the "Tenant Settings" found in the top-right dropdown in the admin UI. If you have your own installation of ForgeRock AM this is often `iPlanetDirectoryPro`.
- `forgerock_url`: The URL of AM within your ForgeRock server installation.
- `forgerock_realm`: The realm of your ForgeRock server (likely `root`, `alpha` or `beta`).
- `forgerock_auth_service_name`: This is the journey/tree that you use for login.
- `forgerock_registration_service_name`: This is the journey/tree that you use for registration.

Note: If you don't use Xcode to create the file, your build may not add it to the workspace creating an error. To fix this, make sure to add the file through Xcode, or manually add the file to the Copy Bundle Resources in the workspace target settings.

### Step 3. Write the `start` method

Staying within the `reactnativetodo` directory, find the `FRAuthSampleBridge` file and open it. We have some of the file already stubbed out and the dependencies are already installed. All you need to do is write the functionality.

For the SDK to initialize with the `FRAuth.plist` configuration from Step 2, write the `start` function as follows:

```diff-swift
  import Foundation
  import FRAuth
  import FRCore
  import UIKit

  @objc(FRAuthSampleBridge)
  public class FRAuthSampleBridge: NSObject {
    var currentNode: Node?

    @objc static func requiresMainQueueSetup() -> Bool {
      return false
    }

+   @objc func start(
+     _ resolve: @escaping RCTPromiseResolveBlock,
+     rejecter reject: @escaping RCTPromiseRejectBlock) {

+     /**
+      * Set log level to all
+      */
+     FRLog.setLogLevel([.all])
+
+     do {
+       try FRAuth.start()
+       let initMessage = "SDK is initialized"
+       FRLog.i(initMessage)
+       resolve(initMessage)
+     } catch {
+       FRLog.e(error.localizedDescription)
+       reject("Error", "SDK Failed to initialize", error)
+     }
+   }

    /**
     * Method for calling the `getUserInfo` to retrieve the user information from
     * the OIDC endpoint
     */
    @objc func getUserInfo(
      _ resolve: @escaping RCTPromiseResolveBlock,
      rejecter reject: @escaping RCTPromiseRejectBlock) {

@@ collapsed @@
```
{: .diff-highlight}

The `start` function above calls the iOS SDK’s `start` method on the `FRAuth` class. There’s a bit more that may be required within this function for a production app. We'll get more into this in a separate part of this series, but for now, let's keep this simple.

### Step 4. Write the `login` method

Once the `start` method is called and it has initialized, the SDK is now ready to handle user requests. Let’s start with `login`.

Just underneath the `start` method we wrote above, add the `login` method.

```diff-swift
@@ collapsed @@

  @objc func start(
    _ resolve: @escaping RCTPromiseResolveBlock,
    rejecter reject: @escaping RCTPromiseRejectBlock) {

    /**
     * Set log level according to  all
     */
    FRLog.setLogLevel([.all])

    do {
      try FRAuth.start()
      let initMessage = "SDK is initialized"
      FRLog.i(initMessage)
      resolve(initMessage)
    } catch {
      FRLog.e(error.localizedDescription)
      reject("Error", "SDK Failed to initialize", error)
    }
  }

+ @objc func login(
+   _ resolve: @escaping RCTPromiseResolveBlock,
+   rejecter reject: @escaping RCTPromiseRejectBlock) {
+
+   FRUser.login { (user, node, error) in
+     self.handleNode(user, node, error, resolve: resolve, rejecter: reject)
+   }
+ }

@@ collapsed @@
```
{: .diff-highlight}

This new `login` function initializes the journey/tree specified for authentication. You call this method without arguments as it does not login the user. This initial call to the ForgeRock server will return the first set of callbacks that represents the first node in your journeyt/tree to collect user data.

Also, notice that we have a special "handler" function within the callback of `FRUser.login`. This `handleNode` method serializes the `node` object that the iOS SDK returns in a JSON string. Data passed between the "native" layer and the React layer is limited to strings. This method can be written in many ways and should be written in whatever way is best for your application. However, a unique use of the ForgeRock JavaScript SDK to convert this basic JSON of data into a decorated object for better ergonomics is used in this tutorial.

### Step 5. Write the `next` method

To finalize the functionality needed to complete user authentication, we need a way to iteratively call `next` until the tree completes successfully or fails. To do this, continue in the bridge file, and add a private method called `handleNode`.

First, we will write the decoding of the JSON string and prepare the node for submission.

```diff-swift
@@ collapsed @@

  @objc func login(
    _ resolve: @escaping RCTPromiseResolveBlock,
    rejecter reject: @escaping RCTPromiseRejectBlock) {

    FRUser.login { (user, node, error) in
      self.handleNode(user, node, error, resolve: resolve, rejecter: reject)
    }
  }

+ @objc func next(
+   _ response: String,
+   resolve: @escaping RCTPromiseResolveBlock,
+   rejecter reject: @escaping RCTPromiseRejectBlock) {
+
+   let decoder = JSONDecoder()
+   let jsonData = Data(response.utf8)
+   if let node = self.currentNode {
+     var responseObject: Response?
+     do {
+       responseObject = try decoder.decode(Response.self, from: jsonData)
+     } catch  {
+       FRLog.e(String(describing: error))
+       reject("Error", "UnknownError", error)
+     }
+
+     let callbacksArray = responseObject!.callbacks ?? []
+
+     for (outerIndex, nodeCallback) in node.callbacks.enumerated() {
+       if let thisCallback = nodeCallback as? SingleValueCallback {
+         for (innerIndex, rawCallback) in callbacksArray.enumerated() {
+           if let inputsArray = rawCallback.input, outerIndex == innerIndex,
+             let value = inputsArray.first?.value {
+
+             thisCallback.setValue(value.value as! String)
+           }
+         }
+       }
+     }
+   } else {
+     reject("Error", "UnknownError", nil)
+   }
+ }

@@ collapsed @@
```
{: .diff-highlight}

Now that you’ve prepared the data for submission, introduce the `node.next` call from the iOS SDK. Then, handle the subsequent `node` returned from the `next` call, or process the success or failure representing the completion of the journey/tree.

```diff-swift
@@ collapsed @@

      for (outerIndex, nodeCallback) in node.callbacks.enumerated() {
        if let thisCallback = nodeCallback as? SingleValueCallback {
          for (innerIndex, rawCallback) in callbacksArray.enumerated() {
            if let inputsArray = rawCallback.input, outerIndex == innerIndex,
              let value = inputsArray.first?.value {

              thisCallback.setValue(value)
            }
          }
        }
      }

+     node.next(completion: { (user: FRUser?, node, error) in
+       if let node = node {
+         self.handleNode(user, node, error, resolve: resolve, rejecter: reject)
+       } else {
+         if let error = error {
+           reject("Error", "LoginFailure", error)
+           return
+         }
+
+         let encoder = JSONEncoder()
+         encoder.outputFormatting = .prettyPrinted
+         if let user = user,
+           let token = user.token,
+           let data = try? encoder.encode(token),
+           let accessInfo = String(data: data, encoding: .utf8) {
+
+           resolve(["type": "LoginSuccess", "accessInfo": accessInfo])
+         } else {
+           resolve(["type": "LoginSuccess", "accessInfo": ""])
+         }
+       }
+     })
    } else {
      reject("Error", "UnknownError", nil)
    }
  }

@@ collapsed @@
```
{: .diff-highlight}

The above code handles a limited number of callback types. Handling full authentication and registration journeys/trees requires additional callback handling. To keep this tutorial simple, we’ll focus just on `SingleValueCallback` type.

### Step 6. Write the `logout` bridge method

Finally, add the following lines of code to enable logout for the user:

```diff-swift
@@ collapsed @@

    } else {
      reject("Error", "UnknownError", nil)
    }

+   @objc func logout() {
+     FRUser.currentUser?.logout()
+   }
  }

@@ collapsed @@
```
{: .diff-highlight}

## Building a React Native form for simple login

### Step 1. Initializing the SDK

First, let’s review how the application renders the home view:

`index.js` > `src/index.js` > `src/router.js` > `src/screens/home.js`

Open up the second file in the above sequence, the `src/index.js` file, and write the following:

1. Import `useEffect` from the React library
2. Import `NativeModules` from the `react-native` package
3. Pull `FRAuthSampleBridge` from the `NativeModules` object
4. Write an `async` function within the `useEffect` callback to call the SDK `start` method

```diff-jsx
- import React from 'react';
+ import React, { useEffect } from 'react';
+ import { NativeModules } from 'react-native';
  import { SafeAreaProvider } from 'react-native-safe-area-context';

  import Theme from './theme/index';
  import { AppContext, useGlobalStateMgmt } from './global-state';
  import Router from './router';

+ const { FRAuthSampleBridge } = NativeModules;

  export default function App() {
    const stateMgmt = useGlobalStateMgmt({});

+   useEffect(() => {
+     async function start() {
+       await FRAuthSampleBridge.start();
+     }
+     start();
+   }, []);

    return (
      <Theme>
        <AppContext.Provider value={stateMgmt}>
          <SafeAreaProvider>
            <Router />
          </SafeAreaProvider>
        </AppContext.Provider>
      </Theme>
    );
  }
```
{: .diff-highlight}

`FRAuthSampleBridge` is the JavaScript representation of the Swift bridge code we developed earlier. Any public methods added to the Swift class within the bridge code are available within the `FRAuthSampleBridge` object.

_Note: It's important to initialize the SDK at a root level. Call this initialization step so it resolves before any other native SDK methods can be used._

### Step 2. Building the login view

Navigate to the app’s login view within the Simulator. You should see a "loading" spinner and a message that’s persistent, since the app doesn’t have the data needed to render the form. To ensure the correct form is rendered, the initial data needs to be retrieved from the ForgeRock server. That will be the first task.

{% include image-caption.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-session-loading.png" title="Login screen with spinner" caption="Screenshot of to-do app’s login with spinner" %}

Since most of the action is taking place in `src/components/journey/form.js`, open it and add the following:

1. Import `FRStep` from the `@forgerock/javascript-sdk` for improved ergonomics for handling callbacks
2. Import `NativeModules` from the `react-native` package
3. Pull `FRAuthSampleBridge` from the `NativeModules` object

```diff-jsx
+ import { FRStep } from '@forgerock/javascript-sdk';
  import React from 'react';
+ import { NativeModules } from 'react-native';

  import Loading from '../utilities/loading';

+ const { FRAuthSampleBridge } = NativeModules;

@@ collapsed @@
```
{: .diff-highlight}

To development the login functionality, we first need to use the `login` method from the bridge code to get the first set of callbacks, and then render the form appropriately. This `login` method is an asynchronous method, so import a few additional packages from React to encapsulate this "side effect". Let’s get started!

Import two new modules from React: `useState` and `useEffect`. The `useState` method is for managing the data received from the ForgeRock server, and the `useEffect` is for the `FRAuthSampleBridge.login` method's asynchronous, network request.

Compose the data gathering process using the following:

1. Import `useEffect` from the React library.
2. Write the `useEffect` function inside the component function.
3. Write an `async` function within the `useEffect` for calling `login`.
4. Write an `async` `logout` function to ensure user if fully logged out before attempting to `login`.
5. Call `FRAuthSampleBridge.login` to initiate the call to the login journey/tree.
6. When the `login` call returns with the data, parse the JSON string.
7. Assign that data to our component state via the `setState` method.
8. Lastly, call this new method to execute this process.

```diff-jsx
  import { FRStep } from '@forgerock/javascript-sdk';
- import React from 'react';
+ import React, { useEffect, useState } from 'react';
  import { NativeModules } from 'react-native';

  import Loading from '../utilities/loading';

  const { FRAuthSampleBridge } = NativeModules;

  export default function Form() {
+   const [step, setStep] = useState(null);
+   console.log(step);
+
+   useEffect(() => {
+     async function getStep() {
+       try {
+         await FRAuthSampleBridge.logout();
+         const dataString = await FRAuthSampleBridge.login();
+         const data = JSON.parse(dataString);
+         const initialStep = new FRStep(data);
+         setStep(initialStep);
+       } catch (err) {
+         setStep({
+           type: 'LoginFailure',
+           message: 'Application state has an error.',
+         });
+       }
+     }
+     getStep();
+   }, []);

    return <Loading message="Checking your session ..." />;
  }
```
{: .diff-highlight}

NOTE: We are passing an empty array as the second argument into `useEffect`. This instructs the `useEffect` to only run once after the component mounts. This is functionally is equivalent to a [class component using `componentDidMount` to run an asynchronous method after the component mounts](https://reactjs.org/docs/react-component.html#componentdidmount).

The above code will result in two logs to your console:

1. `null`
2. An object with a few properties.

The property to focus on is the `callbacks` property. This property contains the instructions for what needs to be rendered to the user for input collection.

Import the components from NativeBase as well as the custom, local components within this `journey/` directory:

```diff-jsx
  import { FRStep } from '@forgerock/javascript-sdk';
+ import { Box, Button, FormControl, ScrollView } from 'native-base';
  import React, { useEffect, useState } from 'react';
  import { NativeModules } from 'react-native';

  import Loading from '../utilities/loading';
+ import Alert from '../utilities/alert';
+ import Password from './password';
+ import Text from './text';
+ import Unknown from './unknown';

@@ collapsed @@
```
{: .diff-highlight}

Now, within the `Form` function body, create the function that maps these imported components to their appropriate callbacks.

```diff-jsx
@@ collapsed @@

  export default function Form() {
    const [step, setStep] = useState(null);
    console.log(step);

@@ collapsed @@

+   function mapCallbacksToComponents(cb, idx) {
+     const name = cb?.payload?.input?.[0].name;
+     switch (cb.getType()) {
+       case 'NameCallback':
+         return <Text callback={cb} inputName={name} key="username" />;
+       case 'PasswordCallback':
+         return <Password callback={cb} inputName={name} key="password" />;
+       default:
+         // If current callback is not supported, render a warning message
+         return <Unknown callback={cb} key={`unknown-${idx}`} />;
+     }
+   }

    return <Loading message="Checking your session ..." />;
  }
```
{: .diff-highlight}

Finally, return the appropriate component for the following states:

- If there is no `step` data, render the `Loading` component to indicate the request is still processing.
- If there is `step` data, and it is of type `'Step'`, then map over `step.callbacks` with the function from above.
- If there is `step` data, but the type is `'LoginSuccess'` or `'LoginFailure'`, render an alert.

```diff-jsx
@@ collapsed @@

+ if (!step) {
    return <Loading message='Checking your session ...' />;
+ } else if (step.type === 'Step') {
+   return (
+     <ScrollView>
+       <Box safeArea flex={1} p={2} w="90%" mx="auto">
+         <FormControl>
+           {step.callbacks?.map(mapCallbacksToComponents)}
+           <Button>Sign In</Button>
+         </FormControl>
+       </Box>
+     </ScrollView>
+   );
+ } else {
+   // Handle success or failure of the journey/tree
+   return (
+     <Box safeArea flex={1} p={2} w="90%" mx="auto">
+       <Alert message={step.message} type={step.type} />
+     </Box>
+   );
+ }
```
{: .diff-highlight}

Refresh the page, and you should now have a dynamic form that reacts to the callbacks returned from our initial call to ForgeRock.

{% include image-caption.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-login-form.png" title="Login screen form" caption="Screenshot of login screen with rendered form" %}

### Step 3. Handling the login form submission

Since a form that can’t submit anything isn’t very useful, we’ll now handle the submission of the user input values to ForgeRock. First, add a second `useState` to track whether the user is authenticated or not, and then edit the current `Button` element adding an `onPress` handler with a simple, inline function. This function should do the following:

1. Submit the modified `step` data to ForgeRock with the `FRAuthSampleBridge.next` method.
2. Test if the response property `type` has the value of `'LoginSuccess'`.
3. If successful, parse the `response` JSON.
4. Call `setStep` with the new object parsed from the JSON (this is mostly just for logging the step to the console).
5. Call `setAuthentication` to true, which is a global state method that triggers the app to react (pun intended!) to the new user state.
6. Handle errors with a generic failure message.

```diff-jsx
@@ collapsed @@

  export default function Form() {
    const [step, setStep] = useState(null);
+   const [isAuthenticated, setAuthentication] = useState(false);
    console.log(step);

@@ collapsed @@

    return (
      <ScrollView>
        <Box safeArea flex={1} p={2} w="90%" mx="auto">
          <FormControl>
            {step.callbacks?.map(mapCallbacksToComponents)}
-         <Button>Sign In</Button>
+         <Button
+           onPress={() => {
+             async function getNextStep() {
+               try {
+                 const response = await FRAuthSampleBridge.next(
+                   JSON.stringify(step.payload),
+                 );
+                 if (response.type === 'LoginSuccess') {
+                   const accessInfo = JSON.parse(response.accessInfo);
+                   setStep({
+                     accessInfo,
+                     message: 'Successfully logged in.',
+                     type: 'LoginSuccess',
+                   });
+                   setAuthentication(true);
+                 } else {
+                   setStep({
+                     message: 'There has been a login failure.',
+                     type: 'LoginFailure',
+                   });
+                 }
+               } catch (err) {
+                 console.error(`Error: form submission; ${err}`);
+               }
+             }
+             getNextStep();
+           }}
+         >
+           Sign In
+         </Button>
        </FormControl>
      </Box>
    </ScrollView>
  );

@@ collapsed @@
```
{: .diff-highlight}

After the app refreshes, use the test user to login. If successful, you should see a success message. Congratulations, you are now able to authenticate users!

{% include image-caption.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-login-success.png" title="Login screen with successful authentication" caption="Screenshot of login screen with success alert" %}

What's more, you can verify the authentication details by going to the Xcode or Safari log and observing the result of the last call to the ForgeRock server. It should have a type of `"LoginSuccess"` along with token information.

{% include image-wide.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-xcode-login-response.png" title="Successful login response from Xcode" caption="Screenshot of successful login response in Xcode's output" %}

NOTE: If you got a login failure, you can re-attempt the login by going to the Device menu on the Simulator and selecting "Shake". This will allow you to reload the app, providing a fresh login form.

#### Handling the user provided values

You may ask, "How did the user’s input values get added to the `step` object?" Let’s take a look at the component for rendering the username input. Open up the `Text` component: `components/journey/text.js`. Notice how special methods are being used on the callback object. These are provided as convenience methods by the JavaScript SDK for getting and setting data.

```diff-jsx
@@ collapsed @@

  export default function Text({ callback }) {

@@ collapsed @@

    const error = handleFailedPolicies(
      callback.getFailedPolicies ? callback.getFailedPolicies() : [],
    );
    const isRequired = callback.isRequired ? callback.isRequired() : false;
    const label = callback.getPrompt();
    const setText = (text) => callback.setInputValue(text);
    return (
      <FormControl isRequired={isRequired} isInvalid={error}>
        <FormControl.Label mb={0}>{label}</FormControl.Label>
        <Input
          autoCapitalize="none"
          autoComplete="off"
          autoCorrect={false}
          onChangeText={setText}
          size="lg"
          type="text"
        />
        <FormControl.ErrorMessage>
          {error.length ? error : ''}
        </FormControl.ErrorMessage>
      </FormControl>
    );
  }
```
{: .diff-highlight}

There are two important items to focus on

- `callback.getPrompt()`: Retrieves the input's label to be used in the UI.
- `callback.setInputValue()`: Sets the user’s input on the callback while they are typing (i.e. `onChangeText`).

Since the `callback` is passed from the `Form` to the components by "reference" (not by "value"), any mutation of the `callback` object within the `Text` (or `Password`) component is also contained within the `step` object in the `Form` component.

NOTE: You may think, "That’s not very idiomatic React! Shared, mutable state is bad." And, yes, you are correct, but we are taking advantage of this to keep everything simple (and this guide from being too long), so I hope you can excuse the pattern.

Each callback type has its own collection of methods for getting and setting data in addition to a base set of generic callback methods. The SDK automatically adds these methods to the callback's prototype. For more information about these callback methods, [see our API documentation](https://sdks.forgerock.com/javascript/api-reference-core/index.html), or [the source code in Github](https://github.com/ForgeRock/forgerock-javascript-sdk/tree/master/src/fr-auth/callbacks), for more details.

### Step 4. Requesting user info and redirecting to home screen

Now that the user can login, let's go one step further and request information about the authenticated user to display their name and other information. We will now utilize the existing `FRAuthSampleBridge.getUserInfo` method already included in the bridge code.

Let's do a little setup before we make the request to the ForgeRock server:

1. Add `useContext` to the import from React so that we have access to the global state
2. Import `AppContext` from the `global-state` module
3. Call `useContext` with our `AppContext` to provide access to the setter methods
4. Add one more `useEffect` function to detect the change of the user's authentication

```diff-jsx
  import { FRStep } from '@forgerock/javascript-sdk';
  import { Box, Button, FormControl, ScrollView } from 'native-base';
- import React, { useEffect, useState } from 'react';
+ import React, { useContext, useEffect, useState } from 'react';
  import { NativeModules } from 'react-native';

+ import { AppContext } from '../../global-state';
@@ collapsed @@

  export default function Form() {
+   const [_, methods] = useContext(AppContext);
    const [step, setStep] = useState(null);
    const [isAuthenticated, setAuthentication] = useState(false);
    console.log(step);

    useEffect(() => {
      async function getStep() {
        try {
          await FRAuthSampleBridge.logout();
          const dataString = await FRAuthSampleBridge.login();
          const data = JSON.parse(dataString);
          const initialStep = new FRStep(data);
          setStep(initialStep);
        } catch (err) {
          console.error(`Error: request for initial step; ${err}`);
        }
      }
      getStep();
    }, []);

+   useEffect(() => {
+
+   }, [isAuthenticated]);

@@ collapsed @@
```
{: .diff-highlight}

It's worth noting that the `isAuthenticated` declared in the array communicates to React that this `useEffect` should only execute if the state of that variable changes. This prevents unnecessary code execution since the value is initially `false`, and continues to be `false` until the user completes authentication.

With the setup is done, implement the request to ForgeRock for the user's information. Within this empty `useEffect`, add an `async` function to make that call to `FRAuthSampleBridge.getUserInfo` and call it only when `isAuthenticated` is `true`.

```diff-jsx
@@ collapsed @@

    useEffect(() => {
+     async function getUserInfo() {
+       const userInfo = await FRAuthSampleBridge.getUserInfo();
+       console.log(userInfo);
+
+       methods.setName(userInfo.name);
+       methods.setEmail(userInfo.email);
+       methods.setAuthentication(true);
+     }
+
+     if (isAuthenticated) {
+       getUserInfo();
+     }
    }, [isAuthenticated]);

@@ collapsed @@
```
{: .diff-highlight}

In the code above, we collected the user information and set a few values to the global state to allow the app to react to this information. In addition to updating the global state, the React Navigation also reacts to the global state change and renders the new screens and tab navigation.

When you test this in the Simulator, completing a successful authentication results in the home screen being rendered with a success message. The user's name and email will be included for visual validation. You will also be able to view the console in Safari and see the user's information logged.

{% include image-caption.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-home-screen-success.png" title="Home screen after successful authentication" caption="Screenshot of home screen with success alert" %}

## Adding logout functionality to our bridge and React Native code

Clicking on the "Sign Out" button within the navigation results in the logout page rendering with a persistent "loading" spinner and message. This is due to the missing logic that we'll add now.

{% include image-caption.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-logging-out.png" title="Logout screen with spinner" caption="Screenshot of logout screen with spinner" %}

### Handle logout request in React Native

Now, add the logic into the view to call this new Swift method. Open up the `screens/logout.js` file and import the following:

1. `useEffect` and `useContext` from React
2. `useHistory` from React Router
3. `AppContext` from the global state module

```diff-jsx
- import React from 'react';
+ import React, { useContext, useEffect } from 'react';
+ import { NativeModules } from 'react-native';

+ import { AppContext } from '../global-state';
  import { Loading } from '../components/utilities/loading';

+ const { FRAuthSampleBridge } = NativeModules;

@@ collapsed @@
```
{: .diff-highlight}

Since logging out requires an `async`, network request, we need to wrap it in a `useEffect` and pass in a callback function with the following functionality:

```diff-jsx
@@ collapsed @@

  export default function Logout() {
+   const [_, { setAuthentication }] = useContext(AppContext);
+
+   useEffect(() => {
+     async function logoutUser() {
+       try {
+         await FRAuthSampleBridge.logout();
+       } catch (err) {
+         console.error(`Error: logout; ${err}`);
+       }
+       setAuthentication(false);
+     }
+     logoutUser();
+   }, []);
+
    return <Loading message="You're being logged out ..." />;
  }
```
{: .diff-highlight}

Since we only want to call this method once, after the component mounts, we will pass in an empty array as a second argument for `useEffect`. The use of the `setAuthentication` method empties or falsifies the global state to clean up and rerender the home screen.

Revisit the app within the Simulator, and tap the "Sign Out" button. You should see a quick flash of the loading screen, and then the home screen should be displayed with the logged out UI state.

{% include image-caption.html imageurl="/assets/images/posts/2022/build-reactnative-part-one/react-native-home-screen.png" title="Logged out home screen" caption="Screenshot of home screen in logged out state" %}

You should now be able to successfully authenticate a user, display the user's information, and log a user out. Congratulations, you just built a ForgeRock protected iOS app with React Native.

In part two of this series, we explore how to write the bridge code for Android, while utilizing the existing React Native UIs written here.
