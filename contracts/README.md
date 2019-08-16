# OVM Universal Adjudication Contracts
The Universal Adjudication contract serves as the single adjudicator for all layer 2 claims & disputes. This repository contains a standard implementation which can serve as a integration target for multiple layer 2 projects and constructions.

### Requirements and Setup
#### Cloning the Repo
```sh
git clone https://github.com/plasma-group/ovm.git
# enter the repository
cd ovm/contracts
```

#### Node.js
We test our contracts using [`Node.js`](https://nodejs.org/en/).
You'll need to install `Node.js` for your system before continuing.
We've provided a [detailed explanation of now to install `Node.js`](https://github.com/plasma-group/pigi/blob/c1c70a9ac6fe741fd937b9ca13ee7c1f6f9f4061/packages/docs/src/pg/src/reference/misc.rst#installing-node-js) on Windows, Mac, and Linux.

**Note**: This is confirmed to work on `Node.js v11.6`, but there may be issues on other versions. If you have trouble, please peg your Node.js version to 11.6.

#### Yarn
We're using a package manager called [Yarn](https://yarnpkg.com/en/).
You'll need to [install Yarn](https://yarnpkg.com/en/docs/install) before continuing.

#### Installing Dependencies
First let's install dependencies:
```sh
yarn install
```

### Building
Next let's build!
```sh
yarn run build
```

### Running Tests
To run contract tests use:

```sh
yarn test
```

### Linting
Clean code is the best code, so we've provided tools to automatically lint your projects.

Lint all packages:

```sh
yarn run lint
```

#### Automatically Fixing Linting Issues
We've also provided tools to make it possible to automatically fix any linting issues.
It's much easier than trying to fix issues manually.

Fix all packages:

```sh
yarn run fix
```

## Contributing
Welcome! If you're looking to contribute to the future of plasma, you're in the right place.

### Contributing Guide and Code of Conduct
Plasma Group follows a [Contributing Guide and Code of Conduct](https://github.com/plasma-group/pigi/blob/master/.github/CONTRIBUTING.md) adapted slightly from the [Contributor Covenant](https://www.contributor-covenant.org/version/1/4/code-of-conduct.html).
All contributors **must** read through this guide before contributing.
We're here to cultivate a welcoming and inclusive contributing environment, and every new contributor needs to do their part to uphold our community standards.

**Contributors: remember to run tests before submitting a pull request!**
Code with passing tests makes life easier for everyone and means your contribution can get pulled into this project faster.
