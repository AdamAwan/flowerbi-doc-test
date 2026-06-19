---
title: Server Setup: .NET Configuration and Query Endpoint
status: draft
---

# Server Setup: .NET Configuration and Query Endpoint

This guide covers how to set up the FlowerBI server side in a .NET application. You will learn:

- How to install and reference the `FlowerBI.Engine` NuGet package
- How to load your YAML schema at startup
- How to configure a single POST endpoint that accepts client queries and returns results
- Best practices for error handling and logging

## 1. Install FlowerBI.Engine

Add the `FlowerBI.Engine` package to your project via the .NET CLI or Package Manager:

```bash
dotnet add package FlowerBI.Engine
```

> **Note:** The package is available on NuGet. Ensure you are using a compatible version. For version 6.x, target .NET 6 or later.

## 2. Load the Schema and Configure the Connection

FlowerBI.Engine uses a `Schema` object parsed from your YAML file. You typically load it once at startup and reuse it for all queries.

Create a configuration section (e.g., `appsettings.json`) with your connection string:

```json
{
  "FlowerBI": {
    "ConnectionString": "Server=.;Database=MyDb;Trusted_Connection=true;"
  }
}
```

Then, in your `Startup.cs` or `Program.cs` (depending on your .NET version), load the schema and register the `QueryExecutor` as a singleton:

```csharp
using FlowerBI.Engine;

var builder = WebApplication.CreateBuilder(args);

// Load schema from YAML file (e.g., schema.yaml)
var yaml = File.ReadAllText("schema.yaml");
var schema = Schema.FromYaml(yaml);

// Register the executor with the schema and connection string
builder.Services.AddSingleton(sp => 
{
    var config = sp.GetRequiredService<IConfiguration>();
    var connectionString = config.GetConnectionString("FlowerBI:ConnectionString");
    return new QueryExecutor(schema, connectionString);
});

// Add controllers
builder.Services.AddControllers();
// ... other service configuration
```

> **Note:** The YAML file should be placed in the project root and set to copy to output directory. See the [YAML Schema documentation](./yaml.md) for schema syntax.

## 3. Create the Query Endpoint

Create a controller (e.g., `QueryController`) with a single POST action that deserializes the client's JSON query, executes it via `QueryExecutor`, and returns the result.

```csharp
using FlowerBI.Engine;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/query")]
public class QueryController : ControllerBase
{
    private readonly QueryExecutor _executor;

    public QueryController(QueryExecutor executor)
    {
        _executor = executor;
    }

    [HttpPost]
    public async Task<IActionResult> Post([FromBody] FlowerBI.Query query)
    {
        try
        {
            var result = await _executor.ExecuteQueryAsync(query);
            return Ok(result);
        }
        catch (Exception ex)
        {
            // Log error and return appropriate HTTP status
            return StatusCode(500, new { error = ex.Message });
        }
    }
}
```

The `FlowerBI.Query` type is defined in the `FlowerBI.Engine` package. It matches the JSON structure sent by the client (e.g., fields like `select`, `aggregations`, `filters`).

## 4. Client-side `fetch` Function

On the client side, define a function that POSTs the query JSON to your endpoint. This example uses the browser's `fetch` API:

```typescript
import { QueryJson, QueryResultJson } from "flowerbi";

export async function myFetch(queryJson: QueryJson): Promise<QueryResultJson> {
    const response = await fetch("http://localhost:5000/api/query", {
        method: "POST",
        cache: "no-cache",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(queryJson),
    });

    if (!response.ok) {
        throw new Error(`Query failed: ${response.statusText}`);
    }

    return await response.json();
}
```

This function can then be passed to `useQuery` (from `flowerbi-react`) or used directly with the core `executeQuery` function.

## 5. Security Considerations

- **Row-level security:** Add extra filters to the query inside the endpoint before execution, based on the authenticated user. The `QueryExecutor` accepts an optional `additionalFilters` parameter.
- **Restrict schema:** The server controls which tables/columns are exposed. Only those defined in the YAML schema can be queried.
- **Validate input:** Always validate the incoming query structure. The `FlowerBI.Engine` will reject malformed queries, but you may want to add early validation.

Example of adding row-level filters:

```csharp
var userId = HttpContext.User.FindFirst("UserId")?.Value;
var extraFilters = new List<Filter>
{
    new Filter("Tenant.TenantId", FilterOperator.Equal, userId)
};
var result = await _executor.ExecuteQueryAsync(query, extraFilters);
```

## 6. Error Handling and Logging

Wrap the `ExecuteQueryAsync` call in a try-catch and log exceptions. Common errors include:
- Invalid schema references in the query
- Missing or malformed YAML schema
- Database connection failures

Use structured logging (e.g., with Serilog) to capture request details without exposing sensitive data.

## 7. Testing Your Endpoint

You can test the endpoint using a tool like curl or Postman:

```json
POST /api/query
Content-Type: application/json

{
  "select": ["Customer.CustomerName"],
  "aggregations": [
    {
      "column": "Bug.Id",
      "function": "Count"
    }
  ],
  "filters": []
}
```

The response will be a JSON object with `columns` and `rows` properties, as defined by `QueryResultJson`.

## Next Steps

- See the [YAML Schemas documentation](./yaml.md) for defining your data model.
- See [Virtual Tables](./virtual-tables.md) for handling multiple uses of the same dimension.
- See [Documenting your schema](./documentation.md) for adding metadata to columns and tables.

For a full working example, refer to the [FlowerBI GitHub repository](https://github.com/danielearwicker/flowerbi) and the `Demo` project under `server/dotnet/Demo/`.