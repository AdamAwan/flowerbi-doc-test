---
title: Testing FlowerBI Queries and Schemas
status: draft
---

# Testing FlowerBI Queries and Schemas

Explicit testing ensures that your queries return the expected data, your schema definitions are correct, and that changes don't break existing logic. This page describes how to test both client‑side query code and server‑side schema/engine logic using the available tooling.

## Unit Testing Queries (TypeScript / React)

The core `flowerbi` package (and its React wrapper `flowerbi-react`) do not depend on a specific test runner, but the project itself uses **Jest** (as seen in `client/packages/flowerbi/package.json` where the `test` script runs `jest`). You can follow the same pattern:

1. **Create a mock fetch function** – Your queries rely on a `QueryFetch` function that POSTs JSON to an API. For testing, replace this with a mock that returns a canned `QueryResultJson`. For example, adapt the minimal example from the [`flowerbi/home.md` source](https://github.com/danielearwicker/flowerbi/blob/master/client/packages/flowerbi/home.md):

   ```typescript
   import { QueryJson, QueryResultJson } from 'flowerbi';

   async function mockFetch(queryJson: QueryJson): Promise<QueryResultJson> {
     // Return arrays matching your expected schema
     return [{ customer: 'Acme', bugCount: 3 }];
   }
   ```

2. **Write Jest tests** that call `useQuery` (or the underlying `query` function from `flowerbi`) with the mock fetch and assert the returned records.

```typescript
import { useQuery } from 'flowerbi-react';
import { renderHook } from '@testing-library/react-hooks';

it('returns expected record shape', async () => {
  const { result, waitForNextUpdate } = renderHook(() =>
    useQuery(mockFetch, {
      select: {
        customer: Customer.CustomerName,
        bugCount: Bug.Id.count()
      },
      filters: []
    })
  );
  await waitForNextUpdate();
  expect(result.current.records[0].customer).toBe('Acme');
  expect(result.current.records[0].bugCount).toBe(3);
});
```

3. **Run tests** – Use the existing project command:

```bash
# From client/packages/flowerbi (or your own package with jest config)
yarn test
```

If you are not using React, you can directly call the `query` function from the `flowerbi` package with a mock fetch.

## Testing Schema Definitions (YAML / C#)

Schemas are written in YAML and compiled to TypeScript and C#. To verify that a YAML file is valid and that generated code is correct, you can:

- **Load the schema in a test** using `Schema.FromYaml(yamlString)` from the `FlowerBI.Engine` NuGet package. The server‑side test project (`FlowerBI.Engine.Tests`) already does this – see the `.travis.yml` file which runs `dotnet test FlowerBI.Engine.Tests`. Add similar tests to your own project:

```csharp
using FlowerBI.Engine;
using Xunit;

public class SchemaTests
{
    [Fact]
    public void LoadsCustomSchema()
    {
        var yaml = @"
schema: MySchema
tables:
  Customer:
    id:
      Id: [int]
    columns:
      Name: [string]
";
        var schema = Schema.FromYaml(yaml);
        Assert.NotNull(schema.GetTable("Customer"));
    }
}
```

- **Validate generated TypeScript or C#** – The `FlowerBI.Tools` command line tool emits `.ts` and `.cs` files. You can run this as part of your build pipeline and then include those files in regular type checks (e.g. `tsc --noEmit`) or in a compilation step. The generated code includes JSDoc/XML doc comments that can be verified with static analysis.

## Integration Testing End‑to‑End

For a fuller test, you can spin up a test instance of your API (using the `FlowerBI.Engine` server component) and send real HTTP requests from a test suite. Because the query format is JSON, you can use any HTTP client (e.g. `supertest` in Node.js, or `HttpClient` in C#).

1. Start a test API (or use an in‑memory database like SQLite for quick checks).
2. POST a query JSON to the `/query` endpoint.
3. Assert that the returned `QueryResultJson` matches your expectations.

The demo site and Playground ([https://earwicker.com/flowerbi/demo/](https://earwicker.com/flowerbi/demo/)) let you manually test queries interactively – useful for exploratory testing before writing automated tests.

## Using the Playground for Quick Verification

The [FlowerBI Playground](https://earwicker.com/flowerbi/demo/) is an interactive environment where you can build a query graphically, see the generated TypeScript and SQL, and inspect the results. This is not a replacement for automated tests but is invaluable for debugging and understanding how your schema and queries interact.

## Summary of Testing Strategies

| Level               | Tool / Framework        | What to test                                      |
|---------------------|-------------------------|---------------------------------------------------|
| Unit (TypeScript)   | Jest                    | Query construction, mock fetch, record shapes     |
| Unit (C#)           | xUnit / NUnit           | Schema loading, join path resolution, filters     |
| Integration         | HTTP test library       | Full round‑trip: API → Engine → SQL → results     |
| Manual / Exploratory| Playground              | Interactive query building and debugging          |

By combining these approaches you can catch errors early and ensure your FlowerBI queries and schemas remain reliable as your application evolves.

## References

- [FlowerBI client package – home.md](https://github.com/danielearwicker/flowerbi/blob/master/client/packages/flowerbi/home.md) (example fetch function)
- [Package.json test script](https://github.com/danielearwicker/flowerbi/blob/master/client/packages/flowerbi/package.json) (`"test": "jest"`)
- [.travis.yml](https://github.com/danielearwicker/flowerbi/blob/master/.travis.yml) (runs `dotnet test` for server tests)
- [YAML schema reference](./yaml.md)
- [Playground](https://earwicker.com/flowerbi/demo/)