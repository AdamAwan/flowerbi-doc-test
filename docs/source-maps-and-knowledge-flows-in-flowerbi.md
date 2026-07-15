---
title: Source Maps and Knowledge Flows in FlowerBI
status: draft
---

# Source Maps and Knowledge Flows in FlowerBI

FlowerBI's architecture centres on a single YAML schema definition that simultaneously drives multiple representations: generated client-side TypeScript, generated server-side C#, and an in-memory runtime model that carries documentation, cross-references, and custom metadata. This document traces how data — columns, types, foreign keys, documentation, metadata, and cross-references — flows from the YAML source through code generation and runtime loading, and how the different representations map to each other.

## The YAML Schema as Single Source of Truth

Every FlowerBI installation begins with a YAML file that declares the logical database model. This file describes tables, their primary keys, column types, foreign-key relationships, nullability, virtual-table inheritance via `extends`, associative-table hints, conjoint-table declarations, and — optionally — free-text documentation (`doc`), cross-references (`see`), and custom metadata (`meta`). The YAML schema acts as the sole authoritative definition; all other artifacts are derived from it.

```yaml
schema: MySchema
tables:
  Vendor:
    name: Supplier
    id:
      Id: [int]
    columns:
      VendorName: [string, SupplierName]
  Invoice:
    id:
      Id: [int]
    columns:
      VendorId: [Vendor]
      Amount:
        type: decimal
        doc: Gross amount in original currency.
        see: [billing, Currency.IsoCode]
        meta:
          unit: GBP
          sensitive: "true"
```

A CLI entry point — the `FlowerBI.Tools` command-line tool — accepts the YAML file and produces both TypeScript and C# output. On the command line the invocation is `ts <yaml-file> <ts-file>` for TypeScript or `cs <yaml-file> <cs-file> <namespace>` for C#.

## Schema Resolution: From YAML Text to Structured Model

Before any code generation or query execution, the YAML text is passed through a resolution phase implemented by `ResolvedSchema.Resolve()`. This parser:

- Deserialises the raw YAML into a `YamlSchema` object using YamlDotNet.
- Normalises column definitions — both the short form `[type]` and the long mapping form with `type`, `name`, `doc`, `see`, and `meta` — into a uniform internal representation.
- Resolves `extends` inheritance, copying columns and metadata from parent tables and allowing the child to override specific entries.
- Validates that topic names do not collide with table names.
- Walks every column type to resolve foreign-key targets: a type value that is not a built-in data type (e.g. `Vendor` or `Currency`) is looked up against the known table names, and the column is marked as a foreign key pointing to that table's primary key.
- Populates associative-column lists from the `associative` property on each table, verifying that each referenced column exists.
- Returns a `ResolvedSchema` containing `ResolvedTable` and `ResolvedTopic` records.

## Mapping to Generated TypeScript

The TypeScript code generator reads a `ResolvedSchema` and produces a `.ts` file that exports:

- **Table objects**: one `export const` per table, each containing one property per column. Every column property is instantiated as a `QueryColumn`, `NumericQueryColumn`, `IntegerQueryColumn`, or `StringQueryColumn` depending on its data type. The column carries a `QueryColumnRuntimeType` that records its `QueryColumnDataType` enum value, its target column (for foreign keys, a string like `"Workflow.Id"`), and any custom `meta` key-value pairs.
- **Topic objects**: when the schema declares a `topics:` block, a `Topics` constant is generated with one property per topic. Each property carries JSDoc with the topic's prose text and `@see` annotations rewritten to point to `Topics.<name>` (or `Topics["..."]` for non-identifier names).
- **Schema aggregate**: a top-level constant named after the schema that groups all table exports.
- **Imports**: `QueryColumnRuntimeType` and `QueryColumnDataType` (and any needed column-class imports) are pulled from the `flowerbi` package.
- **JSDoc annotations**: tables and columns with `doc` and `see` get full JSDoc blocks. Cross-references to topics are rewritten as `@see {@link Topics.<name>}` so that IDEs can resolve the link. Cross-references to other tables or columns are emitted as-is (e.g. `@see {@link Invoice}` or `@see {@link Currency.IsoCode}`).

