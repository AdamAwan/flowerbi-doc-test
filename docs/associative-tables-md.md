---
title: Associative Tables: Preserving Many-to-Many Join Integrity
status: draft
---

# Associative Tables: Preserving Many-to-Many Join Integrity

In a multi-tenant or deeply connected schema, FlowerBI's join-reduction algorithm can eliminate a link table from the generated SQL when it finds an alternative path connecting the same tables. This can produce an unintended Cartesian product, duplicating rows. The `associative` hint tells FlowerBI to keep the link table in the join, preserving the correct many-to-many relationship.

## The problem: accidental Cartesian products

Consider a typical many-to-many relationship, with a link table between `BlogPost` and `Tag`:

```yaml
BlogPost:
  id:
    Id: [int]
  columns:
    TenantId: [Tenant]
    Text: [string]

Tag:
  id:
    Id: [int]
  columns:
    TenantId: [Tenant]
    Name: [string]

BlogPostTag:
  columns:
    BlogPostId: [BlogPost]
    TagId: [Tag]
```

Here, both `BlogPost` and `Tag` reference `Tenant` via a foreign key. If a query depends on `Tenant` directly (for example, filtering by `Tenant.Id`), FlowerBI can connect `BlogPost` to `Tag` through `Tenant` without ever including `BlogPostTag`. The resulting SQL becomes an unconstrained cross join of all posts with all tags in the same tenant — every post appears paired with every tag, as if every tag applied to every post.

## The `associative` hint

To prevent this, mark a link table's foreign-key columns with the `associative` array in the YAML schema:

```yaml
BlogPostTag:
  columns:
    BlogPostId: [BlogPost]
    TagId: [Tag]
  associative:
    - BlogPostId
    - TagId
```

The value is a list of column names that form the associative link. Each name must match a column defined in the table (either in `id` or `columns`). At least two columns are expected, because an associative table must connect at least two other tables.

## How the join algorithm preserves associative tables

FlowerBI's join-construction logic in `Joins.cs` proceeds in two phases:

1. **Reduction phase.** Starting from the root table, the algorithm finds the minimal set of tables needed to connect all tables directly referenced by the query (selected columns, aggregation columns, filter columns). Any table that is not strictly required for connectivity is eliminated.

2. **Associative re-addition phase.** After reduction, the algorithm scans all tables in the schema that have two or more `associative` columns and are not currently in the reachable set. For each such candidate, it checks whether the table has at least two *forward* foreign-key arrows (not reverse) whose key columns are listed in the `associative` property and whose target tables are already present in the reduced join. If both conditions are met, the table is re-added to the join — it is considered an essential link that must not be dropped, even though a theoretical alternative path exists.

This re-addition is iterative: adding one associative table may make another associative table eligible, so the algorithm loops until no further tables can be added.

## Tables without an identity column

Associative link tables are often pure junction tables containing only foreign-key columns and no dedicated primary key. This is fully supported: the `id` property is optional in the YAML schema. FlowerBI maps such tables without an identity column, using only the FK columns to drive joins.

```yaml
InvoiceTag:
  columns:
    InvoiceId: [Invoice]
    TagId: [Tag]
  associative:
    - InvoiceId
    - TagId
```

Here `InvoiceTag` has no `id` — it is purely a collection of FK pairs. The schema resolves successfully and the join engine uses these FK columns to wire the table into queries.

## Concrete YAML examples from test schemas

The test schemas demonstrate several associative patterns:

### Simple many-to-many link

```yaml
InvoiceTag:
  columns:
    InvoiceId: [Invoice]
    TagId: [Tag]
  associative:
    - InvoiceId
    - TagId

InvoiceCategory:
  columns:
    InvoiceId: [Invoice]
    CategoryId: [Category]
  associative:
    - InvoiceId
    - CategoryId
```

These link `Invoice` to `Tag` and `Invoice` to `Category` respectively. Without the `associative` hint, if both `Invoice` and `Tag` (or `Invoice` and `Category`) can be connected via another table (such as `Department`), the link table would be eliminated.

### Conjoint associative table (annotation pattern)

```yaml
InvoiceAnnotation:
  conjoint: true
  columns:
    InvoiceId: [Invoice]
    AnnotationValueId: [AnnotationValue]
  associative:
    - InvoiceId
    - AnnotationValueId
```

`InvoiceAnnotation` is both conjoint and associative. It sits between `Invoice` and the annotation hierarchy (`AnnotationName` → `AnnotationValue`). Because it is conjoint, you can reference it with a label suffix (`@x`, `@y`, `@shopping`) to create multiple virtual instances in a single query — for example, to filter by one annotation name while grouping by another.

The `associative` hint is especially important here: without it, the join reduction could eliminate `InvoiceAnnotation` when both `Invoice` and `AnnotationValue` are already connected via other paths, collapsing the many-to-many annotation link.

## Interaction with conjoint tables

When an associative table is also conjoint (like `InvoiceAnnotation` above), the join engine's `GetLabelledArrows` method fans out each conjoint table instance per join label. The associative re-addition check operates on these labelled instances: a labelled associative table is re-added if at least two of its forward FK arrows connect to tables that are already present under the same label in the reduced join.

This enables queries that use multiple annotation labels simultaneously:

```json
{
  "select": [
    "Vendor.VendorName",
    "AnnotationValue.Value@x",
    "AnnotationValue.Value@y"
  ],
  "aggregations": [
    { "column": "Invoice.Amount", "function": "Sum" }
  ],
  "filters": [
    { "column": "AnnotationName.Name@x", "operator": "=", "value": "Approver" },
    { "column": "AnnotationName.Name@y", "operator": "=", "value": "Instructions" }
  ]
}
```

Each label (`@x`, `@y`) creates a separate logical instance of the conjoint associative chain. The `associative` hint ensures each instance's `InvoiceAnnotation` table remains in the join, preventing cross-label Cartesian products.

## Summary

| Scenario | Behaviour without `associative` | Behaviour with `associative` |
|---|---|---|
| Simple many-to-many, no alternative path | Link table kept (needed for connectivity) | Link table kept |
| Multi-tenant many-to-many with shared FK to tenant | Link table may be eliminated → Cartesian product | Link table re-added after reduction |
| Conjoint annotation pattern | Link table may be eliminated → lost annotation linkage | Link table preserved per label instance |

Add the `associative` hint to any link table whose foreign-key columns connect tables that also share alternative join paths. This guarantees correct many-to-many semantics regardless of which tables a query references.