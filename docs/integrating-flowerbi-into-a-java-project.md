---
title: Integrating FlowerBI into a Java Project
status: draft
---

# Integrating FlowerBI into a Java Project

FlowerBI is designed with a **.NET server-side** and a **TypeScript/JavaScript client-side** stack.
The server engine (`FlowerBI.Engine`, `FlowerBI.Tools`) runs on .NET Core 3.1 / .NET 6+ and is written in C#.
The client packages (e.g., `flowerbi`, `flowerbi-react`) are published as npm packages intended for browser-based TypeScript applications.

This means there is **no native Java implementation** of the FlowerBI server components.
If you are building a Java application (e.g., Spring Boot, Jakarta EE) and wish to use FlowerBI's query capabilities, you have two broad approaches:

## Option 1: Run the .NET Server as a Separate Service

Deploy the FlowerBI engine as an independent HTTP service (for example, via ASP.NET Core). Your Java application can then send JSON queries to that service over HTTP and consume the results.

- Define your schema in YAML and generate the necessary C# and TypeScript support files using `FlowerBI.Tools`.
- Create a minimal .NET web API that exposes a single POST endpoint (as documented in the [README](https://github.com/danielearwicker/flowerbi)).
- From your Java code, use any HTTP client (e.g., `RestTemplate`, `WebClient`, `OkHttp`) to POST the query JSON and parse the response.

**Pros**: Full FlowerBI functionality, no reimplementation.
**Cons**: Requires maintaining a .NET runtime environment alongside your Java application.

## Option 2: Use the Client Packages in a JavaScript Frontend

If your Java project serves a web frontend (JSP, Thymeleaf, or a SPA built with React/Vue/Angular), you can embed FlowerBI’s client query logic entirely on the browser side. The Java backend only needs to provide a data endpoint that accepts FlowerBI query JSON and returns results – but that endpoint must be powered by a .NET server as in Option 1.

Alternatively, you could re‑implement the server logic in Java (decoding the JSON query, building SQL, executing it). **This is strongly discouraged** because it replicates a non‑trivial piece of software and will break if the query format evolves.

## Option 3: Use a Reverse Proxy / Sidecar

Run the FlowerBI .NET application as a sidecar container or behind a reverse proxy (e.g., Nginx, Envoy) so that your Java application can forward query requests to it. This keeps the .NET deployment isolated and simplifies lifecycle management.

## Client‑Side Only Use (Without a Dedicated Server)

If your data is already served by a REST API (e.g., Java backend exposes SQL results), you can still *format* queries using the `flowerbi` npm package on the client and pass the JSON to any endpoint that understands it. However, the endpoint must be written in .NET (or you must implement the query parsing yourself).

## Conclusion

At present, **FlowerBI does not directly support Java on the server side**. The most practical integration path is to run the .NET engine as a separate service and call it from your Java application. The client‑side packages are fully usable from any JavaScript environment, including those served by a Java web application.

If you need a pure‑Java BI solution, you may want to consider alternatives such as Apache Superset, Metabase, or a custom JDBC‑based reporting tool.
