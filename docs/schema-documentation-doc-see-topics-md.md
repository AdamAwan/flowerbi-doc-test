---
title: Schema Documentation with doc, see, and Topics
status: draft
---

# Schema Documentation with `doc`, `see`, and Topics

FlowerBI YAML schemas can carry human-readable descriptions and cross-references on tables, columns, and a top-level `topics:` section. These annotations are surfaced at runtime via the `IDocumented` interface and flow into generated TypeScript (as JSDoc `@see` annotations) and C# (as XML `<summary>` / `<seealso>` comments).

## Adding `doc` and `see` on Tables

Tables accept two optional keys alongside `id`, `columns`, `extends`, and other properties:

- **`doc`** (string): free-text explanation of what the table represents.
- **`see`** (list of strings): cross-references to topics, other tables, or specific columns on other tables (`Table.Column`).

Multi-line prose is best written with YAML's literal block syntax (`|`):

```yaml
Invoice:
    doc: |
        One row per finalised invoice. Pending and draft invoices live
        in the operational `InvoiceDraft` table and are not visible here.
    see: [billing, Customer]
```

## Adding `doc` and `see` on Columns

Columns support two syntactic forms:

**Short form** — a YAML sequence (unchanged from earlier versions):

```yaml
Amount: [decimal, FancyAmount]
```

**Long form** — a YAML mapping, which allows attaching `doc` and `see`:

```yaml
Amount:
    type: decimal
    name: FancyAmount        # optional, same role as the second short-form element
    doc: Gross amount in original currency.
    see: [billing, Currency.IsoCode]
```

Only `type` is required in the long form. Short and long forms can be mixed freely within the same `columns:` block.

## Top-Level `topics:` Section

The top-level `topics:` key defines named, reusable prose snippets that can be referenced from any `see:` list across the schema. Each topic can be written in one of two forms:

**Short form** — a plain string:

```yaml
topics:
    tenancy: |
        Row-level security restricts visibility per user.
```

**Long form** — a mapping with `doc` and an optional `see` of its own (topics can reference other topics, tables, or columns):

```yaml
topics:
    billing:
        doc: |
            Monetary columns are in the invoice's original currency unless
            the column name ends in `USD`.
        see: [Currency.IsoCode]
```

Topic names share a namespace with table names: a topic **must not** have the same name as a table. The schema parser raises an error if they collide, which is what lets a bare name in a `see:` list resolve unambiguously.

## See-Reference Resolution

Each entry in a `see:` list resolves as follows:

| Form              | Meaning                                                    |
| ----------------- | ---------------------------------------------------------- |
| `Customer`        | A topic if a topic of that name exists, otherwise a table. |
| `Customer.Id`     | The column `Id` on table `Customer`.                       |

Resolution happens at schema parse time. An unknown name is a parse error.

Specifically, `Schema.FromYaml(...)` (in `Schema.cs`) processes see lists in a third pass, after all tables, columns, and topic objects have been created. The `ResolveSeeList` method:

1. Checks for a dot separator; if present, splits into `Table.Column` and looks up the column.
2. Otherwise checks the topic dictionary first, then the table dictionary.
3. Throws `FlowerBIException` if the reference cannot be resolved.

This produces a fully resolved graph of `IDocumented` references — consumers can walk `See` lists without needing further lookups or string-parsing.

## Inheritance via `extends`

When a table uses `extends`, it inherits the parent table's `doc` and `see` list (and per column, inherited columns inherit the parent column's `doc` and `see`). The child can override any of these:

```yaml
DateReported:
    extends: Date
    doc: The date a bug was first reported.   # overrides Date's doc
    # see: is inherited from Date (no override)
```

Setting `see: []` explicitly clears the inherited list.

Implementation detail (from `ResolvedSchema.Resolve`): if a child table does not supply its own `see` list, it inherits the parent's; if it does supply one (including an empty list), the child's list is used as-is. The same inheritance pattern applies to column-level `doc` and `see` via the `Extends` property on `ResolvedColumn`.

## The `IDocumented` Interface

At runtime, three types implement `IDocumented`, which extends `INamed` (`DbName` + `RefName`) and adds:

- `string Doc` — free-text explanation.
- `IReadOnlyList<IDocumented> See` — resolved cross-references.

| Type      | Role                                                    |
| --------- | ------------------------------------------------------- |
| `Table`   | Represents a resolved YAML table (implements `IDocumented`). |
| `Column`  | Represents a column (implements `IColumn`, which extends `IDocumented`). |
| `Topic`   | Represents a topic entry from the `topics:` section.     |

`Schema.Topics` is an `IReadOnlyDictionary<string, Topic>`.

## Runtime Introspection Example

`Schema.FromYaml(...)` returns a fully resolved graph where every `See` list holds direct object references (to `Table`, `IColumn`, or `Topic` instances), not strings. This enables runtime introspection without further lookups:

```csharp
var schema = Schema.FromYaml(yaml);
var amount = schema.GetTable("Invoice").GetColumn("Amount");

Console.WriteLine(amount.Doc);
foreach (var related in amount.See)
{
    Console.WriteLine($"  see also {related.RefName} ({related.GetType().Name})");
}

foreach (var topic in schema.Topics.Values)
{
    Console.WriteLine($"{topic.RefName}: {topic.Doc}");
}
```

## Generated TypeScript (JSDoc)

The `FlowerBI.Tools` CLI (command `ts`) emits TypeScript constants with JSDoc `@see` annotations. Topic references in `see:` lists are rewritten to target a `Topics` constant object — for example, `see: [billing]` becomes `@see {@link Topics.billing}`. Table and column references remain as-is (`@see {@link Invoice}`, `@see {@link Customer.CustomerName}`).

```ts
/**
 * One row per billed customer.
 * @see {@link Topics.tenancy}
 * @see {@link Invoice}
 */
export const Customer = {
    /**
     * Display name.
     */
    CustomerName: new StringQueryColumn<string>("Customer.CustomerName", ...),
};

export const Topics = {
    /**
     * Monetary columns are in the invoice's original currency...
     */
    billing: "billing",
    ...
};
```

For topic names that are not valid JavaScript identifiers (e.g. `currency-codes`), the generated code uses bracket notation: `Topics["currency-codes"]` for both the constant definition and the `@see` link.

## Generated C# (XML Doc Comments)

The `FlowerBI.Tools` CLI (command `cs`) emits a static class hierarchy with XML `<summary>` and `<seealso>` annotations. Topic references are rewritten to `Topics.<sanitised-identifier>`:

```csharp
/// <summary>One row per billed customer.</summary>
/// <seealso cref="Topics.tenancy"/>
/// <seealso cref="Invoice"/>
public static class Customer
{
    /// <summary>Display name.</summary>
    public const string CustomerName = "Customer.CustomerName";
}

public static class Topics
{
    /// <summary>Monetary columns are in the invoice's original currency...</summary>
    public const string billing = "billing";
}
```

Topic identifiers are sanitised for C# compatibility: characters that are not letters, digits, or underscores are replaced with `_`. For example, `currency-codes` becomes `currency_codes`. The constant **value** remains the original topic name (e.g. `"currency-codes"`), so it can be used as a key into `Schema.Topics` at runtime.

XML special characters in `doc` text (`<`, `>`, `&`) are automatically escaped (`&lt;`, `&gt;`, `&amp;`) during code generation.