The generated TypeScript is designed for client-side consumption. Developers import the table objects and use their properties to build strongly typed queries with autocompletion in their IDE.

## Mapping to Generated C#

The C# code generator reads the same `ResolvedSchema` and produces a `.cs` file that:

- **Declares a static class** named after the schema, inside the specified namespace.
- **Defines one nested static class per table**, each containing `public const string` constants for every column (including the primary key). The constant values are dot-separated table-and-column strings like `"Invoice.Amount"`.
- **Generates XML doc comments** (`<summary>` and `<seealso>`) from every `doc` and `see` entry. Topic references are rewritten to `Topics.<sanitised_identifier>` for use in `<seealso cref="..."/>`. Identifiers are sanitised by replacing non-alphanumeric characters with underscores.
- **Generates a nested `Topics` class** when the schema has a `topics:` block. Each constant uses the sanitised C# identifier as the field name and the original topic name as the string value, so it can serve as a lookup key into `Schema.Topics` at runtime.
- **Embeds the full YAML text** as a triple-quoted raw string literal inside the generated class.
- **Exposes a static `Schema` property** that invokes `FlowerBI.Schema.FromYaml(YamlSchema)` once, caching the runtime model.

The generated C# is designed for server-side use. The `Schema` property gives the application code access to the full runtime model — tables, columns, foreign keys, documentation, metadata — without needing to parse the YAML again.

## The Runtime Documentation Model

When a YAML schema is loaded at runtime via `Schema.FromYaml()`, the documentation and cross-references are resolved into a live object graph. The core abstraction is the `IDocumented` interface:

```csharp
public interface IDocumented : INamed
{
    string Doc { get; }
    IReadOnlyList<IDocumented> See { get; }
}
```

Three types implement `IDocumented`:

- **`Table`** — each table carries its `Doc` text and a `See` list that may reference other tables, columns, or topics.
- **`IColumn`** — every column (including primary keys, foreign keys, and primary-foreign keys) carries its own `Doc` and `See`.
- **`Topic`** — each topic from the `topics:` block carries its `Doc` and optional `See` cross-references.

Cross-reference resolution happens in a deliberate order inside `Schema.FromYaml()`. First, all tables, columns, and topics are created as objects. Then, every `see` string list — on tables, columns, and topics — is iterated and resolved against the now-populated dictionaries:

- A string containing a dot (e.g. `"Currency.IsoCode"`) is split into table name and column name, and looked up in `_tables[table].GetColumn(column)`.
- A string without a dot is first checked against `_topics`; if a topic of that name exists, the reference resolves to that topic. Otherwise it is checked against `_tables` and resolves to that table.
- Unknown names raise a `FlowerBIException`.

This means that by the time `Schema.FromYaml()` returns, every `See` list contains live references — the actual `Table`, `Column`, or `Topic` objects — rather than strings. Consumers can walk the graph without further lookups.

## Meta Annotations

Tables and columns can carry an optional `meta` property: a free-form map of string key-value pairs. FlowerBI interprets none of these values; it simply carries them through the entire pipeline so that custom tooling can read them.

- **In the YAML**: `meta` is declared as a mapping on tables or on columns in their long form.
- **At the TypeScript level**: column `meta` becomes a third argument to `QueryColumnRuntimeType`, accessible as `Invoice.Amount.type.meta`. Table-level `meta` becomes a `$meta` key on the generated table object, readable as `Invoice.$meta`.
- **At the C# level**: `meta` is available at runtime through `Table.Meta` and `Column.Meta`, both typed as `IReadOnlyDictionary<string, string>` (empty, never null, when none was declared).

When a table extends another, table-level `meta` is merged per key: the derived table inherits all base entries, and any keys it declares itself take precedence. Columns inherited via `extends` carry their base column's `meta` unchanged.

