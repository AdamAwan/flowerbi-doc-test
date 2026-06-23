---
title: Computed Fields and Custom Aggregations
status: draft
---

# Computed Fields and Custom Aggregations

This guide explains how to define derived (computed) columns in a YAML schema and how to extend FlowerBI with custom aggregation functions.

## Computed Fields in YAML

Define a `computed` block in a table:
```yaml
Sale:
  columns:
    Quantity: [int]
    UnitPrice: [decimal]
  computed:
    TotalPrice:
      expression: "Quantity * UnitPrice"
      type: decimal
```

## Using Computed Fields in Queries

```ts
select: {
  totalRevenue: Sale.TotalPrice.sum()
}
```

## Custom Aggregation Functions

Not natively supported. Workarounds:
- Use filtered aggregations to simulate conditional logic.
- Pre-compute in database views.
- Extend the engine via `IAggregationFunction` interface (advanced).

### Example: Server-side registration
```csharp
public class MedianAggregation : IAggregationFunction { ... }
schema.AddAggregationFunction(new MedianAggregation());
```

*Consolidated from: computed-fields-and-custom-aggregations.md, derived-fields-and-custom-aggregations-in-flowerbi.md*