---
title: Conjoint Tables: Auto-Virtualizing Tables for Multi-Instance Queries
status: draft
---

# Conjoint Tables: Auto-Virtualizing Tables for Multi-Instance Queries

## The Problem: Referencing the Same Table Multiple Times

When modelling real-world data, it is common to have a single logical table that needs to appear multiple times within the same query. A canonical example is **annotations** (name-value pairs) attached to records. Suppose you have inventory items that can be annotated with arbitrary metadata — an item might have a `shopping` annotation, a `math` annotation, a `priority` annotation, and so on. All annotations share the same physical schema (`AnnotationName`, `AnnotationValue`, and a linking table), but to query by *two* different annotation names simultaneously you need to join `AnnotationName` twice and `AnnotationValue` twice, as if they were separate tables.

Without a specialised mechanism, the only approach is to create manual virtual tables via `extends`:

```yaml
AnnotationValue1:
    extends: AnnotationValue

AnnotationValue2:
    extends: AnnotationValue

AnnotationName1:
    extends: AnnotationName

AnnotationName2:
    extends: AnnotationName

InventoryItemAnnotation1:
    extends: InventoryItemAnnotation

InventoryItemAnnotation2:
    extends: InventoryItemAnnotation
```

This works but is clunky: you must pre-declare enough copies for every possible query, and the foreign keys on the annotation-value linking table must be rewritten to point at each copy rather than at the original table.

## The Solution: `conjoint: true`

FlowerBI provides a **conjoint** table declaration that eliminates the need for manual virtual-table copies. Adding `conjoint: true` to a table in the YAML schema grants it *auto-virtualisation superpowers*: the table can be referenced with an arbitrary label suffix at query time, and the join system automatically creates as many labelled instances as needed.

### Marking Tables as Conjoint in YAML

To enable the feature, set `conjoint: true` on every table in the annotation chain:

```yaml
AnnotationName:
    conjoint: true
    id:
        Id: [int]
    columns:
        Name: [string]

AnnotationValue:
    conjoint: true
    id:
        Id: [int]
    columns:
        AnnotationNameId: [AnnotationName]
        Value: [string]

InventoryItemAnnotation:
    conjoint: true
    columns:
        InventoryItemId: [InventoryItem]
        AnnotationValueId: [AnnotationValue]
```

Once marked, the tables can be queried with an `@suffix` appended to any column reference:

```json
{
    "select": ["AnnotationValue.Value@math", "AnnotationValue.Value@shopping"],
    "aggregations": [
        {
            "column": "InventoryItem.Id",
            "function": "Count"
        }
    ],
    "filters": [
        {
            "column": "AnnotationName.Name@math",
            "operator": "=",
            "value": "math"
        },
        {
            "column": "AnnotationName.Name@shopping",
            "operator": "=",
            "value": "shopping"
        }
    ]
}
```

The suffix (here `@math` and `@shopping`) creates two independent instances of the entire conjoint-table graph. Each instance carries its own copy of the joins through `AnnotationName` → `AnnotationValue` → `InventoryItemAnnotation` → `InventoryItem`, so the two filters on `AnnotationName.Name` bind to the correct `AnnotationValue.Value` selections.

### The `@suffix` Syntax

Any column reference in a query JSON can carry an `@label` suffix:

- `AnnotationValue.Value@math` — column `AnnotationValue.Value` with label `math`
- `AnnotationName.Name@shopping` — column `AnnotationName.Name` with label `shopping`

The suffix can be any arbitrary string. It never appears in the generated SQL; it is purely a logical label that FlowerBI uses to group column references into separate table instances. Suffixes must match between `select` and `filter` columns that are intended to refer to the same instance. There is no requirement that the suffix correspond to any data value — `@x` and `@y` would work just as well.

## How Conjoint Differs from Manual Virtual Tables

Virtual tables created via `extends` require you to:

1. Pre-declare a fixed number of named copies (e.g. `AnnotationName1`, `AnnotationName2`) in the YAML schema.
2. Re-declare foreign keys for each copy, because the FK type must point to the virtual table name, not the original.
3. Pick an arbitrary upper limit (N copies) and hope no query needs more.