## Virtual Tables and Inheritance Mapping

The `extends` keyword in a YAML table definition creates a virtual table — a logical alias that points to the same physical database table as its parent but establishes a distinct identity in the schema. This is the foundation of the virtual-tables pattern:

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
```

Both `DateReported` and `DateResolved` map to the same physical `Date` table, but the engine treats them as separate logical tables. This lets a `WorkItem` table declare foreign keys to `DateReported` and `DateResolved` independently, allowing the join engine to generate two distinct join paths to the same underlying table without ambiguity.

When `extends` is used, the derived table inherits the parent's columns, `doc`, `see`, and `meta`. If the derived table supplies its own `doc` or `see`, those override the parent's. Setting `see: []` explicitly clears the inherited list. The derived table may also redefine a column — if it does, the redefinition (including its `meta`) wins entirely.

## Conjoint Tables and Labelled References

A table declared with `conjoint: true` gains the ability to be virtually duplicated at query time. Wherever a conjoint table column is referenced, a label suffix (`@<label>`) creates a distinct logical instance of the entire conjoint subgraph for that query:

```yaml
AnnotationValue:
    conjoint: true
    columns:
        AnnotationNameId: [AnnotationName]
        Value: [string]
```

A query can reference `AnnotationValue.Value@shopping` and `AnnotationValue.Value@math`, and the engine fans out joins per label. Conjoint tables solve the problem of needing multiple independent instances of the same logical table within a single query without declaring duplicate virtual tables in the schema.

## The Join Engine and Table Discovery

When a query arrives, the `Query` class deserialises the JSON and loads the referenced columns via `Schema.Load()`. The `Joins` engine then:

1. Determines which labelled tables are required based on the referenced columns.
2. Builds a graph of foreign-key relationships — both forward (FKs from a table) and reverse (FKs pointing to a table).
3. Computes a set of reachable tables starting from the first required table.
4. Optionally eliminates unnecessary intermediate tables.
5. Re-adds any associative tables that have two or more associative columns pointing to surviving tables.
6. For conjoint tables, fans out arrows per unique join label.
7. Generates SQL join clauses in the correct order.

The `fullJoins` flag (exposed as `FullJoins` in the query JSON) switches from `LEFT JOIN` to `FULL JOIN` for multi-aggregation queries, preventing the non-commutative loss of rows that occurs when one aggregation produces rows the other does not.

## Summary of Knowledge Flows

The following artefacts and processes form the complete knowledge flow in FlowerBI:

1. **YAML schema**: the single source of truth — tables, columns, types, foreign keys, documentation, cross-references, metadata, associative hints, conjoint flags, virtual-table inheritance.

2. **Schema resolution** (`ResolvedSchema.Resolve`): normalises column forms, resolves `extends`, validates topic/table name collisions, resolves foreign-key targets, validates associative references.

3. **TypeScript code generation**: produces per-table column object exports with typed `QueryColumn` subclasses, `QueryColumnRuntimeType` metadata, JSDoc annotations, and a `Topics` constant.

4. **C# code generation**: produces constant string classes with XML doc comments, sanitised topic identifiers, embedded YAML, and a cached runtime `Schema` property.

5. **Runtime model loading** (`Schema.FromYaml`): builds `Table`, `IColumn`, and `Topic` objects, resolves `see` string lists into live `IDocumented` references, and merges inherited `meta`.

6. **Query execution**: accepts JSON queries, resolves column references via `Schema.GetColumn()`, generates SQL via `Query.ToSql()` with join discovery and optional `fullJoins` support, executes via Dapper, and maps results back to the typed format.

7. **SQL formatters**: `SqlServerFormatter` (OFFSET/FETCH NEXT, IIF) and `SqlLiteFormatter` (LIMIT/OFFSET, IIF) adapt generated SQL to the target database dialect. `DapperFilterParameters` embeds safe literals for numeric/bool values (aiding query-plan matching) and parameterises others.