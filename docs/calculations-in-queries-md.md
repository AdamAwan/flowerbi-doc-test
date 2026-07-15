# Calculations in Queries: Post-Aggregation Arithmetic in FlowerBI

## Introduction

Calculations allow you to derive new numeric columns from aggregation results within a query. They are evaluated after aggregation, enabling ratios, percentages, weighted scores, and other post-aggregation arithmetic without requiring a separate query or client-side computation.

## The CalculationJson Model

A calculation is expressed as a tree of operands. Each node in the tree may be one of three kinds:

- **`Value`**: a constant decimal number.
- **`Aggregation`**: a reference to an aggregation by its zero-based index in the query's aggregation array.
- **`First` / `Second` / `Operator`**: a binary expression with a left operand (`First`), a right operand (`Second`), and a string `Operator`.

Exactly one of `Value`, `Aggregation`, or the `First`/`Operator`/`Second` combination must be provided per node; if more than one is set, the engine raises a `FlowerBIException`.

### Allowed Operators

The supported binary operators are:

- `"+"` — addition
- `"-"` — subtraction
- `"*"` — multiplication
- `"/"` — division
- `"??"` — null coalescing (returns `Second` if `First` is null, otherwise `First`)

Any other operator string causes a `FlowerBIException` with the message `"Operator '{op}' not supported"`.

### Division Safety

When the operator is `"/"`, the `ToSql` method generates a conditional expression that guards against division by zero. It uses the `ISqlFormatter.Conditional` method to emit:

```sql
CASE WHEN <secondExpr> = 0 THEN 0 ELSE <firstExpr> / CAST(<secondExpr> AS FLOAT) END
```

The right-hand side of the division is cast to a floating-point type via `ISqlFormatter.CastToFloat` — `cast(... as float)` for SQL Server and `cast(... as real)` for SQLite — ensuring the result is a decimal rather than performing integer division.

### Null Coalescing via `??`

The `??` operator also uses `Conditional`, producing:

```sql
CASE WHEN <firstExpr> IS NULL THEN <secondExpr> ELSE <firstExpr> END
```

## SQL Generation: The `calculation_source` CTE

When a query has one or more calculations, the SQL generator uses a Common Table Expression (CTE) pattern. The inner query (select, joins, filters, grouping, ordering without pagination) is wrapped in a CTE named `calculation_source`, and calculations are appended as additional columns in the outer SELECT:

```sql
WITH calculation_source AS (
    select ... group by ...
)
select calculation_source.*
    ,<calc1> AS Value<n>
    ,<calc2> AS Value<n+1>
    ...
from calculation_source
order by ...
offset ... rows fetch next ... rows only
```

Each calculation output column is aliased as `Value` followed by its position offset by the number of aggregations. For example, if there are two aggregations (`Value0`, `Value1`), the first calculation becomes `Value2`, the second `Value3`, and so on.

If there are no calculations, the template omits the CTE entirely and generates a flat SELECT with an optional ORDER BY and pagination footer.

## Ordering by Calculation Results

Query results can be ordered by the output of a calculation using `OrderingType.Calculation` with a zero-based index into the calculations array. The `OrderingJson` type supports a discriminated union:

```json
{ "type": "Calculation", "index": 0 }
```

When the query is constructed from the client-side typed `Query` object, you can use the `calculation` key directly:

```ts
orderBy: [
    { calculation: "successRate" },
    { calculation: "failureRate", descending: true }
]
```

The `Ordering` constructor on the server resolves `OrderingType.Calculation` to an ordinal position that accounts for all select columns and aggregations before it. The generated SQL `ORDER BY` clause then references that position. If the index is out of range, an `ArgumentOutOfRangeException` is raised.

## Client-Side Typed Representation

On the client side, calculations are modelled as a recursive union type:

```ts
export type Calculation<S extends QuerySelect> =
    | number                                          // constant value
    | AggregatePropsOnly<S>                            // reference to an aggregation by property name
    | [Calculation<S>, "+" | "-" | "*" | "/" | "??", Calculation<S>];  // tuple: [left, op, right]
```

Calculations are declared as a record of named calculation expressions:

```ts
export type QueryCalculations<S extends QuerySelect> = Record<string, Calculation<S>>;
```

### The `jsonifyCalculation` Function

When the typed query is serialised to JSON via `jsonifyQuery`, the `jsonifyCalculation` function recurses through the `Calculation<S>` tree:

- A **string** value (the name of an aggregation property) is converted to `{ aggregation: <index> }`, where the index is looked up from the ordered list of aggregation property names.
- A **number** value becomes `{ value: <number> }`.
- An **array** (tuple of three elements) becomes `{ first: ..., operator: ..., second: ... }` with each operand recursively serialised.

