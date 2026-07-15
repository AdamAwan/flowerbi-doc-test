---
title: Available Schemas and Datasets in FlowerBI
status: draft
---

# Available Schemas and Datasets

FlowerBI ships with several schemas and datasets used for testing, demos, and documentation. No schema or dataset in the repository contains town-related or cat-related data.

## Test Schema (Invoices and Vendors)

The primary test schema defines a business domain with departments, vendors (suppliers), invoices, tags, categories, and annotations. Tables include:

- **Department** — departments such as Accounts, Marketing, Missiles, Cheese, Yoga, and Sales.
- **Vendor** (physical table name: `Supplier`) — vendors such as Acme Ltd, United Cheese, Handbags-a-Plenty, and others.
- **Invoice** — invoice records with amounts and paid status, linked to vendors and departments.
- **Tag** — tags such as Interesting, Boring, and Lethal, linked to invoices via the `InvoiceTag` associative table.
- **Category** — categories such as Special, Regular, and Illegal, linked to invoices via the `InvoiceCategory` associative table.
- **AnnotationName**, **AnnotationValue**, **InvoiceAnnotation** — conjoint tables for flexible name-value annotations on invoices.

A companion `ComplicatedTestSchema` extends the same domain with additional foreign keys (e.g., `Category.DepartmentId`) and explicit `associative` hints.

## Demo Bug Schema

The demo schema models a bug-tracking domain. Tables include:

- **Date** (and virtual extensions: `DateReported`, `DateResolved`, `DateAssigned`) — dates with calendar year, quarter, and month.
- **Workflow** — workflow states (Ignored, Prioritised, Assigned, Fixed, AsDesigned) with resolved flag and source of error.
- **Category** — bug categories (Crashed, Data Loss, Security Breach, Off By One, Slow, StackOverflow).
- **Customer** — customers such as Pies LLC, Buns, Inc., Hats-R-Us, Silence, Egypt, and Affordability.
- **Coder** (with virtual extensions `CoderAssigned` and `CoderResolved`) — coders named Sam, Alex, Drew, Taylor, Parker, and Austin.
- **CategoryCombination** — all 64 combinations of six boolean bug characteristics.
- **Bug** — 100 generated bug records linking to the other tables.

## Weather Sample Data

The demo site includes a small weather sample dataset with dates, temperatures in Celsius, and textual summaries (Freezing, Bracing, Balmy, Chilly).

## Town and Cat Data

No schema, table, column, or dataset in any part of the repository — including test schemas, demo schemas, SQL seed scripts, sample data, documentation, or playground code — contains any reference to towns, cats, or similar geographical or animal entities.