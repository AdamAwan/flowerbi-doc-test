---
title: FlowerBI Query System: Syntax, Capabilities and Execution Model
status: draft
---

# FlowerBI Query System: Syntax, Capabilities and Execution Model

This document describes the complete query system of FlowerBI: how a query is expressed in JSON, how the TypeScript client DSL maps to it, and how the server executes it to produce results.

## Query JSON Format (Wire Protocol)

A query is sent to the server as a JSON object with the following properties:

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `select` | `string[]` | Yes | — | Plain column names of the form `Table.Column` to group by. |
| `aggregations` | `AggregationJson[]` | Yes | — | Values computed by applying an aggregation function to a column. |
| `calculations` | `CalculationJson[]` | No | — | Derived expressions built from aggregations, numeric constants, and arithmetic operators. |
| `filters` | `FilterJson[]` | No | `[]` | Filters applied globally to all data before grouping. Combined with AND. |
| `orderBy` | `OrderingJson[]` | No | `[]` | Sorting criteria. Default is by the first aggregation descending. |
| `totals` | `bool` | No | `false` | When true, returns an extra totals record with aggregation values across all data. |
| `skip` | `long` | No | `0` | Number of result rows to skip before returning (offset-based pagination). |
| `take` | `int` | No | `100` | Maximum number of result rows to return. |
| `comment` | `string` | No | — | An arbitrary string injected as a SQL comment for debugging. Non-alphanumeric characters are replaced with spaces to prevent injection. |
| `allowDuplicates` | `bool` | No | `false` | When true and there are no aggregations, omits GROUP BY so duplicate rows are preserved. |
| `fullJoins` | `bool` | No | `false` | When true and there are multiple aggregations, uses FULL JOIN instead of LEFT JOIN so no aggregation's rows are lost. |

### AggregationJson

Each aggregation specifies:

```json
{
    "column": "Bug.Id",
    "function": "Count",
    "filters": [
        { "column": "Workflow.Resolved", "operator": "=", "value": true }
    ]
}
```

- **`column`**: The column to aggregate, in `Table.Column` form.
- **`function`**: One of `"Count"`, `"Sum"`, `"Avg"`, `"Min"`, `"Max"`, `"CountDistinct"`.
- **`filters`** (optional): Filters applied only to this aggregation's rows, using a `CASE WHEN` expression in the generated SQL.

### FilterJson

```json
{
    "column": "Customer.CustomerName",
    "operator": "=",
    "value": "Pies LLC"
}
```

