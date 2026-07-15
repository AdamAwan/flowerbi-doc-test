---
title: Knowledge Flows and Documentation Infrastructure
status: draft
---

# Knowledge Flows and Documentation Infrastructure

FlowerBI embeds domain knowledge directly alongside its schema definitions, creating a structured flow of documentation from the YAML schema file through to generated TypeScript and C# code. Topics, cross-references, and metadata combine to form a navigable knowledge graph that surfaces meaning, context, and relationships to both human readers and automated tooling.

## The Documentation Model

Every YAML schema can carry free-text documentation and cross-references on three levels of granularity: the top-level **topic**, the **table**, and the **column**. The runtime model unifies these through the `IDocumented` interface, which exposes a `Doc` string and a `See` list of resolved references to other documented objects.

### Topics

A schema declares a top-level `topics:` block that defines named snippets of prose. Each topic can be a plain string or a mapping with `doc` and an optional `see` list of its own. Topics exist to capture cross-cutting concerns — the business meaning of monetary amounts, tenancy rules, or units of measure — that span multiple tables and columns.

Topic names share a namespace with table names and must not collide with them. This rule is what allows a bare name in a `see:` list to resolve unambiguously.

### Tables

Tables accept two optional keys alongside their structural properties: `doc` for free-text explanation and `see` for a list of cross-references to topics, other tables, or columns.

### Columns

Columns accept documentation in two forms. The short form (a YAML sequence such as `[decimal, FancyAmount]`) carries no documentation. The long form is a YAML mapping with a `type` key and optional `name`, `doc`, `see`, and `meta` keys. Both forms can be mixed freely within the same `columns:` block.

## How References Resolve

Every entry in a `see:` list follows a simple resolution rule. A bare name such as `Customer` is looked up first as a topic; if no topic of that name exists, it is resolved as a table. A dot-separated form such as `Customer.Id` always refers to a column on a specific table.

Resolution happens at schema load time inside `Schema.FromYaml(...)`. An unknown name produces an error — there are no dangling references at runtime. The resolved `See` lists contain the actual `Table`, `IColumn`, or `Topic` objects rather than strings, so consumers can walk the graph without further lookups.

## The Runtime Model

The `Schema` object built by `Schema.FromYaml(...)` exposes:

- `Schema.Topics` — an `IReadOnlyDictionary<string, Topic>` of all declared topics. Each `Topic` implements `IDocumented` with its own `Doc` and resolved `See` list.
- Every `Table` implements `IDocumented` with its `Doc` and resolved `See`.
- Every `IColumn` (including `Column`, `ForeignKey`, `PrimaryKey`, and `PrimaryForeignKey`) implements `IDocumented` with its `Doc` and resolved `See`.
- `Table.Meta` and `Column.Meta` carry free-form string dictionary metadata that FlowerBI does not interpret but carries through to generated code.

This graph of documented objects means that any consumer — a human browsing an introspection endpoint, an AI agent, or the code generator — can traverse from a table to a topic about billing, from a column to another column it relates to, or from a topic to every table that references it.

## Inheritance of Documentation

When a table uses `extends`, the documentation system follows consistent inheritance rules:

- **`doc`**: A derived table inherits the base table's `doc` unless it supplies its own.
- **`see`**: A derived table inherits the base table's `see` list unless it explicitly overrides it. Setting `see: []` clears the inherited list entirely.
- **Per-column documentation**: Columns inherited via `extends` carry their base column's `doc`, `see`, and `meta` unchanged. If the derived table redefines a column by supplying a column of the same name, the new definition — including its documentation — wins entirely.
- **`meta`**: Table-level metadata is merged per key: the derived table inherits all of the base table's entries, and any keys it declares itself take precedence.

## Code Generation: Documentation in TypeScript and C#

The `FlowerBI.Tools` CLI reads the resolved schema and emits documentation annotations into both target languages.

### TypeScript (JSDoc)

The TypeScript code generator writes `/** */` JSDoc blocks on every exported table constant and every column constant. Cross-references in a `see` list are rewritten from bare topic names into `Topics.<name>` forms (or `Topics["..."]` for non-identifier names) so that IDEs can follow the link. Column-level `meta` becomes a third constructor argument to `QueryColumnRuntimeType`, readable at runtime as `Invoice.Amount.type.meta`. Table-level `meta` becomes a `$meta` key on the generated table object, readable as `Invoice.$meta`. Topics are emitted as a `Topics` constant object where each property carries its JSDoc and a string value equal to the topic name.

### C# (XML Doc)

The C# code generator writes `/// <summary>` and `/// <seealso cref="..."/>` annotations. Topic identifiers are sanitised for C# (e.g. `currency-codes` becomes `currency_codes`) but the constant value retains the original name so it can be used as a key into `Schema.Topics` at runtime. The generator also embeds the full YAML schema as a triple-quoted raw string literal inside the generated class, making it available at runtime via a `static Schema` property.

## Meta: Extensible Knowledge Carriers

Beyond `doc` and `see`, every table and column can carry a `meta` dictionary of arbitrary string key-value pairs. FlowerBI does not interpret these values — it simply carries them through the entire pipeline:

- **In-memory**: Available as `IReadOnlyDictionary<string, string>` on every `Table` and `Column` (empty but never null when none was declared).
- **Generated TypeScript**: Column `meta` becomes a constructor argument to `QueryColumnRuntimeType`; table `meta` becomes a `$meta` key.
- **Generated C#**: Available at runtime through the same `Schema` model, for example `BugSchema.Schema.GetTable("Invoice").Meta["owner"]`.
- **Inheritance**: Table-level `meta` merges per key when a table extends another; the derived table's keys override the base's. Column-level `meta` is inherited unchanged through `extends`.

This makes `meta` a suitable channel for any tool-specific annotations — such as ownership tags, sensitivity labels, or display hints — that should flow from schema definition to generated code without FlowerBI needing to understand them.

## Practical Walkthrough

A schema that declares a billing topic, documents tables and columns, and cross-references them produces this knowledge flow:

1. The YAML author writes `doc` and `see` on tables and columns, and defines shared `topics:` with prose.
2. `Schema.FromYaml(...)` parses the YAML, validates references, and builds a runtime graph of `IDocumented` objects with fully resolved `See` lists.
3. `FlowerBI.Tools` emits TypeScript with JSDoc blocks whose `@see` tags link to `Topics.<name>` or to other tables, and emits C# with XML `<seealso>` elements.
4. A consumer (developer in an IDE, an introspection API, or an AI agent) can walk the graph: from a column's `See` list, discover the topic it references; from a topic's `See` list, find every table and column that references it.

This design means that the human-readable knowledge embedded in the schema is never siloed — it flows into every artefact the schema produces and remains navigable wherever the schema is loaded.