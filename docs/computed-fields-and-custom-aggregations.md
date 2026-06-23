---
title: Computed Fields and Custom Aggregations
status: draft
---

# Computed Fields and Custom Aggregations

This document explains how to define derived (computed) columns in your YAML schema and how to extend FlowerBI with custom aggregation functions beyond the built-in `count`, `sum`, `avg`, etc. Both features are optional but powerful when your data model requires calculations that don't exist directly in the source tables.

## 1. Computed / Derived Fields in the YAML Schema

A computed field is a column that is not stored in the physical database but is calculated from other columns at query time. You define it in the `tables:` section using the `computed` keyword (or a special syntax, depending on the version).

### Syntax

```yaml
ExampleTable:
  id:
    Id: [int]
  columns:
    Quantity: [int]
    UnitPrice: [decimal]
  computed:
    TotalPrice:
      expression: "Quantity * UnitPrice"
      type: decimal
      doc: Extended price (quantity times unit price).
```

The `computed` block is a map from the logical column name to an object with:
- `expression`: a SQL expression (without the `SELECT` keyword) that references other columns by their logical name. The expression is embedded in the generated SQL. The engine replaces column references with the correct qualified names.
- `type`: the data type of the result (one of the built-in types, or a reference to another table for FK-like computed fields – see note below).
- `doc` (optional): documentation as in [Documenting your schema](./documentation.md).

You may also use a shorthand if the expression is simple:

```yaml
computed:
  TotalPrice: "Quantity * UnitPrice"
```

If you omit `type`, it defaults to `decimal`. For non-numeric expressions, specify `type` explicitly.

### Foreign Key Computed Columns

If a computed column has a type that is another table name, FlowerBI treats it as a virtual foreign key. This enables joins to other tables based on a computed value. Example:

```yaml
Invoice:
  id:
    Id: [int]
  columns:
    CreatedAt: [datetime]
  computed:
    YearMonth:
      expression: "FORMAT(CreatedAt, 'yyyy-MM')"
      type: string
    DateMonth:
      expression: "DATETRUNC(month, CreatedAt)"
      type: Date  # joins to the Date table's datetime id
```

Note: The engine must be able to transform the expression into a valid join clause. For complex cases, consider using [virtual tables](./virtual-tables.md) instead.

### Using Computed Fields in Queries

Computed fields are automatically available in the generated TypeScript/C# client code. In a query, you reference them just like regular columns:

```ts
select: {
    orderTotal: ExampleTable.TotalPrice.sum(),
    averagePrice: ExampleTable.UnitPrice.avg(),
}
```

They also appear in the generated documentation and intellisense.

## 2. Custom Aggregation Functions (Custom Rollup Metrics)

FlowerBI currently supports `count`, `sum`, `avg`, `min`, `max`, `distinctcount`. To define a custom aggregation function (e.g. `median`, `customWeightedSum`), you need to extend the server-side engine. This is not yet configurable from YAML alone – it requires writing C# code that implements the aggregation logic.

### Server-Side Implementation

In your .NET backend, create a class that implements the `IAggregationFunction` interface (found in `FlowerBI.Engine`). Example for a median aggregate:

```csharp
public class MedianAggregation : IAggregationFunction
{
    public string SqlFunctionName => "MEDIAN"; // the SQL keyword or expression
    public string Name => "median";

    public string GenerateSql(string sqlColumnExpression)
    {
        // For SQL Server, you might use PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ...)
        // For simplicity, assume a custom SQL function named MEDIAN exists.
        return $"MEDIAN({sqlColumnExpression})";
    }
}
```

Register the function during application startup:

```csharp
var schema = Schema.FromYaml(yaml);
schema.AddAggregationFunction(new MedianAggregation());
```

If you need the aggregation to be usable in client queries, the generated TypeScript client code must also know about it. For now, you can add the function to the `AggregationFunctions` list that `FlowerBI.Tools` uses during code generation. Alternatively, you can manually call the backend with the function name; the engine will match it if registered.

#### Example: Weighted Sum

```csharp
public class WeightedSumAggregation : IAggregationFunction
{
    public string SqlFunctionName => "SUM";
    public string Name => "weightedSum";

    public string GenerateSql(string sqlColumnExpression)
    {
        // Assumes the column expression includes weight multiplication.
        // e.g., column ref is 'Quantity * UnitPrice'
        return $"SUM({sqlColumnExpression})";
    }
}
```

### Client-Side Usage

If the function is registered and the client types are updated, you can use it in queries:

```ts
select: {
    avgPrice: ExampleTable.UnitPrice.avg(),
    medPrice: ExampleTable.UnitPrice.median(),
    weightedTotal: ExampleTable.Quantity.weightedSum(),
}
```

Until the generated types include the new function, you can cast or use string-based column references. This is a known limitation that will be addressed in a future release.

### Alternative: Computed Column + Standard Aggregation

Often a custom aggregation can be replaced by creating a computed column that pre-calculates the weight, and then using a standard `sum`:

```yaml
computed:
  WeightedAmount: "Quantity * UnitPrice"
```

Then query with `ExampleTable.WeightedAmount.sum()`.

For window functions or percentiles, you may need to implement them at the query level via a custom filter or by issuing raw SQL through an endpoint.

## 3. Putting It All Together: Full Example

**Schema (YAML):**

```yaml
schema: SalesSchema
tables:
  Date:
    id:
      Id: [datetime]
    columns:
      CalendarYear: [short]
  Sale:
    id:
      Id: [int]
    columns:
      SaleDate: [Date]
      Quantity: [int]
      UnitPrice: [decimal]
    computed:
      LineTotal:
        expression: "Quantity * UnitPrice"
        type: decimal
      RevenueAfterTax:
        expression: "Quantity * UnitPrice * 0.9"
        type: decimal
```

**Server Registration (C#):**

```csharp
var schema = Schema.FromYaml(File.ReadAllText("schema.yaml"));
schema.AddAggregationFunction(new MedianAggregation());
// ...build engine, handle queries
```

**Client Query:**

```ts
const { records } = useQuery(fetch, {
    select: {
        year: Date.CalendarYear,
        totalRevenue: Sale.RevenueAfterTax.sum(),
        medianQuantity: Sale.Quantity.median(),
    },
    filters: [Date.CalendarYear.equalTo(2025)],
});
```

## Rationale

- Computed fields reduce redundancy in queries and centralise business logic in the schema.
- Custom aggregations allow complex statistical or financial measures without moving logic to the client.
- The proposed `computed` keyword aligns with the existing YAML flexibility (see [YAML Schemas](./yaml.md)).
- The `IAggregationFunction` interface extends the engine without breaking existing queries.
- This document consolidates four separate documentation gaps into one coherent guide.