### Mapping Results Back: `expandQueryRecord`

When query results are returned from the API as a `QueryRecordJson` (with a flat `selected` and `aggregated` array), `expandQueryRecord` maps the calculation result values from the tail of the `aggregated` array into named properties on the output record. Calculation values appear after all aggregation values, in the same order as their definitions in the query.

---

## Complete Example: Percentage Rate Calculation

A common scenario is computing a success rate from two aggregations. The following example defines a query that counts total bugs and bugs fixed, then computes `successRate` and `failureRate` as percentage expressions.

### C# Server-Side (JSON payload)

```json
{
  "select": ["Customer.CustomerName"],
  "aggregations": [
    { "function": "Count", "column": "Bug.Id" },
    { "function": "Count", "column": "Bug.Id", "filters": [{ "column": "Bug.Fixed", "operator": "=", "value": true }] }
  ],
  "calculations": [
    { "first": { "value": 100 }, "operator": "*", "second": { "first": { "aggregation": 1 }, "operator": "/", "second": { "aggregation": 0 } } },
    { "first": { "value": 100 }, "operator": "-", "second": { "first": { "value": 100 }, "operator": "*", "second": { "first": { "aggregation": 1 }, "operator": "/", "second": { "aggregation": 0 } } } }
  ],
  "orderBy": [
    { "type": "Calculation", "index": 0 },
    { "type": "Calculation", "index": 1, "descending": true }
  ]
}
```

The first calculation computes `successRate = 100 * (bugsFixed / bugCount)`. The inner division `{ aggregation: 1 } / { aggregation: 0 }` divides the count of fixed bugs by the total bug count, then multiplies by 100. The second calculation computes `failureRate = 100 - (100 * (bugsFixed / bugCount))`.

Generated SQL (simplified):

```sql
WITH calculation_source AS (
    select [Customer].[CustomerName] AS Select0,
        count([Bug].[Id]) AS Value0,
        count(case when [Bug].[Fixed] = 1 then [Bug].[Id] end) AS Value1
    from ...
    group by [Customer].[CustomerName]
)
select calculation_source.*
    , iif([Value1] / cast([Value0] as float) = 0, 0, 100 * ([Value1] / cast([Value0] as float))) AS Value2
    , 100 - (iif([Value1] / cast([Value0] as float) = 0, 0, 100 * ([Value1] / cast([Value0] as float))) AS Value3
from calculation_source
order by 4 asc, 5 desc
```

### TypeScript Client-Side

```ts
import { useFlowerBI } from "flowerbi-react";

const { records } = useFlowerBI(fetch, {
    select: {
        customer: Customer.CustomerName,
        bugCount: Bug.Id.count(),
        bugsFixed: Bug.Id.count([Bug.Fixed.equalTo(true)]),
    },
    calculations: {
        successRate: [100, "*", ["bugsFixed", "/", "bugCount"]],
        failureRate: [100, "-", [100, "*", ["bugsFixed", "/", "bugCount"]]],
    },
    orderBy: [
        { select: "customer" },
        { select: "bugsFixed", descending: true },
        { calculation: "successRate" },
        { calculation: "failureRate", descending: true },
    ],
});
```

The `ExpandedQueryRecord` type includes `successRate` and `failureRate` as `number` properties. The tuple syntax `[left, op, right]` mirrors the server-side tree structure.

Each record returned is strongly typed:

```ts
{ customer: string; bugCount: number; bugsFixed: number; successRate: number; failureRate: number }
```

If `totals` are enabled, the totals record includes the calculation values computed from the overall aggregation totals, typed as `AggregateValuesOnly<S> & CalculationValues<C>`.

---

## Summary

| Aspect | Details |
|-------|---------|
| **Model** | Tree of `Value` (constants), `Aggregation` (index reference), or `First`/`Second`/`Operator` (binary expressions) |
| **Operators** | `+`, `-`, `*`, `/`, `??` (null coalesce) |
| **Division safety** | `CASE WHEN divisor = 0 THEN 0 ELSE dividend / CAST(divisor AS FLOAT) END` |
| **SQL pattern** | `WITH calculation_source AS (...) SELECT ..., <calc> AS Value<N> FROM calculation_source` |
| **Ordering** | `OrderingType.Calculation` with zero-based index into calculations array |
| **Client type** | `Calculation<S> = number \| keyof aggregates \| [left, op, right]` tuple |
| **Serialisation** | `jsonifyCalculation` recurses the typed tree; results mapped by `expandQueryRecord` |