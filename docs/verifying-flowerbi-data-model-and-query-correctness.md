---
title: Verifying FlowerBI Data Model and Query Correctness
status: draft
---

# Verifying FlowerBI Data Model and Query Correctness

FlowerBI does not ship a dedicated “validate” tool, but the ecosystem provides several practical methods to ensure your YAML schema, query definitions, and generated SQL behave correctly. Use a combination of these approaches as your project grows.

## 1. Validate the YAML schema itself

Before using your schema, confirm that it parses without errors and that all foreign-key references resolve to existing tables.

- **`Schema.FromYaml(yaml)`** – In C# server code, call `Schema.FromYaml(...)`. It raises exceptions for unknown table/column references, missing primary keys, or invalid type names. Catch this exception in a startup test.
- **Generated TypeScript/C#** – Run `FlowerBI.Tools` to produce client-side code. If the generation succeeds without warnings, the YAML is structurally valid.

### Example startup test (C#)
```csharp
[Fact]
public void Schema_IsValid()
{
    var yaml = File.ReadAllText("mySchema.yaml");
    var schema = Schema.FromYaml(yaml); // throws on invalid
    Assert.NotNull(schema);
}
```

## 2. Use the Playground to interactively verify queries

The [Playground](https://earwicker.com/flowerbi/demo/) lets you build a query, see the JavaScript/TypeScript code, inspect the generated SQL, and view tabulated results. This is the fastest way to check that your query’s joins, filters, and aggregations produce the expected output.

- **Verify joins** – Select columns from two tables that should join automatically. If the SQL shows no join or an unexpected cross join, your foreign keys may be missing in the schema.
- **Verify filters** – Add a filter and confirm the SQL’s `WHERE` clause reflects it correctly.
- **Verify aggregations** – Check that `COUNT`, `SUM`, `AVERAGE` etc. appear in the SQL’s `SELECT` and `GROUP BY` clauses.

## 3. Write automated tests against known data

Load a small, representative dataset into a test database (or use an in‑memory SQLite instance as the demo does) and write tests that execute typical queries.

- Use the same `useFlowerBI` hook or the raw `query` function with a mock fetch that calls your test server.
- Assert on the shape and values of the returned records.

### Example (Jest with `flowerbi` package)
```ts
import { query } from "flowerbi";

const fakeFetch = async (json: QueryJson): Promise<QueryResultJson> => {
    // call your test API endpoint or a local server
};

test("customer bug count", async () => {
    const result = await query(fakeFetch, {
        select: {
            customer: Customer.CustomerName,
            bugCount: Bug.Id.count(),
        },
    });
    expect(result.records).toHaveLength(3);
    expect(result.records[0].customer).toBe("Acme");
});
```

## 4. Validate generated SQL manually or via a SQL linter

The Playground shows the SQL, but you can also capture it by adding logging to your API. For critical queries, paste the SQL into your database management tool and run it with `EXPLAIN` or `EXPLAIN ANALYZE` to check performance and correctness.

- **Check for Cartesian products** – If you see multiple `CROSS JOIN`s where you expect `INNER JOIN`s, your schema may be missing intermediate tables (see [Many‑to‑Many](many-to-many.md) and `associative` hints).
- **Check for missing joins** – If a column from a dimension appears but the SQL does not join that table, the schema may be relying on an indirect path that produces incorrect results.

## 5. Enable `fullJoins` for multi‑aggregation queries

If your query contains multiple aggregations (e.g. `countAllBugs` and `countResolvedBugs`), set `fullJoins: true` to avoid asymmetric loss of rows. This flag was introduced to fix a known issue where `LEFT JOIN` could drop rows when aggregation subsets don’t perfectly overlap. Test both with and without the flag to verify expected output.

```ts
const result = await query(fetch, {
    select: { ... },
    aggregations: [ ... ],
    fullJoins: true,
});
```

See [Full Joins](full-joins.md) for details.

## 6. Test edge cases with virtual and conjoint tables

- **Virtual tables** – If you use `extends` to create date variants, verify that two different date filters (e.g. `DateRequested` vs. `DateCompleted`) produce independent joins and do not interfere.
- **Conjoint tables** – For annotation patterns, test that multiple conjoint instances with different `@suffix` values return distinct rows. Use the Playground to see the generated SQL and confirm the table aliases are separate.

## 7. Run the existing test suite

Both the client and server packages include unit tests. Run them to verify the FlowerBI engine itself behaves correctly in your environment.

```bash
# Client (Node.js)
cd client/packages/flowerbi
npm test

# Server (.NET)
dotnet test FlowerBI.Engine.Tests
```

These tests cover query parsing, SQL generation, and result mapping. They serve as a reference for how queries should be structured and what outputs to expect.

## Summary checklist

- [ ] Schema parses without exception in `Schema.FromYaml`.
- [ ] Client code generation completes without errors.
- [ ] Playground shows correct SQL and results for representative queries.
- [ ] Automated tests with known data pass.
- [ ] Generated SQL does not contain unexpected cross joins.
- [ ] Multi‑aggregation queries use `fullJoins: true` or the behavior is explicitly tested.
- [ ] Virtual/conjoint tables produce independent joins in SQL.
- [ ] Existing unit tests pass.

By combining these techniques you can confidently deploy and evolve your FlowerBI data model.
