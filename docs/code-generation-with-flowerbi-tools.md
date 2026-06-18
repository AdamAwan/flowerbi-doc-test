---
title: Code Generation with FlowerBI.Tools
status: draft
---

# Code Generation with FlowerBI.Tools

`FlowerBI.Tools` is the code generation utility that reads a FlowerBI YAML schema and produces strongly-typed TypeScript and C# files. These generated files give your client code auto-completion and type inference, and your server code the ability to run queries safely.

## Prerequisites

- .NET 6.0 SDK or later (the tool runs on any .NET-compatible platform)
- A valid FlowerBI YAML schema file (see [YAML Schemas](./yaml.md) for the format)

## Installing the tool

`FlowerBI.Tools` is available as a .NET global tool. Install it with:

```bash
dotnet tool install --global FlowerBI.Tools
```

To verify installation:

```bash
dotnet flowerbi-tools --version
```

## Command-line usage

The tool is invoked as `dotnet flowerbi-tools` followed by a command. The primary command is `generate`.

```bash
dotnet flowerbi-tools generate [options]
```

### Required options

| Option | Short | Description |
|--------|-------|-------------|
| `--input` | `-i` | Path to the YAML schema file (e.g., `schema.yaml`) |

### Optional options

| Option | Short | Description |
|--------|-------|-------------|
| `--output` | `-o` | Directory where generated files will be written (defaults to `.`) |
| `--language` | `-l` | Target language: `typescript`, `csharp`, or `both` (default: `both`) |
| `--namespace` | `-n` | C# namespace for generated classes (default: `FlowerBI.Schema`) |
| `--prefix` | `-p` | Optional prefix for generated TypeScript variable names (e.g., `Schema`) |

### Examples

**Generate both TypeScript and C# files in the current directory:**

```bash
dotnet flowerbi-tools generate -i schema.yaml
```

**Generate only TypeScript into a specific folder:**

```bash
dotnet flowerbi-tools generate -i schema.yaml -o ./generated/ts -l typescript
```

**Generate C# with a custom namespace:**

```bash
dotnet flowerbi-tools generate -i schema.yaml -o ./generated/cs -l csharp -n MyApp.Data
```

### Output files

Assuming the schema is named `TestSchema` (from the `schema:` field in YAML), the tool produces:

| Language   | File name          | Contents |
|------------|-------------------|----------|
| TypeScript | `TestSchema.ts`   | Exports constants for every table, column, and aggregation function with full type definitions |
| C#         | `TestSchema.cs`   | Static classes with string constants for table/column names and a `ColumnTypes` class for type metadata |

If the schema has a `topic:` section, a separate file (e.g., `Topics.ts`, `Topics.cs`) may be generated with documentation constants.

## Integrating into a build

### MSBuild target (for .NET projects)

You can integrate code generation into your .NET project by adding an MSBuild target in your `.csproj` file. The following example assumes you have the `dotnet flowerbi-tools` command available and your schema is at `schema.yaml`:

```xml
<Target Name="GenerateFlowerBITypes" BeforeTargets="BeforeBuild">
  <Exec Command="dotnet flowerbi-tools generate -i schema.yaml -o $(MSBuildProjectDirectory)/Generated -n MyApp.Data" />
</Target>
```

Add the generated files to your project (or include the output directory with ` <ItemGroup><Compile Include="Generated/**/*.cs" /></ItemGroup>`) to have them compiled automatically.

### NPM script (for TypeScript projects)

If you prefer to run generation from an npm script, add a script to your `package.json`:

```json
{
  "scripts": {
    "generate-schema": "dotnet flowerbi-tools generate -i schema.yaml -o ./src/generated -l typescript"
  }
}
```

Then run `npm run generate-schema` or `yarn generate-schema` before building.

### CI/CD pipelines

In your CI pipeline, install the tool and run generation before the build step:

```bash
dotnet tool install --global FlowerBI.Tools
dotnet flowerbi-tools generate -i schema.yaml -o ./generated
```

## Tool configuration via YAML

Alternatively, you can place a `flowerbi-tools.json` file in your project root to set defaults. Example:

```json
{
  "input": "schema.yaml",
  "output": "./Generated",
  "language": "both",
  "namespace": "MyApp.Data"
}
```

Then simply run:

```bash
dotnet flowerbi-tools generate
```

## Example workflow

1. Write your schema in `schema.yaml`.
2. Run `dotnet flowerbi-tools generate -i schema.yaml -o ./Generated -l both -n MyApp.FlowerBI`.
3. In your TypeScript client, import the generated file:

```ts
import { Vendor, Invoice } from "./Generated/TestSchema";
```

4. In your C# server, reference the generated classes:

```csharp
using MyApp.FlowerBI;
// Use Vendor.Id, Invoice.Amount etc.
```

5. Re-run generation whenever the schema changes.

## Troubleshooting

- **"The tool could not be found"** – Ensure the tool is installed globally and that your PATH includes the .NET global tools directory (usually `~/.dotnet/tools`).
- **"Input file not found"** – Verify the `--input` path is correct relative to the current working directory.
- **TypeScript file not generated for TypeScript output** – Check that the schema has at least one table; the generator produces an empty file if no tables are present.

## See also

- [YAML Schemas](./yaml.md)
- [Documenting your schema](./documentation.md)
- [FlowerBI.Tools GitHub repository](https://github.com/danielearwicker/flowerbi/tree/main/server/dotnet/FlowerBI.Tools)