---
layout: post
title: 'The Layered Cake of Testing'
tags: [Code Quality, Testing, Quick Explainer]
author: justin
featured_image: assets/images/posts/2019/cake-by-jayden-sim-unsplash.jpg
---

Not having a comprehensive test strategy is like architecting a building without knowing the environment in which it will be built and how it will be stressed.

## What's a test strategy?

This is a plan on what and how you plan to test the different concepts or patterns without you application. Ideally, your test strategy is decided during your architectural design phase of your application. The best outcomes are driven by choosing frameworks, libraries, patterns and conventions that emphasize the ease of which these decisions can be tested and maintained.

## The 4 (or maybe 5) layers of testing

A test strategy should be a multi-layered approach to ensure quality code. Each layer has its purpose and should be used in accordance to its strength and weakness. Static code analysis and end-to-end tests are both important, and one cannot replace the other. This is because they have completely different responsibilities and a completely different set of pros and cons.

With that said, let's take a look at the layers and discuss the intentions of each.

### Static code analysis

This includes linting, type checking, code-flow analysis ... TypeScript, Reason, Flowtype, ESLint, TSLint are all examples of static code analysis.

#### Critical focus

1. Speed
2. "Set and forget"
3. Automated
4. Verifies code patterns and syntax

#### When to run

These should be run as frequently as possible. Optimally, they are continuously running while developing.

#### Code smells

1. Ambiguous types on large objects, like TypeScript's `any`
2. Overly typing entities, rather than allowing for type inference
3. Not using the compiler's build as a pass-fail mechanism for your pipeline
4. Not taking advantage of automated "fixing" of code when misaligned from standards

#### Coverage

As close to 100% of the codebase should be evaluated.

#### Recommendation