- **`column`**: Column in `Table.Column` form.
- **`operator`**: One of `=`, `<>`, `!=`, `>`, `<`, `>=`, `<=`, `IN`, `NOT IN`, `BITS IN`, `LIKE`.
- **`value`**: The value to compare against. For `IN` and `NOT IN`, an array of values. Types must be `bool`, `byte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `float`, `double`, `string`, or `DateTime`. An empty array is rejected.
- **`constant`** (optional): Used by `BITS IN` to specify a bitmask as an integer value.

### CalculationJson

Calculations compose aggregations, numeric constants, and arithmetic operators into derived values:

```json
{
    "first": { "value": 100 },
    "operator": "*",
    "second": {
        "first": { "aggregation": 0 },
        "operator": "/",
        "second": { "aggregation": 1 }
    }
}
```

Each node is one of:
- **`{ "value": number }`** — a literal numeric constant.
- **`{ "aggregation": int }`** — a reference to an aggregation by its zero-based index in the `aggregations` array.
- **`{ "first": ..., "operator": "...", "second": ... }`** — a binary expression.

Allowed operators are `+`, `-`, `*`, `/`, and `??` (null-coalescing — returns the first operand if non-null, otherwise the second). The `/` operator includes safety: when dividing, if the divisor evaluates to zero the result is `0` rather than an error. The `??` operator is translated to a SQL `CASE WHEN ... IS NULL` expression.

### OrderingJson

Ordering can be expressed in two forms:

**By column name** (must match one of the `select` columns):
```json
{ "column": "Customer.CustomerName", "descending": true }
```

**By type and index** (more flexible, works for any result column):
```json
{ "type": "Select", "index": 0, "descending": false }
```

Where `type` is `"Select"` (plain columns, zero-based), `"Value"` (aggregations, zero-based), or `"Calculation"` (calculations, zero-based). The index is relative to the type's position in the result set. Internally, type-index ordering is converted to a column ordinal: select columns come first (0, 1, 2...), followed by aggregations, followed by calculations.

## TypeScript Client DSL

FlowerBI provides a strongly-typed client library that lets you build queries programmatically and receive typed results.

### Column Classes

Generated TypeScript code creates one module per table, exporting column objects matching the YAML schema. Four classes are used depending on the column's data type:

| Class | Used For | Methods |
|-------|----------|---------|
| `QueryColumn<T>` | All types | `.count()`, `.countDistinct()`, `.min()`, `.max()` |
| `NumericQueryColumn<T>` | `number` types | Inherits `QueryColumn`, adds `.sum()`, `.avg()` |
| `IntegerQueryColumn<T>` | `int`/`short`/`byte` types | Inherits `NumericQueryColumn`, adds `.bitsIn(mask, values)` |
| `StringQueryColumn<T>` | `string` types | Inherits `QueryColumn`, adds `.like(value)` |

Each column object carries a `type` property of type `QueryColumnRuntimeType`, which exposes the column's `dataType` (one of `None`, `Bool`, `Byte`, `Short`, `Int`, `Long`, `Float`, `Double`, `Decimal`, `String`, `DateTime`), its `targetColumn` (the database column it references or an FK target), and a `meta` dictionary of user-defined name/value pairs declared in the schema.

### Filter Methods

Every column exposes these filter methods:

| Method | SQL Operator | Supported On |
|--------|-------------|--------------|
| `.equalTo(value)` | `=` | All column types |
| `.notEqualTo(value)` | `<>` | All column types |
| `.greaterThan(value)` | `>` | All column types |
| `.lessThan(value)` | `<` | All column types |
| `.greaterThanOrEqualTo(value)` | `>=` | All column types |
| `.lessThanOrEqualTo(value)` | `<=` | All column types |
| `.in(values)` | `IN` | Number or string columns |
| `.notIn(values)` | `NOT IN` | Number or string columns |
| `.like(value)` | `LIKE` | String columns only |
| `.bitsIn(mask, values)` | `BITS IN` | Integer columns only |

The `BITS IN` operator checks whether a column's bits, combined with the given mask via bitwise AND, fall within a set of allowed values.

### Ordering Methods

Every column exposes `.ascending()` and `.descending()` methods returning an `OrderingJson` object with the column name set.

### Aggregation Methods

Aggregation methods return `AggregationJson` objects. Each accepts an optional array of `FilterJson` for per-aggregation filtering:

```ts
Bug.Id.count()                        // total count
Bug.Id.count([Workflow.Resolved.equalTo(true)])  // count of resolved only
```

### Query Definition

The strongly-typed `Query` interface ties together select, aggregations, calculations, filters, ordering, and pagination:

```ts
interface Query<S extends QuerySelect, C extends QueryCalculations<S>> {
    select: S;
    filters?: FilterJson[];
    calculations?: C;
    orderBy?: (OrderingJson | Ordering<S, C>)[];
    totals?: boolean;
    skip?: number;
    take?: number;
    comment?: string;
    allowDuplicates?: boolean;
}
```

The generic parameters `S` (select) and `C` (calculations) enable full type inference of the result records.

### Calculations in the DSL

The `Calculation<S>` union type captures three forms:

- A numeric literal: `42`
- A string referencing an aggregation name: `"bugsFixed"`
- A tuple of `[Calculation, operator, Calculation]`: `[100, "*", ["bugsFixed", "/", "bugCount"]]`

This allows readable specification of derived columns:

```ts
calculations: {
    successRate: [100, "*", ["bugsFixed", "/", "bugCount"]],
}
```

### Ordering by Named Keys

Ordering can refer to select or calculation keys by name:

```ts
orderBy: [
    { select: "customer" },
    { select: "bugsFixed", descending: true },
    { calculation: "successRate" },
]
```

The `jsonifyQuery` function resolves named keys to positional indices in the JSON wire format.

## Execution Pipeline

### 1. Client: `jsonifyQuery`

The `jsonifyQuery` function converts the typed `Query` object into a `QueryJson` wire-format object. It separates select properties (plain columns) from aggregation properties, resolves calculation expressions into `CalculationJson` trees, resolves ordering keys to positional indices, and sets all defaults (`skip: 0`, `take: 100`, `totals: false`, etc.).

### 2. Server: Query Construction

The server's `Query` class takes the `QueryJson` and a `Schema` instance. It:

- **Loads select columns** by resolving each `Table.Column` string via `Schema.GetColumn()`, which also handles conjoint `@suffix` labels.
- **Loads aggregations** by parsing function type, column reference, and per-aggregation filters.
- **Loads filters** by resolving column references and validating operators against the allowed set.
- **Loads ordering** by resolving column-name references or type-index references.
- **Holds calculations** as raw `CalculationJson` trees.

### 3. Server: SQL Generation

The `Query.ToSql()` method generates SQL using Handlebars templates. There are two templates:

- **Without calculations**: A simple `SELECT ... FROM joins WHERE filters GROUP BY select ORDER BY ... OFFSET...FETCH...`
- **With calculations**: A `WITH calculation_source AS ( ... ) SELECT ..., calculated_values FROM calculation_source ORDER BY ...` wrapping the base query in a CTE so that calculations can reference aggregation values.

Key decisions during SQL generation:

- **Grouping**: When no aggregations exist and `allowDuplicates` is true, `GROUP BY` is omitted. When `totals` is true, grouping is omitted for the totals query.
- **Ordering**: Defaults to the first aggregation descending when aggregations exist, otherwise the first select column ascending.
- **Pagination**: Uses SQL-specific `OFFSET...FETCH NEXT` syntax (SQL Server) or `LIMIT...OFFSET` (SQLite), determined by the `ISqlFormatter` strategy.
- **Per-aggregation filters**: Wrapped in a `CASE WHEN ... THEN expr END` SQL construct.
- **Full joins**: When `fullJoins` is true, each CTE result is joined using `FULL JOIN` with `COALESCE` on the select columns to populate from whichever side has values.

### 4. Server: SQL Dialect Support

The `ISqlFormatter` interface abstracts dialect differences:

| Feature | SQL Server (`SqlServerFormatter`) | SQLite (`SqlLiteFormatter`) |
|---------|----------------------------------|-----------------------------|
| Identifier quoting | `[name]` | `[name]` |
| Pagination | `OFFSET n ROWS FETCH NEXT m ROWS ONLY` | `LIMIT m OFFSET n` |
| Conditional | `IIF(pred, then, else)` | `IIF(pred, then, else)` |
| Float cast | `CAST(expr AS float)` | `CAST(expr AS real)` |

### 5. Server: Parameterisation

Filter values are embedded into SQL using the `IFilterParameters` strategy. The default `DapperFilterParameters` embeds numeric and boolean values as safe literals (to aid query plan caching) and parameterises string and DateTime values via Dapper's `DynamicParameters`.

### 6. Server: Result Conversion

Query results are streamed from the database as `dynamic` objects, then converted to `QueryRecordJson` with `selected` (plain column values) and `aggregated` (aggregation and calculation values) arrays. Null handling uses a null-conversion lambda for calculations, while column values use the schema's `ConvertValue` method for each column's CLR type.

### 7. Client: Result Expansion

The `expandQueryResult` function reconstructs strongly-typed records from the wire-format arrays. It maps positional arrays back to named properties using the original query's `select` shape and optional `calculations`. The optional `totals` record returns aggregate-only values (plain columns are omitted).

## Query Variants

### Grouping Query (Default)
A query with one or more aggregations groups by all plain select columns. The general form is:

```sql
SELECT t1.col1 Select0, ..., agg(col) Value0, ...
FROM ...
JOIN ...
WHERE ...
GROUP BY t1.col1, ...
ORDER BY ...
```

### Ungrouped Query (allowDuplicates)
When `allowDuplicates: true` and no aggregations exist, `GROUP BY` is omitted. This can improve performance where duplicate rows are acceptable or impossible.

### Totals Query
When `totals: true`, two SQL queries are executed: the first omits grouping (aggregating across all rows), the second is the standard grouped query. The results are returned as a `totals` record alongside the main `records` array.

### Multi-Aggregation with Full Joins
Queries with multiple aggregations (potentially with per-aggregation filters) produce one CTE per aggregation. By default these are combined with `LEFT JOIN`, meaning the first aggregation controls which rows survive. Setting `fullJoins: true` uses `FULL JOIN` and `COALESCE` on select columns, so every aggregation's contributing rows are preserved.

### Filtered Aggregation
Each individual aggregation can carry its own filter list. The server wraps the aggregated expression in a `CASE WHEN` construct:

```sql
SUM(CASE WHEN filter_column = value THEN column_to_sum ELSE NULL END)
```

This allows comparing metrics under different conditions within a single query, e.g. total bugs vs resolved bugs per customer.

### Conjoint Table Queries
Tables declared `conjoint: true` in the schema can be referenced with an `@label` suffix on column names (e.g. `AnnotationValue.Value@shopping`). The join engine fans out the table into separate labelled instances per query, enabling queries that join the same logical table multiple times with different filters.

## Result Format

The server returns a JSON payload with:

```json
{
    "records": [
        { "selected": ["Pies LLC"], "aggregated": [42] },
        { "selected": ["Hats-R-Us"], "aggregated": [17] }
    ],
    "totals": { "selected": [""], "aggregated": [59] }
}
```

The client's `expandQueryResult` transforms this into typed objects:

```ts
{
    records: [{ customer: "Pies LLC", bugCount: 42 }, ...],
    totals: { bugCount: 59 }
}
```

The `QueryValuesRow` helper class adds a `percentage(key)` method that divides a record's aggregation value by the totals value and returns the percentage, rounded to two decimal places.

## Related Documentation

- [YAML Schemas](./yaml.md) — declaring the data model
- [Virtual Tables](./virtual-tables.md) — creating aliases for tables
- [Many-to-Many](./many-to-many.md) — associative table hints
- [Full Joins](./full-joins.md) — multi-aggregation join behaviour
- [Conjoint Tables](./conjoint.md) — auto-virtualisation for repeated table patterns
- [Documenting your schema](./documentation.md) — doc, see, and meta annotations
