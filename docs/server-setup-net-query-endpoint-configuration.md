---
title: "Server Setup: .NET Query Endpoint Configuration"
status: draft
---

# Server Setup: .NET Query Endpoint Configuration

This guide explains how to set up the FlowerBI server side in a .NET application, focusing on wiring up the single POST endpoint that accepts client queries and returns results.

## 1. Install the FlowerBI.Engine NuGet Package

Add the `FlowerBI.Engine` package to your project via the .NET CLI or Package Manager:

```bash
dotnet add package FlowerBI.Engine
```

> **Note:** The package is available on NuGet. Ensure you are using a compatible version (e.g., .NET 6 or later).

## 2. Load the YAML Schema and Configure the Database Connection

FlowerBI.Engine uses a `Schema` object parsed from your YAML schema file. Load it once at application startup and reuse it for all queries.

Create a configuration section (e.g., in `appsettings.json`) with your connection string and the path to your schema file:

```json
{
  "FlowerBI": {
    "SchemaPath": "schema.yaml",
    "ConnectionString": "Server=localhost;Database=mydb;Trusted_Connection=True;"
  }
}
```

Then, in your startup code, load the schema and configure a singleton service:

```csharp
using FlowerBI.Engine;
using Microsoft.Extensions.DependencyInjection;

// In Program.cs or Startup.cs
var schemaPath = configuration.GetValue<string>("FlowerBI:SchemaPath");
var connectionString = configuration.GetValue<string>("FlowerBI:ConnectionString");

var yaml = File.ReadAllText(schemaPath);
var schema = Schema.FromYaml(yaml);

services.AddSingleton(schema);
services.AddSingleton(new ConnectionConfig(connectionString));
```

> **Note:** The `ConnectionConfig` class is a simple wrapper you can define to hold the connection string. Alternatively, you can inject `IConfiguration` directly into your endpoint handler.

## 3. Create the POST Endpoint

Use Minimal API or a Controller to expose a single POST route that accepts the client's query JSON and returns the result JSON.

### Using Minimal API (Recommended for .NET 6+)

```csharp
// Program.cs
app.MapPost("/query", async (QueryJson query, FlowerBiEngine engine) =>
{
    var result = await engine.Execute(query);
    return Results.Json(result);
});
```

You need to register the `FlowerBiEngine` as a service. Create a simple wrapper class that uses the schema and connection string:

```csharp
public class FlowerBiEngine
{
    private readonly Schema _schema;
    private readonly string _connectionString;

    public FlowerBiEngine(Schema schema, ConnectionConfig config)
    {
        _schema = schema;
        _connectionString = config.ConnectionString;
    }

    public async Task<QueryResultJson> Execute(QueryJson query)
    {
        using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        var sql = _schema.BuildQuery(query);
        // Execute SQL and map results to QueryResultJson
        // (See below for helper method)
    }
}
```

### Using a Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class QueryController : ControllerBase
{
    private readonly Schema _schema;
    private readonly IConfiguration _configuration;

    public QueryController(Schema schema, IConfiguration configuration)
    {
        _schema = schema;
        _configuration = configuration;
    }

    [HttpPost]
    public async Task<IActionResult> Post([FromBody] QueryJson query)
    {
        var connectionString = _configuration.GetConnectionString("FlowerBI");
        using var connection = new SqlConnection(connectionString);
        await connection.OpenAsync();

        var sql = _schema.BuildQuery(query);
        // Execute and return result
    }
}
```

## 4. Execute the Query and Return Results

The `Schema.BuildQuery` method generates a SQL string. You need to execute it and convert the returned rows into a `QueryResultJson` object. The following helper method demonstrates this for SQL Server:

```csharp
public static async Task<QueryResultJson> ExecuteQuery(Schema schema, QueryJson query, SqlConnection connection)
{
    var sql = schema.BuildQuery(query);
    using var command = new SqlCommand(sql, connection);
    using var reader = await command.ExecuteReaderAsync();

    var results = new List<Dictionary<string, object>>();
    while (await reader.ReadAsync())
    {
        var row = new Dictionary<string, object>();
        for (int i = 0; i < reader.FieldCount; i++)
        {
            row[reader.GetName(i)] = reader.GetValue(i);
        }
        results.Add(row);
    }

    return new QueryResultJson
    {
        Columns = query.Select.Select(c => c.Alias ?? c.Column).ToArray(),
        Rows = results
    };
}
```

> **Note:** The exact mechanics of `Schema.BuildQuery` and the result types are part of the `FlowerBI.Engine` API. Check the latest API documentation for details.

## 5. Adding Row-Level Security (Optional)

Your server can inject additional filters (e.g., user ID or tenant) before building the query. For example, assuming you have a user context:

```csharp
var userFilter = new Filter
{
    Column = "Tenant.Id",
    Operator = "=",
    Value = currentUser.TenantId
};
query.Filters.Add(userFilter);
```

Do this before calling `schema.BuildQuery(query)`.

## 6. Full Example: Minimal API Program.cs

```csharp
using FlowerBI.Engine;
using Microsoft.Data.SqlClient;
using System.Data;

var builder = WebApplication.CreateBuilder(args);

// Load schema
var schemaYaml = File.ReadAllText(builder.Configuration["FlowerBI:SchemaPath"]);
var schema = Schema.FromYaml(schemaYaml);
builder.Services.AddSingleton(schema);

var app = builder.Build();

app.MapPost("/query", async (QueryJson query, Schema schema, IConfiguration config) =>
{
    var connectionString = config.GetConnectionString("FlowerBI");
    using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync();

    // Optionally add row-level security filter
    // query.Filters.Add(...);

    var sql = schema.BuildQuery(query);
    using var command = new SqlCommand(sql, connection);
    using var reader = await command.ExecuteReaderAsync();

    var columns = new List<string>();
    var rows = new List<Dictionary<string, object>>();
    while (await reader.ReadAsync())
    {
        if (columns.Count == 0)
        {
            for (int i = 0; i < reader.FieldCount; i++)
                columns.Add(reader.GetName(i));
        }
        var row = new Dictionary<string, object>();
        for (int i = 0; i < reader.FieldCount; i++)
            row[columns[i]] = reader.GetValue(i);
        rows.Add(row);
    }

    return Results.Json(new { columns, rows });
});

app.Run();
```

## 7. Client-Side Fetch Function (for reference)

On the client, write a function that conforms to the `QueryFetch` type, as shown in the `flowerbi` package home page:

```typescript
import { QueryJson, QueryResultJson } from "flowerbi";

export async function localFetch(queryJson: QueryJson): Promise<QueryResultJson> {
    const response = await fetch("http://localhost:5000/query", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(queryJson),
    });
    if (!response.ok) return [];
    return await response.json();
}
```

## Next Steps

- Consult the [YAML Schemas](./yaml.md) documentation for defining your data model.
- Learn about [Virtual Tables](./virtual-tables.md) for handling multiple date dimensions.
- Review the [Documentation](./documentation.md) guide for annotating your schema.
