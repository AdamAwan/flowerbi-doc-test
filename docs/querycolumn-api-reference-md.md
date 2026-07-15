---
title: Client-Side QueryColumn API Reference: Filters, Aggregations, and Ordering
status: draft
---

# Client-Side QueryColumn API Reference: Filters, Aggregations, and Ordering

FlowerBI's TypeScript client exposes a column object hierarchy that mirrors the schema's data types. These objects provide a fluent, type-safe way to construct filters, aggregations, and ordering clauses that are serialised into the JSON wire format and sent to the server.

## Class Hierarchy

Four classes form the column type hierarchy:

| Class | Type Parameter `T` | When to Use |
|---|---|---|
| `QueryColumn<T>` | `FilterValue` (`string \| number \| boolean \| Date \| unknown`) | Base class for all columns. Used directly for booleans, dates, and other non-numeric types. |
| `NumericQueryColumn<T>` | `number \| null` (defaults to `number`) | Columns whose data type is `Decimal`, `Float`, or `Double`. Adds `sum()` and `avg()` aggregation methods. |
| `IntegerQueryColumn<T>` | `number \| null` (defaults to `number`) | Columns with integer types (`Byte`, `Short`, `Int`, `Long`). Extends `NumericQueryColumn` and adds the `bitsIn()` filter method. |
| `StringQueryColumn<T>` | `string \| null` (defaults to `string`) | String-typed columns. Adds the `like()` filter method. |

All column classes share the same constructor signature:

```ts
constructor(public readonly name: string, public readonly type: QueryColumnRuntimeType)
```

The `name` parameter is a dot-delimited identifier of the form `table.column` (e.g. `"Customer.CustomerName"`).

### Code-Generated Columns

The FlowerBI CLI can auto-generate a TypeScript module from a YAML schema file. Each table in the schema becomes a named export object whose properties are pre-configured column instances with the correct class, generic type, and `QueryColumnRuntimeType` metadata. For example:

```ts
import { StringQueryColumn, QueryColumnRuntimeType, QueryColumnDataType } from "flowerbi";

export const Customer = {
    CustomerName: new StringQueryColumn<string>("Customer.CustomerName",
        new QueryColumnRuntimeType(QueryColumnDataType.String, "")
    ),
    // ...
};
```

Generated files import `QueryColumnRuntimeType`, `QueryColumnDataType`, and the appropriate column class from the `flowerbi` package.

## QueryColumnRuntimeType and QueryColumnDataType

Every column instance carries a `QueryColumnRuntimeType` object that stores metadata about the column's data type:

```ts
export class QueryColumnRuntimeType {
    constructor(
        public readonly dataType: QueryColumnDataType,
        public readonly targetColumn: string
    ) {}
}
```

The `targetColumn` property holds a reference to a foreign target column (e.g. `"Customer.Id"`) when the column is a foreign key; otherwise it is an empty string.

`QueryColumnDataType` is a string enum with the following members:

- `None`
- `Bool`
- `Byte`
- `Short`
- `Int`
- `Long`
- `Float`
- `Double`
- `Decimal`
- `String`
- `DateTime`

The TypeScript code generator maps YAML data types to these enum values (e.g. a YAML `bool` column becomes `QueryColumnDataType.Bool`).

## Filter Methods

All filter methods return a `FilterJson` object — a plain JSON structure with `column`, `operator`, and `value` properties that can be passed directly into a query's `filters` array or into an aggregation's per-aggregation filter list.

### Comparison Operators (all column types)

Every `QueryColumn<T>` instance exposes these methods:

| Method | Operator | Signature |
|---|---|---|
| `equalTo(value)` | `=` | `(value: T) => FilterJson` |
| `notEqualTo(value)` | `<>` | `(value: T) => FilterJson` |
| `greaterThan(value)` | `>` | `(value: T) => FilterJson` |
| `lessThan(value)` | `<` | `(value: T) => FilterJson` |
| `greaterThanOrEqualTo(value)` | `>=` | `(value: T) => FilterJson` |
| `lessThanOrEqualTo(value)` | `<=` | `(value: T) => FilterJson` |

Example:

```ts
import { Customer } from "./gen/schema";

const filter = Customer.CustomerName.equalTo("Acme Ltd");
// Produces: { column: "Customer.CustomerName", operator: "=", value: "Acme Ltd" }
```

### Set Membership Filters

The `in()` and `notIn()` methods are available on all column types but are constrained by TypeScript to only accept arrays when `T` is `number` or `string`:

```ts
in(value: T extends number | string ? T[] : never): FilterJson
notIn(value: T extends number | string ? T[] : never): FilterJson
```

Example:

```ts
const filter = Customer.Region.in(["North", "South"]);
// Produces: { column: "Customer.Region", operator: "IN", value: ["North", "South"] }
```

### String LIKE Filter (`StringQueryColumn` only)

`StringQueryColumn` adds a `like()` method for SQL `LIKE` pattern matching:

```ts
like(value: T): FilterJson
```

Example:

```ts
const filter = Customer.CustomerName.like("Acme%");
// Produces: { column: "Customer.CustomerName", operator: "LIKE", value: "Acme%" }
```

### Bitwise Filter (`IntegerQueryColumn` only)

`IntegerQueryColumn` provides `bitsIn(mask, value)` for bitwise comparison. The method takes a bitmask and an array of values to match against:

