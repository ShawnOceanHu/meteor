# Meteor Tool

This part of the code-base is Meteor's CLI tool that reads commands from the
user, builds the application, adds runtime code, provides an interface to Meteor
Services ([Accounts](https://www.meteor.com/services/developer-accounts),
[Packages](https://www.meteor.com/services/package-server),
[Build Farms](https://www.meteor.com/services/build), deployments, etc).

The Meteor Tool is designed to be the "minimal kernel" as most of the
functionality that goes into a typical Meteor app can pulled from core and
3rd-party packages.

## Getting set up

Using Meteor in development is very simple. If Meteor spots that it is running
from a Git checkout folder (having a `.git` directory), it will run in dev mode,
download `npm` dependencies dynamically and pull the latest `dev_bundle` on the
first run.

`dev_bundle` is a tarball with prebuilt binaries (`node`, `npm`, `mongod`, etc)
and npm modules necessary for the Meteor tool. `dev_bundle`s are versioned and
are built with a script in the `admin` directory. It is commonly built on
[Jenkins](http://jenkins.meteor.io/).

Usually it doesn't take long to get a new `dev_bundle` but if you are on a
spotty network or switching between branches referencing different versions
often, you can set the environment variable that will cache all downloaded
versions indefinitely:

```bash
set SAVE_DEV_BUNDLE_TARBALL=t
```

You can also run `./meteor --get-ready` to install all npm dependencies for the
tool.

Usually, the `meteor` script can download a new dev-bundle without any
dependencies installed, but on Windows, it requires `7z` to be in the path for
unpacking of a tarball. (Get 7-zip [here](http://www.7-zip.org/))

## Testing

Since the tool is a node app, it is not testable with general Meteor testing
tools such as Tinytest or Velocity. Instead the home-grown system "self test" is
used.

"Self test" is a testing library that is focused on testing the CLI interactions
and is rather an end-to-end testing tool (not a unit-testing tool). Albeit, it
is often used for testing individual functions.

Besides monitoring the process output, "self test" is capable of moching the
package catalog, running from template apps.

The asserting syntax of "self test" is rather unusual since it operates on the
process's stdout/stderr output after the process has run (not in real-time).
A lot of assertions depend on timeouts and waits.

To run all tests, run the following:

```bash
# download all npm dependencies, etc
./meteor --get-ready

# set the multiplier for time-outs
set TIMEOUT_SCALE_FACTOR=3

# run the tests
./meteor self-test
```

Note, the scale factor for time-outs might be different depending on the
hardware, but 3 is a safe choice for automation.

To quickly run an individual test or a group, pass a regular expression as an
argument, it will be matched against test names:

```bash
./meteor self-test "login.*"
```

You can also run a particular file, or list all tests matching certain
pattern, run with phantom or browserstack.
See more at `./meteor help self-test`.


## Profiling

Profiling is done in an ad-hoc way but it works well enough to spot obvious
differences in things like build performance.

To enable profiling, set the environment variable to a "cut off" point, which is
100ms by default.

```bash
set METEOR_PROFILE=200
```

In this case, the reporter will only print calls that took more than 200ms to
complete.

Internally, every profiled function should be wrapped into a `Profile(fn)` call.
The entry point should be started explicitly with the `Profile.run()`
call. Otherwise, it won't start measuring anything.

## Debugging

Currently, debugging the tool with `node-inspector` is only possible if you edit
the `meteor` bash/batch script and add the `--debug` or `--debug-brk` flag to
the node call. Note that `node-inspector` should be compatible with the `node`
version in the `dev_bundle`.


## Development

The entry-point of the tools code is in `index.js`.

### Devel vs Prod environment

The Meteor Tool code has two modes of running:

- from local checkout for development
- from a production release installed by running
`curl -L https://install.meteor.com | sh` or a Windows installer.

There are two different `meteor` / `meteor.bat` starting scripts in development
and production. The production one is written by the packaging code.

In addition to that, the checkout version stores its own catalog (`.meteor` dir)
and the production version keeps it in `~/.meteor`.

When the release is published (with `./meteor publish-release --from-checkout`),
the files committed into Git are copied in a compiled form into the built
package (Isopack). You can find the list of copied sub-trees in
`Isopack#_writeTool`.


### Buildmessage

Throughout the code-base, there is an extensive use of `buildmessage`, which is
a custom try/catch/finally system with recovery. See
[`/tools/utils/buildmessage.md`](utils/buildmessage.md) for more details.
