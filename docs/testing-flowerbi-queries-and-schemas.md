---
title: Testing FlowerBI Queries and Schemas
status: draft
---

# Testing FlowerBI Queries and Schemas

This guide covers strategies and tools for testing FlowerBI queries and schema definitions. FlowerBI projects typically consist of a TypeScript/React client (containing query definitions) and a .NET server (containing the schema and query engine). Both sides have established testing patterns using frameworks already present in the project's toolchain.

## Client-Side Testing with Jest

The core `flowerbi` package and the demo site both include Jest as a test runner. You can write unit tests for your query definitions and helper functions. The `flowerbi` package's `package.json` contains:

```json
"scripts": {
    "test": "jest"
}
```

To test a query, create a test file (e.g., `myQuery.test.ts`) and use Jest's `expect` to verify the structure of the query JSON or the results after processing. Since FlowerBI queries are plain objects, you can assert their shape:

```ts
import { QueryJson } from 'flowerbi';

describe('My Query', () => {
    it('should select customer name and count of bugs', () => {
        const query: QueryJson = {
            select: ["Customer.CustomerName"],
            aggregations: [{ column: "Bug.Id", function: "Count" }]
        };
        expect(query.select).toContain("Customer.CustomerName");
        expect(query.aggregations?.length).toBe(1);
    });
});
```

For React components using `useQuery`, you can use testing libraries like `@testing-library/react` along with mocked fetch functions, as shown in the `flowerbi` package's home documentation ([flowerbi/home.md](https://github.com/danielearwicker/flowerbi/blob/main/client/packages/flowerbi/home.md)). A typical mock fetch returns a `QueryResultJson` array.

## Integration Testing with the Server Engine

The server side, built on .NET Core, includes an engine (`FlowerBI.Engine`) that executes queries against a database. The project has a test suite (`FlowerBI.Engine.Tests`) run via `dotnet test`. The `.travis.yml` file shows this:

```yml
script:
    - pushd server/dotnet
    - dotnet build
    - dotnet test FlowerBI.Engine.Tests/FlowerBI.Engine.Tests.csproj
    - popd
```

These tests verify that generated SQL is correct and that aggregations, joins, filters, and schema features (like virtual tables or conjoint tables) work as expected. You can extend these tests by adding new test cases for your custom schema. The test project is located at `server/dotnet/FlowerBI.Engine.Tests/`. To run them locally:

```bash
cd server/dotnet
dotnet test FlowerBI.Engine.Tests
```

## Ad-Hoc Testing with the Playground

FlowerBI provides a [Playground](https://earwicker.com/flowerbi/demo/) that lets you build queries interactively, view the generated SQL, and see tabulated results. This is an excellent tool for quick validation of query logic without writing any code. You can copy the generated JavaScript/TypeScript query definition into your client code. The playground runs the entire stack in-browser using WASM ([demo site README](https://github.com/danielearwicker/flowerbi/blob/main/client/packages/demo-site/README.md)).

## Validating YAML Schemas

Schema definitions (YAML files) can be validated by loading them with `Schema.FromYaml(...)` in a test. The `FlowerBI.Engine` package exposes methods to parse and validate schemas. You can write a test that loads your YAML and asserts that tables and columns exist, foreign keys are correct, and documentation is present. Example:

```csharp
var yaml = File.ReadAllText("schema.yaml");
var schema = Schema.FromYaml(yaml);
Assert.NotNull(schema.GetTable("Invoice"));
Assert.True(schema.GetTable("Invoice").GetColumn("Amount").IsNumeric);
```

## Best Practices

- **Test both success and failure paths**: Ensure queries return expected data and handle empty results gracefully (see [Troubleshooting Common Issues](troubleshooting-common-issues-in-flowerbi-queries.md) for empty result sets).
- **Use `fullJoins` for multi-aggregation queries**: As described in [Full Joins](full-joins.md), enable this flag to avoid unexpected row omissions.
- **Mock the fetch function**: Isolate client tests by mocking the POST request, using a sample `QueryResultJson`.
- **Run tests in CI**: The `.travis.yml` configuration shows how to run both client and server tests during deployment.

## Quiz: Testing Your Knowledge

1. What command runs the server-side tests in the FlowerBI project? **Answer:** `dotnet test FlowerBI.Engine.Tests`
2. Which test runner is used for the client-side JavaScript packages? **Answer:** Jest
3. Where can you interactively test a query without writing any code? **Answer:** The [FlowerBI Playground](https://earwicker.com/flowerbi/demo/)

## References

- `flowerbi` package test script: [client/packages/flowerbi/package.json](https://github.com/danielearwicker/flowerbi/blob/main/client/packages/flowerbi/package.json)
- Demo site test script: [client/packages/demo-site/package.json](https://github.com/danielearwicker/flowerbi/blob/main/client/packages/demo-site/package.json)
- .NET test project in CI: [.travis.yml](https://github.com/danielearwicker/flowerbi/blob/main/.travis.yml)
- Playground: [https://earwicker.com/flowerbi/demo/](https://earwicker.com/flowerbi/demo/)
- FlowerBI home documentation for fetch mocking: [client/packages/flowerbi/home.md](https://github.com/danielearwicker/flowerbi/blob/main/client/packages/flowerbi/home.md)