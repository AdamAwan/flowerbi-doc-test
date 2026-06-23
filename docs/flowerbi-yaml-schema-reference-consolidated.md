---
title: FlowerBI YAML Schema Reference
status: draft
---

# FlowerBI YAML Schema Reference

FlowerBI uses a YAML file to define tables, columns, relationships, and metadata. This reference covers all features with examples.

## Basic Structure

```yaml
schema: MySchema
tables:
  Customer:
    id:
      Id: [int]
    columns:
      CustomerName: [string]
      Email: [string?]
```

## Data Types

Supported: `bool`, `byte`, `short`, `int`, `long`, `float`, `double`, `decimal`, `string`, `datetime`. Append `?` for nullable.

## Foreign Keys

Use another table name as the type:
```yaml
Invoice:
  columns:
    CustomerId: [Customer]
    Amount: [decimal]
```

## Physical Names

Override with `name:` on table or second element in column type list.

## Virtual Tables (extends)

```yaml
DateRequested:
  extends: Date
```

## Associative Tables

```yaml
BlogPostTag:
  columns:
    BlogPostId: [BlogPost]
    TagId: [Tag]
  associative: [BlogPostId, TagId]
```

## Conjoint Tables

```yaml
AnnotationValue:
  conjoint: true
  id:
    Id: [int]
  columns:
    Value: [string]
```

## Documentation

Add `doc:` and `see:` on tables, columns, and topics.

## Complete Example

See the [Complete Example](complete-yaml-schema-example.md) for a full schema. (This section is now incorporated.)

*Consolidated from: describing-database-structure-in-flowerbi-yaml-schemas.md, flowerbi-yaml-schema-tables-foreign-keys-virtual-tables-and.md, complete-yaml-schema-example.md*