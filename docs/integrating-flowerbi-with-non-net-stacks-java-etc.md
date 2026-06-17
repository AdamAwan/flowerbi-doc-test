---
title: Integrating FlowerBI with Non-.NET Stacks
status: draft
---

# Integrating FlowerBI with Non-.NET Stacks

## Architecture Overview

FlowerBI consists of two primary components:
- **Server-side**: The `FlowerBI.Engine` library (and related tools) is built on .NET Core / .NET 5+. It processes query JSON and generates SQL against the configured database.
- **Client-side**: The client libraries (`flowerbi`, `flowerbi-react`, etc.) are TypeScript/JavaScript packages that run in the browser or Node.js. They construct query objects and send them to the server via HTTP.

**Important**: There is no Java (or Python, Ruby, etc.) server-side engine. The server component is strictly .NET. However, any language that can make HTTP requests can act as a client by sending properly formatted query JSON to a .NET endpoint.

## How a Java Application Can Use FlowerBI

A Java application (web backend, desktop, etc.) can consume FlowerBI data by sending queries to an existing FlowerBI server endpoint. The server must be a .NET application hosting the `FlowerBI.Engine` middleware.

### Steps:

1. **Set up a FlowerBI server** (if not already present). This is a .NET application that exposes a single POST route (e.g., `/query`) accepting a JSON body conforming to the FlowerBI query schema. The server is responsible for parsing the schema, validating, and executing SQL against the database.

2. **Construct the query JSON** in Java. The JSON structure is the same as used by the TypeScript client. For example:
```json
{
  "select": ["Customer.CustomerName"],
  "aggregations": [{"column": "Bug.Id", "function": "Count"}],
  "filters": []
}
```

3. **Send HTTP POST** from Java. Use standard HTTP libraries like `java.net.HttpURLConnection`, Apache HttpClient, or OkHttp.

4. **Parse the response**. The server returns a JSON array of record objects. Use any JSON parsing library (Jackson, Gson, etc.) to deserialize into Java objects.

### Example Java Snippet (using OkHttp):

```java
OkHttpClient client = new OkHttpClient();

String queryJson = "{\"select\":[\"Customer.CustomerName\"],\"aggregations\":[{\"column\":\"Bug.Id\",\"function\":\"Count\"}],\"filters\":[]}";

RequestBody body = RequestBody.create(queryJson, MediaType.parse("application/json; charset=utf-8"));

Request request = new Request.Builder()
    .url("https://your-flowerbi-server.com/query")
    .post(body)
    .addHeader("Authorization", "Bearer <token>") // if needed
    .build();

try (Response response = client.newCall(request).execute()) {
    if (response.isSuccessful()) {
        String jsonData = response.body().string();
        // Parse using Jackson:
        // ObjectMapper mapper = new ObjectMapper();
        // List<Map<String, Object>> records = mapper.readValue(jsonData, ...);
    }
}
```

## Limitations

- **No Java server SDK**: You cannot run `FlowerBI.Engine` inside a Java process. The engine is .NET-only.
- **Schema generation**: The `FlowerBI.Tools` generate TypeScript and C# definitions. For Java, you would need to manually replicate the schema types or generate equivalents (e.g., using JSON schema).
- **Row-level security**: Must be implemented on the .NET server side (e.g., by appending filters per user). The Java client only sends the query; server applies extra constraints.

## Alternative: Embedding the .NET Engine in a Sidecar

If you must avoid a separate .NET service, you could run the FlowerBI engine as a sidecar process (e.g., via a .NET Core executable) and communicate with it via standard input/output or a local HTTP endpoint. This adds operational complexity but allows Java to remain the primary backend.

## Conclusion

FlowerBI's architecture makes it straightforward for any HTTP-capable client, including Java, to query data. The .NET server is a necessary backend component. There is no plan to port the engine to other languages, but the RESTful nature of the query API makes integration simple.
