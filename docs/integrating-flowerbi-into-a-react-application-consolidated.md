---
title: Integrating FlowerBI into a React Application
status: draft
---

# Integrating FlowerBI into a React Application

This guide covers the full React integration: installation, hooks, tables, charts, and custom rendering.

## Installation

```bash
npm install flowerbi-react flowerbi-react-chartjs
```

## Fetch Function

```ts
import { QueryJson, QueryResultJson } from "flowerbi";

export async function myFetch(queryJson: QueryJson): Promise<QueryResultJson> {
  const response = await fetch("/api/query", { method: "POST", ... });
  return response.json();
}
```

## Hooks

### useFlowerBI

```ts
const { records, loading, error } = useFlowerBI(fetch, {
  select: { customer: Customer.CustomerName, bugCount: Bug.Id.count() },
  filters: []
});
```

### usePageFilters

Share filters across components:
```tsx
const { filters, addFilter, clearFilters } = usePageFilters();
```

## Tables

### FlowerBITable
```tsx
<FlowerBITable records={records} />
```

### Custom Table
```tsx
<table>
  <thead><tr><th>Customer</th><th>Bugs</th></tr></thead>
  <tbody>
    {records.map(r => <tr key={r.customer}><td>{r.customer}</td><td>{r.bugCount}</td></tr>)}
  </tbody>
</table>
```

## Charts (Chart.js)

```tsx
const data = {
  labels: records.map(r => r.customer),
  datasets: [{ label: 'Bugs', data: records.map(r => r.bugCount) }]
};
return <Bar data={data} />;
```

For pre-built charts, see `flowerbi-react-chartjs`.

*Consolidated from: building-charts-with-chart-js.md, building-table-layouts-with-flowerbi.md, integrating-flowerbi-into-a-react-application.md, react-hooks-useflowerbi-and-usepagefilters.md*