```ts
bitsIn(mask: number, value: NonNullable<T>[]): FilterJson
```

The `mask` parameter is serialised as the `constant` property in the resulting `FilterJson`, and the `value` array is serialised as the `value` property. The operator is `"BITS IN"`.

Example:

```ts
// Bits 2 (4) and 3 (8) set in the mask = 12
const filter = Customer.Permissions.bitsIn(4 | 8, [0, 4]);
// Produces: { column: "Customer.Permissions", operator: "BITS IN", constant: 12, value: [0, 4] }
```

## Aggregation Methods

Aggregation methods return `AggregationJson` objects — structures with `function`, `column`, and optional `filters` properties. These can be placed into a query's `select` alongside plain columns.

### Common Aggregations (all column types)

Every `QueryColumn<T>` exposes these aggregation methods:

| Method | SQL Function | Signature |
|---|---|---|
| `count(filters?)` | `COUNT` | `(filters?: FilterJson[]) => AggregationJson` |
| `countDistinct(filters?)` | `COUNT(DISTINCT ...)` | `(filters?: FilterJson[]) => AggregationJson` |
| `min(filters?)` | `MIN` | `(filters?: FilterJson[]) => AggregationJson` |
| `max(filters?)` | `MAX` | `(filters?: FilterJson[]) => AggregationJson` |

### Numeric Aggregations (`NumericQueryColumn` and subclasses)

`NumericQueryColumn<T>` extends `QueryColumn<T>` with two additional aggregation methods:

| Method | SQL Function | Signature |
|---|---|---|
| `sum(filters?)` | `SUM` | `(filters?: FilterJson[]) => AggregationJson` |
| `avg(filters?)` | `AVG` | `(filters?: FilterJson[]) => AggregationJson` |

Both `IntegerQueryColumn` and `NumericQueryColumn` inherit `sum()` and `avg()`.

### Per-Aggregation Filters

Every aggregation method accepts an optional array of `FilterJson` objects. These filters are applied to the aggregation independently — they constrain only the rows included in that aggregation calculation, not the overall query result.

Example:

```ts
const query: Query<{ bugsFixed: AggregationJson }, {}> = {
    select: {
        bugsFixed: Bug.Id.count([Bug.Fixed.equalTo(true)]),
    },
};
```

This counts only those bugs where `Fixed` is `true`. The per-aggregation filter is serialised into the `filters` array within the `AggregationJson` object.

### Aggregation JSON Wire Format

An `AggregationJson` object has this shape:

```ts
interface AggregationJson {
    function: "Count" | "Sum" | "Avg" | "Min" | "Max" | "CountDistinct";
    column: string;          // e.g. "Bug.Id"
    filters?: FilterJson[];
}
```

When `jsonifyQuery()` converts a typed query, it extracts properties from the `select` object that correspond to `AggregationJson` values and places them into the `aggregations` array of the `QueryJson` payload.

## Ordering Methods

Every `QueryColumn<T>` exposes two ordering methods that return `OrderingJson` objects:

```ts
ascending(): OrderingJson   // { column: this.name, descending: false }
descending(): OrderingJson  // { column: this.name, descending: true }
```

These produce a simple `OrderingJson` with a `column` string referencing the column by its `table.column` name. This form only works when the column appears in the `select` list of the query.

Example:

```ts
const orderings = [Customer.CustomerName.ascending(), Bug.Id.count().descending()];
// Produces: [{ column: "Customer.CustomerName", descending: false }, { column: "Bug.Id", descending: true }]
```

For more complex ordering (by position or by calculation), use the `Ordering<S, C>` type from `queryModel.ts` instead, which supports `{ select: keyof S }` and `{ calculation: keyof C }` references. The `jsonifyQuery` function resolves these named keys to zero-based positional indices.

## Relationship Between Client-Side Objects and the JSON Wire Format

The column objects themselves are never sent over the wire. The `jsonifyQuery()` function performs the conversion:

1. **Select columns**: Extracts property names from the `select` object. For properties whose value is a `QueryColumn` instance, the column's `.name` (a `table.column` string) goes into the `select` array. For properties whose value is an `AggregationJson`, it goes into the `aggregations` array.
2. **Filters**: `FilterJson` objects produced by filter methods are passed directly into the `filters` array or into an aggregation's per-aggregation `filters`.
3. **Ordering**: `OrderingJson` objects from `ascending()`/`descending()` are placed into the `orderBy` array. Named-key ordering objects (`{ select: "name" }`) are resolved to positional `{ type: "Select", index: N }` or `{ type: "Value", index: N }` form.
4. **Calculations**: Tuple-based `Calculation<S>` expressions (e.g. `[100, "*", ["bugsFixed", "/", "bugCount"]]`) are recursively converted to `CalculationJson` trees referencing aggregations by zero-based positional index.

On the response side, `expandQueryResult()` reverses the process: the `selected` and `aggregated` positional arrays in `QueryRecordJson` are mapped back onto the named properties of the `select` object, producing a strongly-typed `ExpandedQueryRecord<S, C>`.

The `FilterJson` interface sent over the wire:

```ts
interface FilterJson {
    column: string;
    operator: "=" | "<>" | ">" | "<" | ">=" | "<=" | "IN" | "NOT IN" | "BITS IN" | "LIKE";
    value: FilterValue;
    constant?: unknown;   // used by BITS IN for the mask
}
```