---
title: "Integrating FlowerBI into a .NET / React Project"
status: draft
---

# Integrating FlowerBI into a .NET / React Project

This guide walks through adding FlowerBI to an existing ASP.NET Core backend and a React (TypeScript) frontend. FlowerBI allows you to describe a database schema in YAML, then execute client‑driven analytical queries from your React app via a single API endpoint.

## Prerequisites

- A .NET 6.0+ project with access to a SQL Server or compatible database
- A React project (e.g., created with Create React App) using TypeScript
- Basic familiarity with YAML, npm, and ASP.NET Core middleware

## Step 1: Install Server‑Side Packages

Add the **FlowerBI.Engine** NuGet package to your ASP.NET Core project:

```bash
dotnet add package FlowerBI.Engine
```

If you want code generation from YAML schemas (recommended), also add:

```bash
dotnet add package FlowerBI.Tools
```

## Step 2: Define Your Schema in YAML

Create a YAML file (e.g., `schema.yaml`) that describes your database tables, primary keys, foreign keys, and columns. Place it in your project root or a `Schemas` folder. Example:

```yaml
schema: MyApp

tables:
  Customer:
    id:
      Id: [int]
    columns:
      CustomerName: [string]
      CreatedDate: [datetime]

  Order:
    id:
      Id: [int]
    columns:
      CustomerId: [Customer]
      Amount: [decimal]
      OrderDate: [datetime]
```

For a full reference see [YAML Schemas](./yaml.md).

## Step 3: Register FlowerBI.Engine in Your .NET API

In `Program.cs` (or `Startup.cs`), read the YAML schema, load it into a `Schema` object, and register the `FlowerBIEngine` service:

```csharp
using FlowerBI.Engine;

var schemaYaml = File.ReadAllText("Schemas/schema.yaml");
var schema = Schema.FromYaml(schemaYaml);

builder.Services.AddSingleton(schema);
```

Then create an API endpoint (e.g., `/api/query`) that accepts a JSON payload and returns results:

```csharp
app.MapPost("/api/query", async (QueryJson query, FlowerBIEngine engine, HttpContext context) =>
{
    // Optional: apply row-level security filters based on the authenticated user
    // engine.AddFilter(...);

    var result = await engine.ExecuteAsync(query);
    return Results.Json(result);
});
```

The `FlowerBIEngine` class is part of the `FlowerBI.Engine` namespace. It parses the query, generates SQL, executes it against your database (requires a configured `IDbConnection`), and returns the result set.

You will need to inject the `IDbConnection` for your database (e.g., using Dapper). A simple approach:

```csharp
builder.Services.AddTransient<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("DefaultConnection")));
```

**Important:** The engine does not manage connections itself – you need to pass a connection factory or use a scoped service that provides it. Refer to the [FlowerBI.Engine documentation](https://github.com/danielearwicker/flowerbi) for more details.

## Step 4: Generate TypeScript Client Code (Optional but Recommended)

Using **FlowerBI.Tools**, generate strongly‑typed TypeScript constants and helper types from your YAML schema. Add a script to your project:

```bash
dotnet fbi generate-ts --input Schemas/schema.yaml --output ../client/src/generated/schema.ts
```

This creates a file you can import in your React app, providing intellisense for table and column names.

## Step 5: Install Client‑Side Packages

In your React project, install the core `flowerbi` package and the React helpers:

```bash
npm install flowerbi flowerbi-react
```

Optional for chart integration:

```bash
npm install flowerbi-react-chartjs chart.js react-chartjs-2
```

## Step 6: Create a Fetch Function

Define a function that posts a `QueryJson` to your API endpoint. This function will be passed to the `useQuery` hook.

```tsx
// api/fetchQuery.ts
import { QueryJson, QueryResultJson } from 'flowerbi';
import jsonDateParser from 'json-date-parser';

export async function fetchQuery(queryJson: QueryJson): Promise<QueryResultJson> {
    const response = await fetch('/api/query', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(queryJson),
    });

    if (!response.ok) throw new Error('Query failed');

    return JSON.parse(await response.text(), jsonDateParser);
}
```

## Step 7: Use the Hook in a React Component

Now you can use the `useQuery` hook from `flowerbi-react` to fetch data and render it:

```tsx
import { useQuery } from 'flowerbi-react';
import { Customer, Order } from './generated/schema';

function CustomerReport() {
    const { records, loading, error } = useQuery(fetchQuery, {
        select: {
            customer: Customer.CustomerName,
            totalAmount: Order.Amount.sum(),
        },
        filters: [],
    });

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;

    return (
        <table>
            <thead>
                <tr><th>Customer</th><th>Total Amount</th></tr>
            </thead>
            <tbody>
                {records.map((r, i) => (
                    <tr key={i}>
                        <td>{r.customer}</td>
                        <td>{r.totalAmount}</td>
                    </tr>
                ))}
            </tbody>
        </table>
    );
}
```

The `useQuery` hook automatically re‑fetches when the query definition changes (thanks to `json-stable-stringify`). You can also use `usePageFilters` for shared filtering across multiple charts.

## Next Steps

- Learn about [virtual tables](./virtual-tables.md) for handling multiple date dimensions.
- Explore the [FlowerBI React utilities](https://github.com/danielearwicker/flowerbi/tree/master/client/packages/flowerbi-react-utils) for building report layouts.
- Read about [conjoint tables](./conjoint.md) for advanced annotation patterns.

## Troubleshooting

| Problem | Likely Solution |
|---------|----------------|
| “No connection factory registered” | Ensure you have registered an `IDbConnection` in DI. |
| Query returns unexpected results | Check your YAML foreign keys and nullable markers. |
| TypeScript errors in generated schema | Run the code generation tool again after schema changes. |

For more details, see the [FlowerBI README](https://github.com/danielearwicker/flowerbi).
