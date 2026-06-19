---
title: Verifying and Testing FlowerBI Queries and Data Models
status: draft
---

# Verifying and Testing FlowerBI Queries and Data Models

Ensuring that your FlowerBI queries return correct and expected results requires a combination of interactive exploration, unit tests, and runtime inspection. This article covers the main approaches available to developers.

## 1. Interactive Exploration with the Playground

The [FlowerBI Playground](https://earwicker.com/flowerbi/demo/) lets you interactively build a query, see the generated JavaScript/TypeScript code, the resulting SQL, and the tabulated output. This is the fastest way to check that your schema definitions produce the correct joins, filters, and aggregations. You can experiment with different select columns, filter conditions, and aggregation functions to confirm behavior before writing code.

*Source: project README.md (under `## Use Case` and `## Live Demo`)*

## 2. Unit Testing the Server-Side Engine

The server-side `FlowerBI.Engine` library includes a dedicated test project (`FlowerBI.Engine.Tests`) that verifies SQL generation and query logic. To run these tests, execute:

```bash
cd server/dotnet
dotnet test FlowerBI.Engine.Tests/FlowerBI.Engine.Tests.csproj
```

These tests cover edge cases such as multiple aggregations, filter combinations, conjoint tables, and the `fullJoins` flag. When modifying your schema or adding new features, ensure the existing tests pass and consider adding new test cases that reflect your specific data model.

*Source: .travis.yml (test execution) and .vscode/launch.json (debug configuration)*

## 3. Client-Side Unit Tests with Jest

The client packages (`flowerbi` and `flowerbi-dates`) use Jest for unit testing. Run the test suite with:

```bash
cd client
yarn test
```

Individual packages can also be tested directly:

```bash
cd client/packages/flowerbi
yarn test
cd ../flowerbi-dates
yarn test
```

These tests validate core query building, type inference, and date handling logic. They run in watch mode during development for fast feedback.

*Source: package.json files for flowerbi and flowerbi-dates (under `"scripts": { "test": "jest" }`)*

## 4. Runtime Verification with React Hooks

When using `flowerbi-react`, the `useFlowerBI` hook returns an object that includes the current `records` (query results) and a `status` field. You can log these to the console or display them in a debug component to inspect the actual data returned from your API:

```tsx
import { useFlowerBI } from 'flowerbi-react';

function MyReport() {
  const { records, status } = useFlowerBI(fetch, query);
  console.log('Query status:', status);
  console.log('Records:', records);
  return <FlowerBITable records={records} />;
}
```

Check that `status` is `"loaded"` (or `"error"`) and that `records` contains the expected number of rows and correct values. This is especially useful during development to catch incorrect filters or join conditions.

*Source: react-hooks-useflowerbi-and-usepagefilters.md (`useFlowerBI` section)*

## 5. Visual Validation with FlowerBITable

The `FlowerBITable` component (from `flowerbi-react`) renders query results as a plain HTML table. Temporarily including this table in your report layout gives you a direct view of the raw data behind any chart. Compare the table rows with expected output from the Playground or your test database.

```tsx
import { FlowerBITable } from 'flowerbi-react';

<FlowerBITable records={records} />
```

*Source: building-table-layouts-with-flowerbi.md*)

## 6. Testing Multi-Aggregation Queries with Full Joins

If your query uses multiple aggregations (each with potentially different filters), the order of results can be affected by the join type. By default, segments are left-joined, which may drop rows that only appear in later aggregations. Enable `fullJoins: true` in the query to ensure symmetric behavior. Always test multi-aggregation queries with and without this flag to confirm you are getting the intended result set.

```ts
const query = {
  select: { colour: Colour.Name, countAll: Bug.Id.count(), countOpen: Bug.Id.count([Status.Resolved.equalTo(false)]) },
  fullJoins: true,
};
```

*Source: full-joins.md*)

## 7. End-to-End Integration Tests

For complete confidence, write integration tests that run the entire pipeline: client query → API → database → returned result. You can use the same Jest environment to fire queries against a test instance of your API and assert on the response. The demo site already does this implicitly (see `demo-site/` ).

## Summary

| Method                | When to Use                                    |
|-----------------------|------------------------------------------------|
| Playground            | Quick exploration of schema and query behavior |
| Server unit tests     | Validate SQL generation and schema mappings    |
| Client unit tests     | Verify query building and utility functions    |
| Runtime logs          | Debug a specific report in development         |
| FlowerBITable         | Visual inspection of raw data                  |
| Full joins testing    | Multi-aggregation correctness                  |

Start with the Playground, then add unit tests for your schema, and use runtime inspection to catch integration issues early.