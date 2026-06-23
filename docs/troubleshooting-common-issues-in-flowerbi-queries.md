---
title: Troubleshooting Common Issues in FlowerBI Queries
status: draft
---

# Troubleshooting Common Issues in FlowerBI Queries

This page covers the most common problems encountered when writing FlowerBI queries:
empty result sets, ambiguous join paths, and how extended/associative tables affect path resolution.

## 1. Query Returns No Rows

An empty result set can be caused by several factors:

### a) Filters Are Too Restrictive
Check that your filters are not excluding all rows inadvertently.
For example, a filter on a date column that doesn't match any data, or a combination of
filters that cannot be simultaneously satisfied.

**Debug tip:** Remove filters one by one to isolate the problematic filter.

### b) Missing Data in the Database
The expected rows may simply not exist. Verify by running the equivalent SQL directly
(using the generated SQL if available).

### c) Aggregation Without Grouping Columns
If you select only aggregated columns (e.g., `count()` without any non-aggregated column),
the query will always return a single row (the aggregate over all matching rows).
If that aggregate is zero, the result will contain one row with `0` – not empty.
An empty result only appears if no rows match at all.

### d) Joins Producing No Matches
If your query spans multiple tables, a join may eliminate all rows if foreign keys
are null or mismatched. Ensure that the foreign key columns have values and that the
referenced rows exist.

**Debug tip:** Use the playground or enable query logging to see the generated SQL
and run it directly against your database.

---

## 2. Ambiguous Join Paths (Multiple Paths Between Tables)

FlowerBI automatically builds join graphs based on foreign keys. If there are multiple
ways to join two tables (e.g., via a shared dimension table or an associative table),
the engine may pick a path that causes unexpected results or a cross-join.

### Symptom
You get unexpectedly many rows (cartesian product) or an error about ambiguous joins.

### Common Scenario: Many-to-Many with a Shared Tenant
Consider a schema where `BlogPost` and `Tag` both belong to a `Tenant`, and there is an
associative table `BlogPostTag`. If you filter on `Tenant.Id`, FlowerBI might connect
`BlogPost` and `Tag` via `Tenant` instead of via `BlogPostTag`, leading to a cross-join
of all posts and tags in that tenant.

**Solution:** Mark the associative table with the `associative` hint in your YAML schema:

```yaml
BlogPostTag:
    columns:
        BlogPostId: [BlogPost]
        TagId: [Tag]
    associative: [BlogPostId, TagId]
```

This tells FlowerBI that the table must be included even if another path exists.
See [Many-to-Many](./many-to-many.md) for details.

### Other Causes
- Using `extends` to create virtual tables (e.g., `DateReported` and `DateCompleted`)
  can introduce multiple paths if the base table has multiple foreign keys to the same
t  arget. Ensure virtual tables are used correctly (see [Virtual Tables](./virtual-tables.md)).
- If you have multiple foreign keys between the same pair of tables, you must alias them
  with different FK column names – the engine cannot disambiguate otherwise.

### Debugging Ambiguity
1.  Review the YAML schema for tables that have more than one foreign key pointing
    to the same table.
2.  Use the `associative` hint where appropriate.
3.  Simplify the query: reduce the number of tables referenced to see if the ambiguity goes away.
4.  Enable verbose logging in the server to see the join graph the engine is building.

---

## 3. How `extends` and `associative` Tables Affect Path Resolution

### Virtual Tables via `extends`
When a table uses `extends`, it creates a virtual table that shares the same physical
table name as the parent by default. This is useful for giving different meanings to
the same table (e.g., `DateRequested` and `DateCompleted`). However, the engine treats
each virtual table as a separate join node. If two virtual tables both point to the
same base table, they can create multiple paths to that table in the join graph.

**Example:**
```yaml
Date:
    id:
        Id: [datetime]
    columns:
        CalendarYearNumber: [short]

DateReported:
    extends: Date

DateResolved:
    extends: Date

WorkItem:
    id:
        Id: [int]
    columns:
        RequestedDate: [DateReported]
        CompletedDate: [DateResolved]
```
Here, `DateReported` and `DateResolved` both reference the physical `Date` table.
A query that includes both `DateReported` and `DateResolved` will produce two separate
joins to `Date`. This is correct; they are treated as distinct instances.

**Potential issue:** If you have a filter on `Date` (the base table), it will apply to
both joins simultaneously, which may not be what you want. Always filter on the virtual
table (`DateReported`) to target a specific role.

### Associative Tables
Associative tables (declared with `associative: [fk1, fk2]`) are special: they are not
removed from the join graph even if another path connects the two tables they bridge.
This prevents accidental cross-joins when a third table (like a tenant) also connects them.

**When to use:** Whenever you have a many-to-many relationship implemented via a junction
table, and that table has only the two foreign keys (plus possibly other attributes).
Declaring it as `associative` ensures the engine uses it as the connector.

**Impact on join resolution:** The engine will prefer paths that include associative tables
over alternative paths. If there are still multiple candidates, the engine picks the one
with the smallest number of joins. To force a specific path, you may need to restructure
the schema or use virtual tables to isolate the relationship.

---

## General Debugging Tips

- **Use the Playground:** The interactive demo at https://earwicker.com/flowerbi/demo/
  lets you build queries and see the generated SQL and results. Replicate your issue there.
- **Enable Engine Logging:** Set `Logging` in the `FlowerBI.Engine` options to verbose
  to see the join graph and SQL generated.
- **Check the YAML Schema:** Validate your schema with `FlowerBI.Tools` to catch syntax errors.
- **Simplify:** Start with the smallest possible query (one table, one column) and gradually
  add complexity until the problem appears.

---

## Related Documentation

- [YAML Schemas](./yaml.md) – Declaring tables, foreign keys, and `associative`.
- [Virtual Tables](./virtual-tables.md) – Using `extends` to create date roles.
- [Many-to-Many](./many-to-many.md) – The `associative` hint in detail.
- [Conjoint Tables](./conjoint.md) – Advanced virtualisation for repeated annotations.
