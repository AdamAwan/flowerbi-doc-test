---
title: Testing FlowerBI Queries and Schemas
status: draft
---

# Testing FlowerBI Queries and Schemas

This guide covers strategies for verifying FlowerBI queries and schema definitions.

## Interactive Playground

Use the [Playground](https://earwicker.com/flowerbi/demo/) to build queries interactively, see generated SQL, and inspect results.

## Unit Tests (Jest)

Client packages use Jest. Write tests for query building and fetch logic.

```ts
import { query } from "flowerbi";

const mockFetch = async (json) => [];

test("query structure", async () => {
  const result = await query(mockFetch, {
    select: { customer: Customer.CustomerName, count: Bug.Id.count() }
  });
  expect(result.records).toEqual([]);
});
```

## Server Integration Tests

Run `dotnet test FlowerBI.Engine.Tests` to verify SQL generation and engine logic.

## Schema Validation

Call `Schema.FromYaml(yaml)` in a startup test to catch parsing errors.

## Visual Verification with FlowerBITable

Temporarily render `FlowerBITable` to see raw data during development.

## Best Practices

- Test with small known datasets.
- Validate multi-aggregation queries with `fullJoins: true`.
- Check generated SQL for expected joins and filters.

*Consolidated from: testing-flowerbi-queries-and-schemas.md, verifying-and-testing-flowerbi-queries-and-data-models.md, verifying-and-testing-flowerbi-requests.md, verifying-flowerbi-data-model-and-query-correctness.md*