---
title: Mapping Records Directly for Custom Table Layouts
status: draft
---

# Mapping Records Directly for Custom Table Layouts

`FlowerBITable` is a quick way to render query results as a plain HTML table. For more control over the layout, styling, or non-tabular visualizations, you can bypass `FlowerBITable` and iterate over the `records` array returned by `useQuery` directly.

This approach works well when you need:

- Custom column ordering or headers
- Conditional formatting (e.g., highlighting cells based on values)
- Mixed content (e.g., embedding charts or links within cells)
- Non-standard table structures (e.g., grouped rows, matrix layouts)

## Worked Example: A Styled Custom Table

Assume you have the following query (from the FlowerBI Playground or a real data source):

```tsx
import { useQuery } from 'flowerbi-react';
import { Customer, Bug } from './generated/schema'; // generated from your YAML
import { localFetch } from './fetch';

function CustomTableExample() {
    const { records } = useQuery(localFetch, {
        select: {
            customer: Customer.CustomerName,
            bugCount: Bug.Id.count(),
            resolvedCount: Bug.Id.count([Bug.Resolved.equalTo(true)]),
        },
    });

    // `records` is an array of objects like:
    // { customer: string, bugCount: number, resolvedCount: number }

    return (
        <table style={{ borderCollapse: 'collapse', width: '100%' }}>
            <thead>
                <tr>
                    <th>Customer</th>
                    <th>Total Bugs</th>
                    <th>Resolved</th>
                    <th>Resolution Rate</th>
                </tr>
            </thead>
            <tbody>
                {records.map((record) => (
                    <tr key={record.customer}>
                        <td>{record.customer}</td>
                        <td>{record.bugCount}</td>
                        <td>{record.resolvedCount}</td>
                        <td>
                            {record.bugCount > 0
                                ? `${Math.round(
                                      (record.resolvedCount / record.bugCount) * 100
                                  )}%`
                                : 'N/A'}
                        </td>
                    </tr>
                ))}
            </tbody>
        </table>
    );
}
```

### What’s happening:

1. **`useQuery`** returns an object with `records` – an array strongly typed to match your `select` shape.
2. **We map over `records` directly**, rendering a `<tr>` for each item.
3. We can **compute derived values** (here, the resolution rate) that aren't in the raw query result.
4. The **key** attribute uses a unique value per row (here, `customer` – adjust if names aren’t unique).

The result is a fully controlled HTML table with custom styling and logic.

## Using Row/Column Components vs Direct Mapping

The `Row` and `Column` components from `flowerbi-react` are designed for **flexible grid layouts** – they don’t produce `<table>` elements. If you need a genuine HTML table (for accessibility, sorting libraries, or column-based alignment), direct mapping is the better choice.

Use `Row`/`Column` when:

- You want a dashboard-like arrangement of independent blocks.
- You don’t need column headers or row/column alignment.

Use direct mapping over `records` when:

- You need a traditional table with headers and cells.
- You want to apply per-cell formatting or calculations.
- You plan to integrate with a table library (e.g., `react-table`, `ag-grid`).

## Handling Empty Records

Always handle the case when the query hasn’t returned data yet or yields no results:

```tsx
if (!records) {
    return <p>Loading…</p>;
}

if (records.length === 0) {
    return <p>No results found.</p>;
}
```

## Summary

- Directly mapping over `records` is a simple, powerful way to build custom table layouts.
- Combine with inline CSS or a library like styled-components for fine-grained control.
- For complex interactivity (sorting, filtering), consider pairing with an existing table library – the `records` array can be passed directly as the data source.

> **Note:** This approach works with any query that uses `useQuery` or raw `Query.execute`. The same pattern applies if you receive records from `QueryResultJson`.
