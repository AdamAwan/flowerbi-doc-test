---
title: FlowerBI External Dependencies
status: draft
---

# FlowerBI External Dependencies

This document lists the external runtime and development dependencies for the various packages that make up FlowerBI. Dependencies are inferred from the project's `package.json` files and other configuration files in the repository.

## Client-Side Packages (npm)

### Core package: `flowerbi`
- **Runtime dependencies**: `@types/json-stable-stringify`, `json-stable-stringify`
- **Peer dependencies**: none
- **Dev dependencies**: `@types/jest`, `jest`, `prettier`, `ts-jest`, `typedoc`, `typescript`

### React integration: `flowerbi-react`
- **Runtime dependencies**: `@types/json-stable-stringify`, `@types/react`, `flowerbi` (same repository), `json-stable-stringify`
- **Peer dependencies**: `react` (^16.13.1 || ^17.0.2 || ^18.0.0 || ^19.0.0)
- **Dev dependencies**: `prettier`, `react`, `typedoc`, `typescript`

### React utilities: `flowerbi-react-utils`
- **Runtime dependencies**: `@types/react`
- **Peer dependencies**: `react` (^16.13.1 || ^17.0.2 || ^18.0.0 || ^19.0.0)
- **Dev dependencies**: `prettier`, `react`, `typedoc`, `typescript`

### Date helpers: `flowerbi-dates`
- **Runtime dependencies**: none (uses `moment` as peer dependency)
- **Peer dependencies**: `moment` (^2.29.4)
- **Dev dependencies**: `@types/jest`, `jest`, `moment`, `prettier`, `ts-jest`, `typedoc`, `typescript`

### Demo site: `demo-site`
- **Runtime dependencies**:
  - `@types/blazor__javascript-interop`, `@types/chart.js`, `@types/node`, `@types/react`, `@types/react-dom`, `@types/sql.js`
  - `chart.js`
  - `flowerbi-react`, `flowerbi-react-utils`, `flowerbi-dates` (same repository)
  - `json-date-parser`
  - `moment`
  - `prettier`
  - `react`, `react-chartjs-2`, `react-dom`, `react-scripts`
  - `typescript`
- **Peer dependencies**: none (all listed as dependencies)
- **Dev dependencies**: none explicitly separate

## Server-Side Dependencies (.NET)

The server components (FlowerBI.Engine, FlowerBI.Tools, etc.) are built with:
- `.NET Core 3.1` / `.NET 6.0` runtime (as observed in `.travis.yml` and `launch.json`)
- Tools: `csharpier` (formatting), `dotnet-test`
- NuGet packages (inferred from `Fsdk` usage): `Newtonsoft.Json` (likely), others unknown without `.csproj` files

## Development Tools
- **TypeScript** (^4.5.5)
- **Jest** (^27.4.7)
- **ts-jest** (^27.1.3)
- **Prettier** (^2.5.1)
- **Typedoc** (^0.22.11)
- **React Scripts** (^5.0.0, for demo site)
- **.NET SDK** (3.1 / 6.0)
- **CSharpier** (1.2.5)

## Notes
- The `flowerbi` core package has no external peer dependencies, making it lightweight for non-React applications.
- The demo site includes many type definitions for development that are not required for production use.
- This list may not be exhaustive; refer to the actual `package.json` files in the repository for the most accurate information.