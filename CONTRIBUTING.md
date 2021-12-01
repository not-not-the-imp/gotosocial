# Contributing <!-- omit in toc -->

Hey! Welcome to the CONTRIBUTING.md for GoToSocial :) Thanks for taking a look, that kicks ass.

This document will expand as the project expands, so for now this is basically a stub.

Contributions are welcome at this point, since the API is fairly stable now and the structure is somewhat coherent.

Check the [issues](https://github.com/superseriousbusiness/gotosocial/issues) to see if there's anything you fancy jumping in on.

## Table Of Contents  <!-- omit in toc -->

- [Communications](#communications)
- [Code of Conduct](#code-of-conduct)
- [Setting up your development environment](#setting-up-your-development-environment)
  - [Stylesheet / Web dev](#stylesheet--web-dev)
  - [Golang forking quirks](#golang-forking-quirks)
- [Setting up your test environment](#setting-up-your-test-environment)
  - [Standalone Testrig with Pinafore](#standalone-testrig-with-pinafore)
  - [Running automated tests](#running-automated-tests)
    - [SQLite](#sqlite)
    - [Postgres](#postgres)
    - [Both](#both)
- [Project Structure](#project-structure)
- [Style](#style)
- [Linting and Formatting](#linting-and-formatting)
- [Updating Swagger docs](#updating-swagger-docs)
- [CI/CD configuration](#cicd-configuration)
- [Building releases and Docker containers](#building-releases-and-docker-containers)
  - [With GoReleaser](#with-goreleaser)
  - [Manually](#manually)
- [Financial Compensation](#financial-compensation)

## Communications

Before starting on something, please comment on an issue to say that you're working on it, and/or drop into the GoToSocial Matrix room [here](https://matrix.to/#/#gotosocial:superseriousbusiness.org).

This is the recommended way of keeping in touch with other developers, asking direct questions about code, and letting everyone know what you're up to.

## Code of Conduct

In lieu of a fuller code of conduct, here are a few ground rules.

1. We *DO NOT ACCEPT* PRs from right-wingers, Nazis, transphobes, homophobes, racists, harassers, abusers, white-supremacists, misogynists, tech-bros of questionable ethics. If that's you, politely fuck off somewhere else.
2. Any PR that moves GoToSocial in the direction of surveillance capitalism or other bad fediverse behavior will be rejected.
3. Don't spam the general chat too hard.

## Setting up your development environment

To get started, you first need to have Go installed. GtS is currently using Go 1.17, so you should take that too. See [here](https://golang.org/doc/install).

Once you've got go installed, clone this repository into your Go path. Normally, this should be `~/go/src/github.com/superseriousbusiness/gotosocial`.

Once that's done, you can try building the project: `./scripts/build.sh`. This will build the `gotosocial` binary.

If there are no errors, great, you're good to go!

For automatic re-compiling during development, you can use [nodemon](https://www.npmjs.com/package/nodemon):

```bash
nodemon -e go --signal SIGTERM --exec "go run ./cmd/gotosocial --host localhost testrig start || exit 1"
```

### Stylesheet / Web dev

To work with the stylesheet for templates, you need [Node.js](https://nodejs.org/en/download/) and [Yarn](https://classic.yarnpkg.com/en/docs/install).

To install Yarn dependencies:

```bash
yarn install --cwd web/gotosocial-styling
```

To recompile bundles:

```bash
node web/gotosocial-styling/index.js --build-dir="web/assets"
```

You can do automatic live-reloads of bundles with:

``` bash
NODE_ENV=development node web/gotosocial-styling/index.js --build-dir="web/assets"
```

### Golang forking quirks

One of the quirks of Golang is that it relies on the source management path being the same as the one used within `go.mod` and in package imports within individual Go files. This makes working with forks a bit awkward.

Let's say you fork GoToSocial to `github.com/yourgithubname/gotosocial`, and then clone that repository to `~/go/src/github.com/yourgithubname/gotosocial`. You will probably run into errors trying to run tests or build, so you might change your `go.mod` file so that the module is called `github.com/yourgithubname/gotosocial` instead of `github.com/superseriousbusiness/gotosocial`. But then this breaks all the imports within the project. Nightmare! So now you have to go through the source files and painstakingly replace `github.com/superseriousbusiness/gotosocial` with `github.com/yourgithubname/gotosocial`. This works OK, but when you decide to make a pull request against the original repo, all the changed paths are included! Argh!

The correct solution to this is to fork, then clone the upstream repository, then set `origin` of the upstream repository to that of your fork.

See [this blogpost](https://blog.sgmansfield.com/2016/06/working-with-forks-in-go/) for more details.

In case this post disappears, here are the steps (slightly modified):

>
> Pull the original package from the canonical place with the standard go get command:
>
> `go get github.com/superseriousbusiness/gotosocial`
>
> Fork the repository on Github or set up whatever other remote git repo you will be using. In this case, I would go to Github and fork the repository.
>
> Navigate to the top level of the repository on your computer. Note that this might not be the specific package you’re using:
>
> `cd $GOPATH/src/github.com/superseriousbusiness/gotosocial`
>
> Rename the current origin remote to upstream:
>
> `git remote rename origin upstream`
>
> Add your fork as origin:
>
> `git remote add origin git@github.com/yourgithubname/gotosocial`
>

## Setting up your test environment

GoToSocial provides a [testrig](https://github.com/superseriousbusiness/gotosocial/tree/main/testrig) with a bunch of mock packages you can use in integration tests.

One thing that *isn't* mocked is the Database interface, because it's just easier to use an in-memory SQLite database than to mock everything out.

### Standalone Testrig with Pinafore

You can launch a testrig as a standalone server running at localhost, which you can connect to using something like [Pinafore](https://github.com/nolanlawson/pinafore).

To do this, first build the gotosocial binary with `./scripts/build.sh`.

Then, launch the testrig by invoking the binary as follows:

```bash
GTS_HOST="localhost" GTS_DB_TYPE="sqlite" GTS_DB_ADDRESS=":memory:" ./gotosocial --host localhost:8080 testrig start
```

To run Pinafore locally in dev mode, first clone the [Pinafore](https://github.com/nolanlawson/pinafore) repository, and then run the following command in the cloned directory:

```bash
yarn run dev
```

The Pinafore instance will start running on `localhost:4002`.

To connect to the testrig, navigate to `https://localhost:4002` and enter your instance name as `localhost:8080`.

At the login screen, enter the email address `zork@example.org` and password `password`. You will get a confirmation prompt. Accept, and you are logged in as Zork.

Note the following constraints:

- Since the testrig uses an in-memory database, the database will be destroyed when the testrig is stopped.
- If you stop the testrig and start it again, any tokens or applications you created during your tests will also be removed. As such, you need to log out and in again every time you stop/start the rig.
- The testrig does not make any actual external http calls, so federation will not work from a testrig.

### Running automated tests

There are a few different ways of running tests. Each requires the use of the `-p 1` flag, to indicate that they should not be run in parallel.

#### SQLite

If you want to run tests as quickly as possible, using an SQLite in-memory database, use:

```bash
GTS_DB_TYPE="sqlite" GTS_DB_ADDRESS=":memory:" go test ./...
```

#### Postgres

If you want to run tests against a Postgres database running on localhost, run:

```bash
GTS_DB_TYPE="postgres" GTS_DB_ADDRESS="localhost" go test -p 1 ./...
```

In the above command, it is assumed you are using the default Postgres password of `postgres`.

#### Both

Finally, to run tests against both database types one after the other, use:

```bash
./scripts/test.sh
```

## Project Structure

For project structure, GoToSocial follows a standard and widely-accepted project layout [defined here](https://github.com/golang-standards/project-layout). As the author writes:

> This is a basic layout for Go application projects. It's not an official standard defined by the core Go dev team; however, it is a set of common historical and emerging project layout patterns in the Go ecosystem.

Most of the crucial business logic of the application is found inside the various packages and subpackages of the `internal` directory.

Where possible, we prefer more files and packages of shorter length that very clearly pertain to definable chunks of application logic, rather than fewer but longer files: if one `.go` file is pushing 1,000 lines of code, it's probably too long.

## Style

It is a good idea to read the short official [Effective Go](https://golang.org/doc/effective_go) page before submitting code: this document is the foundation of many a style guide, for good reason, and GoToSocial more or less follows its advice.

Another useful style guide that we try to follow: [this one](https://github.com/bahlo/go-styleguide).

In addition, here are some specific highlights from Uber's Go style guide which agree with what we try to do in GtS:

- [Group Similar Declarations](https://github.com/uber-go/guide/blob/master/style.md#group-similar-declarations).
- [Reduce Nesting](https://github.com/uber-go/guide/blob/master/style.md#reduce-nesting).
- [Unnecessary Else](https://github.com/uber-go/guide/blob/master/style.md#unnecessary-else).
- [Local Variable Declarations](https://github.com/uber-go/guide/blob/master/style.md#local-variable-declarations).
- [Reduce Scope of Variables](https://github.com/uber-go/guide/blob/master/style.md#reduce-scope-of-variables).
- [Initializing Structs](https://github.com/uber-go/guide/blob/master/style.md#initializing-structs).

## Linting and Formatting

Before you submit any code, make sure to run `go fmt ./...` to update whitespace and other opinionated formatting.

We use [golangci-lint](https://golangci-lint.run/) for linting, which allows us to catch style inconsistencies and potential bugs or security issues, using static code analysis.

If you make a PR that doesn't pass the linter, it will be rejected. As such, it's good practice to run the linter locally before pushing or opening a PR.

To do this, first install the linter following the instructions [here](https://golangci-lint.run/usage/install/#local-installation).

Then, you can run the linter with:

```bash
golangci-lint run
```

If there's no output, great! It passed :)

## Updating Swagger docs

GoToSocial uses [go-swagger](https://goswagger.io) to generate Swagger API documentation from code annotations.

You can install go-swagger following the instructions [here](https://goswagger.io/install.html).

If you change Swagger annotations on any of the API paths, you can generate a new Swagger file at `./docs/api/swagger.yaml` by running:

```bash
swagger generate spec -o docs/api/swagger.yaml --scan-models
```

## CI/CD configuration

GoToSocial uses [Drone](https://www.drone.io/) for CI/CD tasks like running tests, linting, and building Docker containers.

These runs are integrated with Github, and will be run on opening a pull request or merging into main.

The Drone instance for GoToSocial is [here](https://drone.superseriousbusiness.org/superseriousbusiness/gotosocial).

The `drone.yml` file is [here](./.drone.yml) -- this defines how and when Drone should run. Documentation for Drone is [here](https://docs.drone.io/).

Please note that the `drone.yml` file must be signed by the Drone admin account in order to be considered valid. This must be done every time the file is changed. This is to prevent tampering and hijacking of the Drone instance. See [here](https://docs.drone.io/signature/).

To sign the file, first install and setup the [drone cli tool](https://docs.drone.io/cli/install/). Then, run:

```bash
drone -t PUT_YOUR_DRONE_ADMIN_TOKEN_HERE -s https://drone.superseriousbusiness.org sign superseriousbusiness/gotosocial --save
```

## Building releases and Docker containers

### With GoReleaser

GoToSocial uses the release tooling [GoReleaser](https://goreleaser.com/intro/) to make multiple-architecture + Docker builds simple.

GoReleaser is also used by GoToSocial for building and pushing Docker containers.

Normally, these processes are handled by Drone (see CI/CD above). However, you can also invoke GoReleaser manually for things like building snapshots.

To do this, first [install GoReleaser](https://goreleaser.com/install/).

Then, to create snapshot builds, do:

```bash
goreleaser release --rm-dist --snapshot
```

If all goes according to plan, you should now have a bunch of multiple-architecture binaries and tars inside the `./dist` folder, and a snapshot Docker image should be built (check your terminal output for version).

### Manually

If you prefer a simple approach with fewer dependencies, you can also just build a Docker container manually in the following way:

```bash
./scripts/build.sh && docker build -t superseriousbusiness/gotosocial:latest .
```

## Financial Compensation

Right now there's no structure in place for financial compensation for pull requests and code. This is simply because there's no money being made on the project apart from the very small weekly Liberapay donations.

If money starts coming in, I'll start looking at proper financial structures, but for now code is considered to be a donation in itself.
