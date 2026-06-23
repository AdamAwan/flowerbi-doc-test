---
title: Verifying and Testing FlowerBI Requests
status: draft
---

# Verifying and Testing FlowerBI Requests

Ensuring your FlowerBI queries return correct results is critical. This guide covers several methods to verify that your data model and requests behave as expected, from interactive experimentation to automated unit tests.

## 1. Use the Playground for Interactive Verification

The [Playground](https://earwicker.com/flowerbi/demo/) lets you build queries interactively and see the generated SQL and results. This is the fastest way to check if a query produces the intended output. You can:

- Add columns and aggregations.
- Apply filters.
- View the generated TypeScript/JavaScript code.
- See the exact SQL that will be executed.
- Inspect the tabulated results.

This is ideal for prototyping and verifying query logic before writing code.

## 2. Unit Test Queries with Jest

The core `flowerbi` package includes Jest as a dev dependency and a `test` script (`jest`). You can write unit tests for your query functions or custom hooks. Example test structure:

```typescript
import { QueryJson } from 'flowerbi';
import { localFetch } from './fetch';

describe('MyQuery', () => {
  it('returns expected shape', async () => {
    const query: QueryJson = {
      select: ['Customer.CustomerName'],
      aggregations: [{ column: 'Bug.Id', function: 'Count' }],
      filters: []
    };
    const result = await localFetch(query);
    expect(result).toHaveLength(3);
    expect(result[0]).toHaveProperty('Customer.CustomerName');
    expect(result[0]).toHaveProperty('Value0');
  });
});
```

Run tests with `npm test` or `yarn test` in your project. The `flowerbi` package itself uses this approach – see its `package.json` (`"test": "jest"`) and the `server/dotnet/FlowerBI.Engine.Tests` project for server-side tests.

## 3. Use `useFlowerBI` Hook to Monitor Data and Errors

When building React components, the `useFlowerBI` hook (from `flowerbi-react`) manages query state. It returns `records`, `loading`, `error`, and `fetch` states. You can:

```tsx
import { useFlowerBI } from 'flowerbi-react';

function MyComponent() {
  const { records, loading, error } = useFlowerBI(fetch, {
    select: { customer: Customer.Name, count: Bug.Id.count() }
  });

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  // render records
}
```

Log `records` or `error` to the console during development to verify behavior. This is especially useful for verifying that filters and joins produce expected data.

## 4. Validate Schema Definitions with Generated Code

FlowerBI.Tools generates TypeScript and C# files from your YAML schema. These generated files provide type safety and early errors. To verify your schema:

- Run the code generation tool and check for any parsing errors (e.g., unknown table references, duplicate topics).
- Inspect the generated `.d.ts` or `.cs` files to ensure all tables, columns, and relationships are present.
- Use the strongly typed query builders – if a column name is misspelled, TypeScript will catch it at compile time.

Example generated TypeScript:

```typescript
/** @see {@link Topics.billing} */
export const Customer = {
  CustomerName: new StringQueryColumn<string>("Customer.CustomerName", ...),
  Id: new IntQueryColumn<number>("Customer.Id", ...),
};
```

## 5. Inspect Generated SQL

FlowerBI’s engine generates SQL from your query JSON. You can capture this SQL for debugging by:

- Using the Playground to see the SQL output.
- Adding server-side logging in your API before executing the query.
- If using `FlowerBI.Engine` in .NET, you can intercept the `ISqlBuilder` or log the generated string.

Example server-side logging (C#):

```csharp
var sql = engine.BuildSql(queryJson);
Console.WriteLine($"Generated SQL: {sql}");
```

Compare the SQL with expected joins and filters.

## 6. Use `FlowerBITable` for Quick Visual Verification

The `FlowerBITable` component (from `flowerbi-react`) renders query results as a plain HTML table. Drop it into a development page to see raw data:

```tsx
import { FlowerBITable } from 'flowerbi-react';

<FlowerBITable records={records} />
```

This gives you an immediate, unsophisticated view of the data, making it easy to spot missing records or incorrect aggregations.

## 7. Testing with Multiple Aggregations and Filters

When using multiple aggregations with different filters, enable `fullJoins` to avoid unexpected row loss (see [Full Joins](./full-joins.md)). Test both modes to confirm behavior.

Example test for multi-aggregation:

```ts
const query = {
  select: ['Customer.CustomerName'],
  aggregations: [
    { column: 'Bug.Id', function: 'Count' },
    { column: 'Bug.Id', function: 'Count', filters: [/* resolved filter */] }
  ],
  fullJoins: true
};
```

## Summary

| Method | When to Use |
|--------|-------------|
| Playground | Quick exploration and prototyping |
| Unit tests with Jest | Automated regression testing |
| `useFlowerBI` hook | Debugging data/errors in React components |
| Schema validation | Catch errors in data model definition |
| SQL inspection | Verify exact query execution |
| `FlowerBITable` | Ad-hoc visual inspection of results |

Combine these techniques to build confidence that your FlowerBI requests are correct.