---
title: Server Setup: .NET Configuration and Query Endpoint
status: draft
---

# Server Setup: .NET Configuration and Query Endpoint

This guide covers setting up the FlowerBI server side in .NET.

## Install NuGet Package

```bash
dotnet add package FlowerBI.Engine
```

## Load Schema and Configure Connection

In `appsettings.json`:
```json
{
  "FlowerBI": {
    "SchemaPath": "schema.yaml",
    "ConnectionString": "Server=.;Database=MyDb;Trusted_Connection=true;"
  }
}
```

In startup code:
```csharp
var yaml = File.ReadAllText(config["FlowerBI:SchemaPath"]);
var schema = Schema.FromYaml(yaml);
services.AddSingleton(schema);
```

## Create the Query Endpoint

### Minimal API (recommended)
```csharp
app.MapPost("/query", async (QueryJson query, Schema schema, IConfiguration config) =>
{
    // Optionally add RLS filters
    query.Filters.Add(new Filter { Column = "Tenant.Id", Operator = "=", Value = tenantId });
    var connectionString = config.GetConnectionString("FlowerBI");
    using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync();
    var sql = schema.BuildQuery(query);
    // Execute and return results
});
```

### Controller
```csharp
[ApiController]
[Route("api/[controller]")]
public class QueryController : ControllerBase { ... }
```

## Error Handling and Logging

Wrap execution in try-catch and log exceptions.

*Consolidated from: server-setup-net-configuration-and-query-endpoint.md, server-setup-net-query-endpoint-configuration.md*