Conjoint tables eliminate all of that. You declare the table once with `conjoint: true` and the engine creates instances on demand from the `@` labels that appear in a query. No pre-declaration, no FK rewrites, no arbitrary limits.

## How It Works Inside the Engine

### Schema Parsing: `Table.Conjoint`

When the YAML schema is resolved, each table's `conjoint` property is read from `YamlTable.conjoint` and stored as the `bool Conjoint` property on the runtime `Table` object.

### Column Name Parsing: `Schema.GetColumn(string)`

The method `Schema.GetColumn(string labelledName)` is the entry point for turning a query column reference into a `LabelledColumn`. It splits the string on the first `@` character:

- If an `@` is found, the portion before it is the table-qualified column name (e.g. `AnnotationValue.Value`) and the portion after is the label (e.g. `math`).
- If no `@` is found, the label is `null`.

If the resolved table is not conjoint but a label was supplied, an exception is thrown — only conjoint tables accept labels.

### Join Label Propagation: `LabelledTable` and `LabelledColumn`

The records `LabelledColumn(string JoinLabel, IColumn Value)` and `LabelledTable(string JoinLabel, Table Value)` carry the join label alongside the column or table. When a `LabelledTable` is constructed from a non-conjoint table, the label is forced to `null` regardless of what was passed, ensuring that non-conjoint tables always participate in a single, unlabelled join instance.

### Join Expansion: `Joins.GetLabelledArrows`

During SQL generation, the `Joins` class collects all distinct join labels from the `Aliases` dictionary. The method `GetLabelledArrows(LabelledTable table)` iterates over foreign-key relationships. For each arrow:

- If the **target** table is conjoint but the **source** table is not, a separate labelled arrow is emitted for **every** distinct join label in the query. This is what fans out a single conjoint table into multiple labelled instances.
- Otherwise, the source's label is propagated to the target table, keeping table instances consistent.

### Associative Hint Integration

When conjoint tables are used with the many-to-many pattern (an associative table with at least two foreign keys to other conjoint tables), the `associative` hint tells the join engine to keep that table in the join system even when it could theoretically be eliminated. The conjoint `InvoiceAnnotation` table in the test schemas declares `associative: [InvoiceId, AnnotationValueId]`, ensuring it is always present to connect `Invoice` and `AnnotationValue` correctly.

## Example from the Test Schemas

Both `TestSchema` and `ComplicatedTestSchema` define a set of conjoint annotation tables:

| Table | `conjoint` | Notes |
|---|---|---|
| `AnnotationName` | `true` | Holds annotation name strings |
| `AnnotationValue` | `true` | Holds annotation values, FK to `AnnotationName` |
| `InvoiceAnnotation` | `true` | Links `Invoice` to `AnnotationValue`; marked `associative` with `[InvoiceId, AnnotationValueId]` |

In `ComplicatedTestSchema`, the annotation tables also have a foreign key `DepartmentId` to `Department`, demonstrating that conjoint tables can participate in additional relationships beyond the annotation chain.

A test query named `MultipleManyToManyWithSpecifiedJoins` selects `AnnotationValue.Value@x` and `AnnotationValue.Value@y`, filters on `AnnotationName.Name@x = 'Approver'` and `AnnotationName.Name@y = 'Instructions'`, and correctly returns annotation values paired by label.

Another test, `ManyToManyConjointWithComplicatedSchema`, adds a third filter on `Department.DepartmentName` to demonstrate conjoint tables coexisting with ordinary dimension tables in the same query.

## Summary

- Mark tables with `conjoint: true` in the YAML schema to enable auto-virtualisation.
- Reference columns with `Table.Column@label` in query JSON to create labelled instances.
- The engine creates as many join instances as there are distinct labels — no pre-declaration needed.
- Conjoint tables are fully compatible with the `associative` hint for safe many-to-many relationships.
- The suffix is purely logical; it never appears in generated SQL.