Although [ReasonML](https://reasonml.github.io/) is a _great_ solution due to being both completely type safe and sound, it's not the most well-known and does have a cognitive challenge to its very FP nature. My default recommendation is [TypeScript](https://www.typescriptlang.org/), due to it being type safe (though not sound), but is very well-known and is written and feels very much like JavaScript with types, and not a completely different language.

With this, one should have ESLint and Prettier run automatically at file save or at least at each commit. These tools allow the automation of code "fixing" when there is a misalignment of code from the configured standards. Running this as a part of local development process removes the need for commenting on common syntactical "nits", allowing developers to focus on what really matters in code reviews.

### Unit testing

These are super light-weight tests that run assertions against the most atomized parts of code, the units. These are just testing the result for a given input for a function. These functions/units are _mostly_ pure, stateless functions, so there is no need to mock or stub anything other than the input/arguments provided. In other words, there is nothing external to the unit tested that's needed, no servers, no APIs, no DOM, no HTTP ... nothing.

#### Critical focus

1. Speed
2. Simplicity
3. Low effort
4. Great dev experience
5. Verifying internal structures

#### When to run

These should be run, at minimum, at every `git commit`.

#### Code smells

1. Stubbing functions or methods that have side-effects
2. Mocking (other than the arguments passed) additional objects and global variables
3. `beforeAll` for testing "setup"
4. `afterAll` for testing "teardown"
5. You can't just simply run the test `node myunit.test.js` without running other processes or servers

#### Coverage

Coverage is a dangerous proposition as it can lead to low-value testing just for the sake of numbers. I think the majority of the app's functionality should have unit tests, but whether that's 70%, 80% or 90% is quite arbitrary and quickly surpasses the point of diminishing return.

My view is any function that has hand-written logic in it should have both positive and negative testing. Here are some easy rules that apply to all testing:

1. Only test hand-written logic; don't test native functions, or methods from external libraries or frameworks
2. The more complex the hand-written logic, the more priority should be placed on testing
3. The more critical or "hot path" the function, the more priority should be placed on testing
4. Don't test functions that primarily call out to external entities (usually require mocking) and return the response

**The focus should be on the value (quality) of the tests, not the number of tests (quantity).**

#### Recommendation

My very favorite unit testing library is [`tape`](https://github.com/substack/tape), but it does require quite a bit of manual configuration at scale. For better "out of the box" configuration for scale, [Jest](https://jestjs.io/) is a great alternative that breaks the capability of running the unit test file alone (e.g. `node test.js`), but the benefits that come with it are too great to ignore.

Unit tests should reside right along the files they test. Example:

```text
- todo-list/
  | - todo-filter.ts
  | - todo-filter.test.ts
```

The Jest library has everything needed for unit testing. There should be no additional libraries or frameworks needed.

### Integration testing

These are more similar to unit tests than they are other types of tests. These tests are still input-output driven, but since it tests the composition of units, it tests at a higher, more complex level, so mocking/stubbing is almost always required.

Mocking and stubbing should be done without the need for additional processes, servers or APIs. The mocks or stubs should be static and side-effect free.

#### Critical focus

1. Speed
2. Completeness
3. Good dev experience
4. Verifying internal structures

#### When to run

These should be run at minimum at every `git push`.

#### Code smells

1. `beforeAll` for testing "setup"
2. `afterAll` for testing "teardown"
3. You can't just simply run the test `node myunit.test.js` without running other processes or servers

#### Coverage

All full suites of units should be tested up to the point of needing to run additional processes, servers or needing a real DOM. These should test as much surface area as possible with as few tests as possible and should be contained to just testing the local codebase or module. Never test the functionality of external/third-party functionality.

#### Recommendation

Continuing with the Jest recommendation from above, Jest can also handle integration testing with its ability to mock modules.

### View-component testing

These are analogous to unit or integration tests for React or Vue components. These can be run along with your unit and integration tests. They are input and output based and do not require additional process, servers or live DOM. The difference between a view-component test is that the output of data, the output is a rendered component. Assertions are made against the component's properties.

#### Critical focus

1. Little mocking
2. Simplicity
3. Low effort
4. Good dev experience
5. Verifying rendered component states

#### When to run

These should be run at minimum for each PR to the main branch, but could be run at every `git push`.

#### Code smells

1. Testing a native library or framework function
2. Testing something that is too simple
3. Testing if component has a class or text that is not driven by logic or is static
4. Heavy use of `beforeAll` for testing "setup"
5. Heavy use of `afterAll` for testing "teardown"

#### Coverage

All view-components that have complexity, logic, derived data or branching should be tested.

#### Recommendation

Jest will continue our testing capabilities with the addition of JSDom for mocking the DOM in order for us to synthetically render our components for testing.

### End-to-end testing (aka functional testing/e2e)

This type of testing is to mimic user interaction to the closest degree. It requires a running app and running APIs to simulate the real environment as much as possible. Tests require a Web driver to mimic clicking, typing, navigating and submitting data to a running application against appropriate testing stages.

#### Critical focus

1. Functional completeness
2. Predictability and reliability
3. Mimics real world usage
4. Verifying the API contracts between external, dependent systems
5. Can be run against live systems and mocked systems

#### When to run

These should be run for each PR to the main branch.

#### Code smells

1. You’re testing a native library or framework function
2. You’re testing something that is too simple
3. You’re testing the DOM or browser function

#### Coverage

Critical user flows, security and privacy related features, anything that’s considered high-risk. The intention of a e2e test is to verify the entire user flow within a running environment. Avoid testing functionality that can more easily be tested though unit or integration tests or redundantly testing something that’s already well covered in your unit or integration tests.

#### Recommendation

Playwright, Puppeteer, Cypress are good options. I prefer Playwright due to the support of Microsoft and it being cross-browser.

## Additional references

- [Automated Testing for Terraform, Docker, Packer, Kubernetes, and More](https://www.infoq.com/news/2019/11/automated-testing-infrastructure/)
- [Learn the smart, efficient way to test any JavaScript application](https://testingjavascript.com/)
