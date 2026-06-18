---
title: Contributing to FlowerBI: Developer Workflow
status: draft
---

# Contributing to FlowerBI: Developer Workflow

Thank you for your interest in contributing to FlowerBI! This document describes the development workflow for the project. FlowerBI is open source under the [MIT license](https://github.com/danielearwicker/flowerbi/blob/master/LICENSE), and we welcome contributions of all kinds — bug fixes, features, documentation, and more.

## Repository Structure

The repository is organized as follows:

- **`server/dotnet/`** – The .NET backend (FlowerBI.Engine, tools, and tests).
- **`client/`** – A Yarn workspace containing all npm packages:
  - `packages/flowerbi` – Core client library.
  - `packages/flowerbi-react` – React hooks and components.
  - `packages/flowerbi-react-utils` – Additional React utilities.
  - `packages/flowerbi-dates` – Date helpers.
  - `packages/demo-site` – The demo/playground application.
- **`docs/`** – Markdown documentation and generated API docs.
- **`.travis.yml`** – CI configuration (used for building and testing).
- **`.config/dotnet-tools.json`** – Local .NET tools, e.g. [CSharpier](https://csharpier.com/) for formatting.

## Prerequisites

- **.NET SDK 6.0** (or later) – for building the server components.
- **Node.js** (v16 or later) and **Yarn** (v1) – for client packages.
- A code editor of your choice (VS Code, Visual Studio, Rider, etc.).

## Building and Testing the .NET Engine

All commands should be run from the `server/dotnet` directory.

1. Restore dependencies:
   ```bash
   dotnet restore
   ```
2. Build the solution:
   ```bash
   dotnet build
   ```
3. Run the tests:
   ```bash
   dotnet test FlowerBI.Engine.Tests/FlowerBI.Engine.Tests.csproj
   ```

The CI pipeline also runs a version script (`node apply-version.js`) before building – you can ignore this for local development unless you're cutting a release.

## Building and Testing npm Packages

All commands should be run from the `client` directory (the root of the Yarn workspace).

1. Install dependencies:
   ```bash
   yarn
   ```
2. Build all packages (TypeScript compilation):
   ```bash
   yarn workspaces run build
   ```
3. Run tests for packages that have them (e.g. `flowerbi` and `flowerbi-dates`):
   ```bash
   yarn workspace flowerbi test
   yarn workspace flowerbi-dates test
   ```
4. To build the demo site in development mode:
   ```bash
   cd packages/demo-site && yarn start
   ```
   This runs the React app at [http://localhost:3000](http://localhost:3000).

### Individual Package Commands

Each npm package defines its own scripts in `package.json`. Common ones include:

| Script       | Description                                  |
|--------------|----------------------------------------------|
| `build`      | Compile TypeScript (tsc).                    |
| `watch`      | Watch mode for TypeScript compilation.       |
| `test`       | Run Jest tests.                              |
| `docs`       | Generate TypeDoc documentation.              |
| `fbi-release`| Custom script for package release (test+build+docs). |

## Code Style and Formatting

- **C#** code is formatted using [CSharpier](https://csharpier.com/) (version 1.2.5). Run `dotnet csharpier .` from `server/dotnet` before committing.
- **TypeScript/JavaScript** code uses [Prettier](https://prettier.io/) (version 2.5.1). Run `prettier --write .` from the `client` directory or your editor’s Prettier integration.
- We recommend enabling editorconfig and the relevant formatting extensions in your IDE.

## Branch Conventions and Pull Requests

- The main development branch is `master`. All changes should be made on feature branches and submitted as pull requests to `master`.
- Branch names should be descriptive, e.g. `fix/issue-123`, `feat/add-widget`, `docs/update-readme`.
- Keep commits atomic and write clear commit messages.
- Before opening a PR, ensure:
  - All tests pass locally.
  - Code is formatted (CSharpier/Prettier).
  - New features include appropriate tests or documentation.
- PRs will be reviewed by maintainers. You may be asked to make changes before merging.

## CI/CD

The project uses Travis CI (see `.travis.yml`). On pushes to `master`, the CI builds both .NET and client projects, runs all tests, and if successful, triggers a release script (`release.sh`).

## Release Process

Releases are manually triggered by maintainers via the `fbi-release` scripts defined in each package. Typically a version bump is performed across all packages to keep them in sync. The release script publishes npm packages and deploys updated documentation.

## Getting Help

If you have questions or run into issues, please open a [GitHub Issue](https://github.com/danielearwicker/flowerbi/issues) or start a discussion. We're happy to help!