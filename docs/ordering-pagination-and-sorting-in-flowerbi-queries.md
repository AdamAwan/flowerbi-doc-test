---
title: Ordering, Pagination, and Sorting in FlowerBI Queries
status: draft
---

# Ordering, Pagination, and Sorting in FlowerBI Queries

This article explains how to request ordered and paginated results from a FlowerBI query. It covers the syntax for the `order-by` clause, configuring the schema to enable ordering on specific columns, and combining ordering with `offset` and `limit` for pagination.

## Order-By Clause Syntax

FlowerBI queries are sent as JSON to a single POST endpoint. To specify ordering, include an `order-by` key in the query object. The value is an array of objects, each describing a sort directive:

```json
{
  "measures": [...],
  "dimensions": [...],
  "order-by": [
    { "field": "product_name", "direction": "asc" },
    { "field": "revenue", "direction": "desc" }
  ]
}
```

Each object has two properties:
- `field`: the name of a dimension or measure included in the query (must appear in `dimensions` or `measures`).
- `direction`: either `"asc"` (ascending) or `"desc"` (descending). This property is optional; the default is ascending.

The server will sort the result set by the listed fields in the order they appear.

## Configuring the Schema to Enable Ordering

Not all columns are sortable by default. To allow ordering on a dimension or measure, the schema must declare the field as sortable. In your FlowerBI schema definition (typically a TypeScript object), set the `sortable` property to `true` on the relevant field:

```typescript
const schema = {
  dimensions: {
    product_name: {
      sql: `products.name`,
      sortable: true
    },
    // other dimensions...
  },
  measures: {
    revenue: {
      sql: `SUM(order_items.price)`,
      sortable: true
    },
    // other measures...
  }
};
```

If `sortable` is not explicitly set, it defaults to `false` (or is determined by the server configuration). Querying with `order-by` on a non‑sortable field will result in an error. Always ensure the target field is marked as sortable before including it in an `order-by` clause.

## Pagination with Offset and Limit

FlowerBI supports pagination using `offset` (number of rows to skip) and `limit` (maximum number of rows to return). These are top‑level keys in the query object:

```json
{
  "measures": [...],
  "dimensions": [...],
  "order-by": [ { "field": "id", "direction": "asc" } ],
  "offset": 0,
  "limit": 50
}
```

- `offset`: integer, starting at 0 for the first page.
- `limit`: integer, the page size.

Ordering is essential for meaningful pagination when using `offset`/`limit`. Without a consistent order, the same row might appear on different pages. Always pair a `limit`/`offset` with an `order-by` clause that uniquely identifies row order (e.g., a primary key).

## Example: Sorting and Paginating a Query

Below is a complete query JSON requesting the top 10 products by revenue, sorted descending by revenue:

```json
{
  "dimensions": ["product_name"],
  "measures": ["revenue"],
  "order-by": [
    { "field": "revenue", "direction": "desc" },
    { "field": "product_name", "direction": "asc" }
  ],
  "limit": 10,
  "offset": 0
}
```

This returns the first page of the most revenue‑generating products. To get the next page, increase `offset` to 10, then 20, etc.

## Notes

- The `order-by`, `offset`, and `limit` keys are optional. If omitted, the server returns all matching rows in an unspecified order.
- The `FlowerBITable` component (from `flowerbi-react`) does **not** handle pagination or sorting internally. You must manage page state and re‑query with updated `offset`/`limit` and `order-by` clauses. For advanced interactivity, consider integrating a table library such as `react-table`.
- For details on constructing query objects and using the `useFlowerBI` hook, see the [React Hooks documentation](react-hooks-useflowerbi-and-usepagefilters.md).