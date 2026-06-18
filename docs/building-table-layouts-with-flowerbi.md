---
title: Building Table Layouts with FlowerBI
status: draft
---

# Building Table Layouts with FlowerBI

FlowerBI provides a `FlowerBITable` component (from `flowerbi-react`) that renders query results as a plain HTML table. For more complex layouts, you can also use the `Row` and `Column` components or directly map over records to build custom tables. This guide covers both approaches with code samples.

## Prerequisites

- Ensure you have `flowerbi-react` installed:
  ```bash
  npm install flowerbi-react
  ```
- Your backend must expose a query endpoint that accepts the JSON query and returns `QueryResultJson`.

## Using `FlowerBITable`

`FlowerBITable` is a simple component that takes a query and optional configuration and renders a `<table>`.

### Basic Example

```tsx
import { useQuery, FlowerBITable } from 'flowerbi-react';
import { fetch } from './api';
import { Customer, Bug } from './generated-schema'; // generated from your YAML schema

function BugTable() {
  const { records, loading } = useQuery(fetch, {
    select: {
      customer: Customer.CustomerName,
      bugCount: Bug.Id.count(),
    },
    filters: [Workflow.Resolved.equalTo(true)],
  });

  if (loading) return <div>Loading...</div>;

  return <FlowerBITable records={records} />;
}
```

The component automatically uses the keys of the `select` object as column headers. Each record becomes a row.

### Custom Column Headers

You can override the header labels:
```tsx
<FlowerBITable
  records={records}
  headers={['Customer Name', 'Resolved Bugs']}
/>
```

## Using `Row` and `Column` Components

The `Row` and `Column` components from `flowerbi-react` help you create flexible grid-based layouts without a `<table>` element. They work well for dashboard-style views.

```tsx
import { Row, Column } from 'flowerbi-react';

function Dashboard() {
  const { records } = useQuery(fetch, { /* ... */ });

  return (
    <Row>
      <Column size={6}>
        <FlowerBITable records={records} />
      </Column>
      <Column size={6}>
        {/* Other content like a chart */}
      </Column>
    </Row>
  );
}
```

`Column` accepts a `size` prop (1-12) to define the width. By default, `Row` lays out columns in a flex container.

## Custom Table Rendering with Records

If you need full control over the table markup (e.g., custom formatting, conditional styling), map over the records directly.

```tsx
function CustomBugTable({ records }: { records: BugRecord[] }) {
  return (
    <table>
      <thead>
        <tr>
          <th>Customer</th>
          <th>Resolved Bugs</th>
        </tr>
      </thead>
      <tbody>
        {records.map((row, i) => (
          <tr key={i}>
            <td>{row.customer}</td>
            <td>{row.bugCount.toLocaleString()}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

The records array contains strongly typed objects that match your `select` shape.

## Pagination and Sorting

`FlowerBITable` itself does not handle pagination or sorting. You can implement these by managing state and re-querying with offset/limit and order-by clauses (if your schema supports them). For simple cases, consider using a library like `react-table` with the records array.

## Summary

- Use `FlowerBITable` for quick, no-frills tables.
- Use `Row` and `Column` for responsive grid layouts.
- For total control, iterate over the records array directly.

All components work with the strongly typed results from `useQuery`, ensuring type safety in your templates.
