---
title: "Server Setup: .NET Configuration and Query Endpoint"
status: draft
---

# Server Setup: .NET Configuration and Query Endpoint

This guide covers how to set up the FlowerBI server side in a .NET application. You will learn:

- How to install and reference the `FlowerBI.Engine` NuGet package
- How to load your YAML schema at startup
- How to configure dependency injection for the engine
- How to wire up a single POST endpoint that accepts client queries and returns results
- How to enforce row-level security

## Prerequisites

- A .NET 6.0+ or .NET Core 3.1+ project
- A SQL Server database (or other supported RDBMS – FlowerBI targets SQL Server by default)
- A FlowerBI YAML schema file (see [YAML Schemas](./yaml.md))

## 1. Install FlowerBI.Engine

Add the `FlowerBI.Engine` package to your project via the .NET CLI or Package Manager:

```bash
dotnet add package FlowerBI.Engine
```

> **Note:** The package is available on NuGet. Ensure you are using a compatible version.

## 2. Load the Schema and Configure the Connection

FlowerBI.Engine uses a `Schema` object parsed from your YAML file. You typically load it once at startup and reuse it for all queries.

Create a configuration section (e.g., `appsettings.json`) with your connection string:

```json
{
  "ConnectionStrings": {
    "FlowerBI": "Server=(localdb)\\mssqllocaldb;Database=MyDb;Trusted_Connection=True;"
  }
}
```

In your `Program.cs`, read the schema file and register the engine:

```csharp
using FlowerBI.Engine;

var builder = WebApplication.CreateBuilder(args);

// Load the YAML schema (adjust path as needed)
var schemaYaml = File.ReadAllText("schema.yaml");
var schema = Schema.FromYaml(schemaYaml);

// Register the schema as a singleton
builder.Services.AddSingleton(schema);

// Register a factory for creating DbConnection (using SqlClient)
builder.Services.AddTransient<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("FlowerBI")));

// Register the QueryEngine (transient is safe; it is stateless)
builder.Services.AddTransient<QueryEngine>();

var app = builder.Build();
```

## 3. Wire the Single POST Query Endpoint

FlowerBI exposes a single endpoint that accepts a JSON body conforming to `QueryJson` and returns a `QueryResultJson`. The minimal API approach is straightforward:

```csharp
app.MapPost("/query", async (QueryJson query, QueryEngine engine, IDbConnection db) =>
{
    // Execute the query and return the result
    var result = await engine.ExecuteAsync(db, query);
    // QueryEngine.ExecuteAsync returns a QueryResultJson (serializable to the client)
    return Results.Ok(result);
});
```

If you prefer a controller, create a `QueryController`:

```csharp
[ApiController]
[Route("[controller]")]
public class QueryController : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult<QueryResultJson>> Post([FromBody] QueryJson query,
        [FromServices] QueryEngine engine,
        [FromServices] IDbConnection db)
    {
        var result = await engine.ExecuteAsync(db, query);
        return Ok(result);
    }
}
```

Register the controller in `Program.cs`:

```csharp
builder.Services.AddControllers();
// ...
app.MapControllers();
```

## 4. What FlowerBI.Engine Does Internally

When you call `engine.ExecuteAsync(db, query)`:

1. **Parses the client query** (JSON with `select`, `aggregations`, `filters`, etc.)
2. **Resolves columns and tables** against the loaded schema
3. **Generates SQL** – builds the appropriate `SELECT`, `JOIN`, `GROUP BY`, `WHERE`, and `ORDER BY` clauses
4. **Executes the SQL** against the provided `IDbConnection`
5. **Transforms the results** into a `QueryResultJson` object (a list of records with strongly typed fields)

The engine never executes raw SQL from the client – only SQL it constructs based on the schema. This gives you full control over what columns and tables are accessible.

## 5. Adding Row-Level Security

The engine allows you to inject additional filters right before execution. This is useful for per-user authorization.

First, inject a service that determines the user's restrictions:

```csharp
public interface IUserContext
{
    int TenantId { get; }
}

// Example implementation reading from HttpContext
public class HttpUserContext : IUserContext
{
    private readonly IHttpContextAccessor _accessor;
    public HttpUserContext(IHttpContextAccessor accessor)
    {
        _accessor = accessor;
    }

    public int TenantId =>
        int.Parse(_accessor.HttpContext.User.FindFirst("tenant")?.Value ?? "0");
}
```

Then, before calling `ExecuteAsync`, amend the query:

```csharp
app.MapPost("/query", async (QueryJson query, QueryEngine engine, IDbConnection db,
    IUserContext user) =>
{
    // Add a filter to restrict to the user's tenant
    query.Filters.Add(new Filter
    {
        Column = "Tenant.Id",
        Operator = "=",
        Value = user.TenantId
    });

    var result = await engine.ExecuteAsync(db, query);
    return Results.Ok(result);
});
```

## 6. Full Example Program.cs (Minimal API)

```csharp
using FlowerBI.Engine;
using System.Data;
using System.Data.SqlClient;

var builder = WebApplication.CreateBuilder(args);

// Load schema
var schemaYaml = File.ReadAllText("schema.yaml");
var schema = Schema.FromYaml(schemaYaml);
builder.Services.AddSingleton(schema);

// Database connection
builder.Services.AddTransient<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("FlowerBI")));

// Engine
builder.Services.AddTransient<QueryEngine>();

// User context (optional)
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<IUserContext, HttpUserContext>();

var app = builder.Build();

app.MapPost("/query", async (QueryJson query, QueryEngine engine, IDbConnection db,
    IUserContext user) =>
{
    // Apply row-level security
    query.Filters.Add(new Filter { Column = "Tenant.Id", Operator = "=", Value = user.TenantId });

    var result = await engine.ExecuteAsync(db, query);
    return Results.Ok(result);
});

app.Run();
```

## 7. Next Steps

- See [YAML Schemas](./yaml.md) for how to define your schema
- See [Documenting your schema](./documentation.md) for adding metadata
- Explore the [Client-Side Fetch](./flowerbi/examples/client-fetch.md) to see how the client POSTs the query JSON

For more advanced topics (virtual tables, conjoint tables, many-to-many), refer to the respective documentation pages.
