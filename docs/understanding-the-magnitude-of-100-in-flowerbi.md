---
title: "Understanding the Magnitude of 100 in FlowerBI"
status: draft
---

# Understanding the Magnitude of 100 in FlowerBI

When reading FlowerBI documentation, you may encounter the number **100** used as an example threshold or a unit of scale. This guide provides context on what "100" means in various scenarios, helping you gauge its significance relative to typical deployments.

## 100 in Deployment Scale

In the comparison with Power BI ([flowerbi-vs-power-bi-a-developer-centric-comparison.md](./flowerbi-vs-power-bi-a-developer-centric-comparison.md)), the number 100 appears indirectly when calculating the total number of report files for a multi-tenant, multi-language setup. For instance:

- With **1000 tenants**, **20 reports**, and **5 languages**, the total is 100,000 files.
- Scaling down to **100 tenants**, the same 20 reports and 5 languages produce **10,000 files** – still a significant number that highlights the cost of non-localised BI solutions.

Thus, **100 tenants** is a moderate multi-tenant deployment. If you have fewer than 100 tenants, the advantage of FlowerBI's client-side localisation (using i18n libraries) becomes even more pronounced.

## 100 in Data Volume

The performance guide ([performance-characteristics-and-scalability.md](./performance-characteristics-and-scalability.md)) discusses behaviour with large data volumes. While it doesn't define a specific "100" threshold, typical examples use orders of magnitude:

- **100 rows**: negligible; queries complete in milliseconds.
- **100,000 rows**: moderate; proper indexing is recommended.
- **1 million+ rows**: large; requires careful index design and possibly pagination.

So if you see "100" in a query result count, it indicates a very small dataset that won't stress the engine.

## 100 in Reporting

In the custom table example ([building-table-layouts-with-flowerbi.md](./building-table-layouts-with-flowerbi.md)), the `useQuery` hook returns a set of records. While the example doesn't specify a number, a typical report might display **100 records** per page. This is a common pagination size; FlowerBI's client-side rendering handles such volumes instantly.

## Summary

The number 100 serves as a handy benchmark in FlowerBI documentation:

- **100 tenants**: a moderate multi-tenant footprint.
- **100 rows**: a tiny dataset for performance testing.
- **100 records**: a typical page size for table displays.

Understanding these contexts helps you plan your deployment and tune performance accordingly.
