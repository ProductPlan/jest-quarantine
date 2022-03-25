# jest-quarantine

> A quarantine method and reporter for [jest](https://jestjs.io/) to help quite flappy tests.

## Features

- Use a new `quarantine` method to wrap test assertions in a try/catch
- Opt in to using a date string to specify an expiration for a quarantined test or keep it there forever
- Track pass/fail status and the it last ran for quarantined tests
- Report on all the things in a test run:
  - the number of quarantined tests
  - which tests are quarantined (with a link back to the file)
  - whether or not quarantined tests passed

## Motivation

Sometimes tests flap (pass -> fail -> pass -> etc) and we don't have the time at hand to fix them. We want to, or at least should, but we know enough to know that a proper fix isn't simple and will need to given dedicated time. In cases like these we want to have a way to easily quarantine tests so that they don't break builds, without skipping them or commenting them out, until we have a chance to properly address them.

### The underlying idea

Unfortunately jest doesn't have a quarantine mechanism built in so we're often left to come up with our own approaches. Some of these commonly include:

- **run it again until it passes** - which isn't ideal because it wastes resources and doesn't address the actual issue in any way
- **skip the test (using the [skip method](https://jestjs.io/docs/api#testskipname-fn))** - this isn't the worst but skipped tests are grouped together and don't get run so they often get forgotten
- **comment out the test** - a bit worse than skipping because now there is no record of them unless someone goes to that file but again they're out of the way
- **mark it as todo (using the [todo method](https://jestjs.io/docs/api#testtodoname))** - this is nice approach since "todos" are reported separately but the method doesn't take a callback so we still have to change the underlying test and don't know if they would pass/fail during any given run
- **delete the test** - not a big deal if we know we don't need it but when a test is flapping and preventing other planned work might not be the best time to make that determination

To be fair none of these are bad in their own right - they are totally appropriate cases, except maybe for choosing any of them without cursory investigation, but they just aren't ideal. What we might prefer is to run the test, prevent it from blocking if it fails, let it through like normal if it passes, and to report on all the results in some way.

## Introducing jest-quarantine

That's where `jest-quarantine` comes in, or at least where it attempts to help. In a nutshell, it gives us more flexibility so that when we run into flapping tests, and unfortunately we will, we aren't pigeonholed into fixing it or ignoring it but rather we can add it a new unique set of tests that we can track and easily come back to.

---

## Installation

| NPM                              | Yarn                          |
| -------------------------------- | ----------------------------- |
| `npm install -D jest-quarantine` | `yarn add -D jest-quarantine` |

## Quick setup

If all you care about is quarantining tests, and you don't need/want any reporting, then you can just import the `quarantine` method and use it in place of `it/test` where you need it.

```javascript
// some-test-file.spec.js
import { quarantine } from "jest-quarantine";

quarantine("a test that fails but will be quarantined forever", () => {
  expect(1).toBe(2);
});

quarantine(
  "a test that fails but will only be caught until a certain date",
  "2023-01-01",
  () => {
    expect(1).toBe(2);
  }
);
```

In this case our first quarantined test will stay quarantined forever while the second will start being run as normal after "2023-01-01" and any flapping behavior it has will affect test runs again.

_Note: you can also follow the directions for using `setupQuarantine` below if you'd like for the `quarantine` method to exist in the global space but don't want reporting._

## Kitchen sink setup

If on the other hand you want quarantining as well as reporting we also need to call the setup method. This is good case for anyone who wants to surface quarantined results during local runs or possibly store the output as a build artifact that can be tracked over time.

To do so we need to call `setupQuarantine` in any file already being passed to [setupFilesAfterEnv](https://jestjs.io/docs/configuration#setupfilesafterenv-array). This will:

- add the new `quarantine` method to the `global` space
- set up the required tracking array
- and save the results to a combined tracking log as a part of `afterAll`

```javascript
// jest.testSetup.js - which is run via setupFilesAfterEnv
import { setupQuarantine } from "jest-quarantine";
setupQuarantine();
```

Then hook up the `reporter` in the jest config if you want to surface the results locally (not required, just helpful). See the reporter details below configuration details.

```json
reporters: [
    "default",
    ["jest-pseudo-quarantine/reporter.js", { enabled: true, showTests: true, enforceExpiration: true }],
],
```

## Reporter options

| Option     | Type    | Default | Description                                                                     |
| ---------- | ------- | ------- | ------------------------------------------------------------------------------- |
| enabled    | boolean | true    | Turns on/off the reporter                                                       |
| showTests  | boolean | false   | Shows which tests were quarantined in the output and whether or not they passed |
| colorLevel | number  | 2       | Determines whether or not the output has color                                  |
| logger     | object  | console | Determines where to log output to                                               |

## Exported methods

| Method          | Use                                                                                   |
| --------------- | ------------------------------------------------------------------------------------- |
| quarantine      | Can be used as a replacement to `it/test/jest.test`.                                  |
| setupQuarantine | Adds the `quarantine` method and an array for tracking results to the `global` space. |

## Generated files

Multiple files are created while saving results:

- `quarantined-tests/test/file/path/and/name.log` = used to store quarantine results for a single file
- `quarantined-tests/combined-results.log` = used to store all of the results combined and should be added to your `.gitignore`

---

## Caveats

**The `quarantine` method doesn't currently support the [timeout argument](https://jestjs.io/docs/api#testname-fn-timeout)**

This could be added in at any point, especially if someone has a strong need and is willing to open a PR for it, however we simply don't use that option and so it's not baked in.

**Jest doesn't expose suite names**

Because of this the store test names are only those that are passed to `quarantine` - which means the output for a given file might not be very helpful if you have tests like these:

```javascript
describe("suite a", () => {
  it("does something", () => {});
});

describe("suite b", () => {
  it("does something", () => {});
});
```

# Contributing

First off, thank you for considering contributing. We are always happy to accept contributions and will do our best to ensure they receive the appropriate feedback.

Things to know:

- Commits are required to follow the [Conventional Commits Specification](https://www.conventionalcommits.org/en/v1.0.0/).
- There is no planned/automated cadence for releases. Because of this once changes are merged they will be manually published, at least for the time being.

## Making changes/improvements

To contribute changes:

- Fork the repo
- Create your feature branch (git checkout -b my-new-feature)
- Make, test, and commit your changes (git commit -am 'feat: Add some feature')
- Push to the branch (git push origin my-new-feature)
- Create a new Pull Request

Note: only changes accompanied by tests will be considered for merge.

## Reporting issues

If you encounter any issue please feel free to open a new ticket and report it.
