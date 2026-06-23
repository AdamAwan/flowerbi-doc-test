---
title: Building Charts with Chart.js
status: draft
---

# Building Charts with Chart.js

FlowerBI query results can be easily mapped to Chart.js datasets for rendering bar, line, pie, and other chart types. This guide shows how to transform `useQuery` records into the format expected by [react-chartjs-2](https://github.com/jerairrest/react-chartjs-2) (the React wrapper for Chart.js).

## Prerequisites

- A FlowerBI query set up with `useQuery` from `flowerbi-react`
- Chart.js and react-chartjs-2 installed:
  ```bash
  npm install chart.js react-chartjs-2
  ```

## Basic Example: Bar Chart

Assume you have a query that returns the number of bugs per customer:

```tsx
import { useQuery } from 'flowerbi-react';
import { Bar } from 'react-chartjs-2';
import { Chart as ChartJS, CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend } from 'chart.js';

ChartJS.register(CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend);

function BugBarChart() {
  const { records } = useQuery(fetch, {
    select: {
      customer: Customer.CustomerName,
      bugCount: Bug.Id.count(),
    },
    filters: [],
  });

  const data = {
    labels: records.map(r => r.customer),
    datasets: [
      {
        label: 'Bugs',
        data: records.map(r => r.bugCount),
        backgroundColor: 'rgba(53, 162, 235, 0.5)',
      },
    ],
  };

  return <Bar data={data} />;
}
```

## Line Chart

For a line chart over time, you might group by a date column:

```tsx
const { records } = useQuery(fetch, {
  select: {
    date: DateReported.Id,
    bugCount: Bug.Id.count(),
  },
  filters: [],
});

const data = {
  labels: records.map(r => r.date.toLocaleDateString()),
  datasets: [
    {
      label: 'Bugs Reported',
      data: records.map(r => r.bugCount),
      borderColor: 'rgb(255, 99, 132)',
      backgroundColor: 'rgba(255, 99, 132, 0.5)',
    },
  ],
};
```

## Multi-bar Chart (with conditioned aggregations)

Use separate aggregation functions with different filters to create grouped bars:

```tsx
const { records } = useQuery(fetch, {
  select: {
    customer: Customer.CustomerName,
    allBugs: Bug.Id.count(),
    resolvedBugs: Bug.Id.count([Workflow.Resolved.equalTo(true)]),
  },
  filters: [],
});

const data = {
  labels: records.map(r => r.customer),
  datasets: [
    {
      label: 'All Bugs',
      data: records.map(r => r.allBugs),
      backgroundColor: 'rgba(255, 99, 132, 0.5)',
    },
    {
      label: 'Resolved Bugs',
      data: records.map(r => r.resolvedBugs),
      backgroundColor: 'rgba(53, 162, 235, 0.5)',
    },
  ],
};
```

## Using with `flowerbi-react-chartjs`

FlowerBI also offers a dedicated package `flowerbi-react-chartjs` with pre-built chart components. See its [API documentation](https://earwicker.com/flowerbi/typedoc/flowerbi-react-chartjs) for a higher-level integration.

## Key Points

- `records` is an array of strongly typed objects matching your `select`.
- Map each field to `labels` (for x-axis or category) and `datasets[].data` (for values).
- Multiple datasets can be added for grouped/comparison charts.
- For date formatting, consider using `moment` or `Intl.DateTimeFormat`.

Now you can render interactive charts with minimal boilerplate.