# Incremental Source Generator Development Guide

This guide is focused on **Incremental Source Generators**. The older
`ISourceGenerator` API should be treated as deprecated for new work: it runs as a
coarser `Initialize`/`Execute` pass, gives the host fewer opportunities to cache
work, and scales poorly in large IDE sessions compared with `IIncrementalGenerator`.
Use the old API only when maintaining existing generators that cannot yet be
migrated.

Primary sources used for this guide:

- Roslyn design document, *Incremental Generators*:
  <https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md>
- Roslyn design document, *Incremental Generators Cookbook*:
  <https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.cookbook.md>
- Andrew Lock, *Creating a source generator* series:
  <https://andrewlock.net/series/creating-a-source-generator/>

## Table of contents

- [Quick start checklist](#quick-start-checklist)
- [1. What incremental source generators are](#1-what-incremental-source-generators-are)
- [2. Execution model](#2-execution-model)
- [3. Recommended project configuration](#3-recommended-project-configuration)
- [4. Minimal generator skeleton](#4-minimal-generator-skeleton)
- [5. Pipeline API essentials](#5-pipeline-api-essentials)
- [6. Syntax discovery](#6-syntax-discovery)
- [7. Marker attributes](#7-marker-attributes)
- [8. Building cache-friendly models](#8-building-cache-friendly-models)
- [9. Emitting source](#9-emitting-source)
- [10. Diagnostics and analyzers](#10-diagnostics-and-analyzers)
- [11. AdditionalFiles](#11-additionalfiles)
- [12. Analyzer config and MSBuild properties](#12-analyzer-config-and-msbuild-properties)
- [13. Packaging and deployment](#13-packaging-and-deployment)
- [14. Testing generators](#14-testing-generators)
- [15. Performance checklist](#15-performance-checklist)
- [16. AOT, trimming, and runtime-code replacement](#16-aot-trimming-and-runtime-code-replacement)
- [17. Interceptors, UnsafeAccessor, and advanced runtime features](#17-interceptors-unsafeaccessor-and-advanced-runtime-features)
- [18. Debugging and generated output](#18-debugging-and-generated-output)
- [19. Troubleshooting](#19-troubleshooting)
- [20. Source map to Andrew Lock's series](#20-source-map-to-andrew-locks-series)
- [21. Pre-publish checklist](#21-pre-publish-checklist)
- [22. Key takeaways](#22-key-takeaways)

## Quick start checklist

When creating a new source generator, verify the following first:

- Implement `IIncrementalGenerator`, not `ISourceGenerator`.
- Target `netstandard2.0` unless every supported host can load a newer TFM.
- Pin `Microsoft.CodeAnalysis.CSharp` to the lowest minor version that exposes
  the APIs you need. The referenced Roslyn package version becomes your minimum
  host version.
- Mark the generator with `[Generator(LanguageNames.CSharp)]` or another
  supported language list.
- Keep the generator stateless. Store no per-compilation data in instance or
  static mutable fields.
- Define all work as a pipeline inside `Initialize`; do not do real generation
  work in `Initialize` itself.
- Prefer `SyntaxProvider.ForAttributeWithMetadataName` for syntax-driven
  generators.
- Extract symbols and syntax into small equatable model values as early as
  possible.
- Do not carry `ISymbol`, `Compilation`, `SemanticModel`, `SyntaxNode`, or
  `Location` values deep into the pipeline unless there is no alternative.
- Use `RegisterPostInitializationOutput` for marker attributes and bootstrap
  code that does not depend on user source.
- Use `RegisterSourceOutput` for generated source that affects the compilation.
- Consider `RegisterImplementationSourceOutput` for implementation-only source
  where IDE hosts can safely skip it during analysis.
- Generate text with a deterministic writer or `StringBuilder`; avoid building
  large syntax trees and calling `NormalizeWhitespace` for generated output.
- Pass cancellation tokens to Roslyn APIs and check cancellation in expensive
  loops.
- Add unit tests for generated output, diagnostics, configuration, and pipeline
  cacheability.
- Package generator DLLs under `analyzers/dotnet/cs` or a suitable
  `roslyn{version}` analyzer folder.
- Validate the package in both `dotnet build` and Visual Studio live analysis.

## 1. What incremental source generators are

An incremental source generator is a compiler extension that adds C# source code
to the user's compilation through a deterministic, cacheable pipeline.

Core concepts:

- **`IIncrementalGenerator`**: generator entry point.
- **`IncrementalGeneratorInitializationContext`**: registration surface used to
  define the pipeline.
- **`IncrementalValueProvider<T>`**: provider of one value.
- **`IncrementalValuesProvider<T>`**: provider of zero or more values.
- **`SyntaxValueProvider`**: optimized syntax discovery surface.
- **`SourceProductionContext`**: output context used to call `AddSource` and,
  when necessary, `ReportDiagnostic`.
- **`RegisterPostInitializationOutput`**: emits source before the main pipeline
  runs; useful for generated attributes.
- **`RegisterSourceOutput`**: emits source that is part of the compilation.
- **`RegisterImplementationSourceOutput`**: emits implementation source that a
  host may skip during IDE-only analysis.

Source generators are **additive only**. They can add source files, but they
cannot modify existing user files. They are not a replacement for IL weaving,
post-build rewriting, runtime monkey patching, or new language features. If a
feature needs to rewrite user code, a generator is usually the wrong tool.

Good use cases:

- strongly typed route, endpoint, serializer, mapper, or options binding code
- compile-time lookup tables
- enum helpers
- DI registration helpers
- generated reflection metadata for AOT-friendly code
- generated partial method implementations
- generated interop or `UnsafeAccessor` declarations
- generated glue for framework conventions that are otherwise discovered at
  runtime

Bad use cases:

- replacing language syntax with custom dialects
- editing existing method bodies
- injecting logging into existing call sites by rewriting code
- depending on generator execution order
- generating code that depends on another generator's generated code

The official Roslyn design document emphasizes that generators run unordered and
all generators see the same input compilation, not the outputs of other
generators: <https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md>.

## 2. Execution model

Incremental generators run inside compiler hosts such as `csc.exe`,
`VBCSCompiler.exe`, Visual Studio's Roslyn services, or a test generator driver.
They are passed to the compiler as analyzer assets — historically through the
`/analyzer:` command-line switch (and the MSBuild `<Analyzer>` item), with newer
Roslyn versions also supporting a dedicated `/generator:` switch that
distinguishes generators from analyzers. In practice the SDK still emits
generators as `<Analyzer>` items for most projects. The host loads the
assembly, reflects over it, and discovers types decorated with `[Generator]`
that implement `IIncrementalGenerator`.

Build-time loading has several layers:

1. **NuGet restore** resolves generator package assets and records them in the
   project assets file.
2. **MSBuild evaluation** collects `PackageReference`, `ProjectReference`, and
   explicit `<Analyzer>` items.
3. **SDK/targets logic** maps those items into compiler inputs and decides
   which generator DLLs should participate in the build.
4. **Compiler invocation** passes generator paths to the compiler as analyzer
   assets (`/analyzer:` switches, or the dedicated `/generator:` switch where
   the SDK opts in).
5. **Compiler host loading** reflects over those assemblies, discovers
   `IIncrementalGenerator` implementations, builds their pipelines, and runs
   the pipelines when inputs are available.

Visual Studio adds a design-time-build and live-analysis layer. The IDE host and
the CLI host can use different Roslyn versions, especially when a repository
has `global.json`, so always validate important generator behavior in both
Visual Studio and `dotnet build`.

Important execution rules:

- `Initialize` is called once for the generator instance to build the pipeline.
- Pipeline callbacks run later, when the host has input values.
- The generator instance can be reused across projects and compilations.
- Pipeline steps can run concurrently.
- Pipeline steps can run repeatedly during IDE typing.
- Generator output must be deterministic for the same effective inputs.
- No callback should rely on execution order relative to another generator.

For SDK-style projects with `<TargetFrameworks>`, generators execute separately
for each target framework inner build. A project targeting `net8.0;net472`
normally gives the generator two independent compilations. Reference
assemblies, preprocessor symbols, `build_property.TargetFramework`, available
APIs, and generated output can legitimately differ between those runs. Treat
each target framework as a separate compilation boundary and never cache symbols
or parsed configuration globally across TFMs.

Treat the generator instance as immutable. The following is wrong:

```csharp
[Generator(LanguageNames.CSharp)]
internal sealed class BadGenerator : IIncrementalGenerator
{
    private Compilation? _compilation;

    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        _compilation = null;
    }
}
```

The state belongs in the pipeline values:

```csharp
[Generator(LanguageNames.CSharp)]
internal sealed class GoodGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var assemblyName = context.CompilationProvider
            .Select(static (compilation, _) => compilation.AssemblyName ?? string.Empty);

        context.RegisterSourceOutput(
            assemblyName,
            static (productionContext, name) =>
                productionContext.AddSource(
                    "AssemblyName.g.cs",
                    $$"""
                    // <auto-generated/>
                    namespace Generated;

                    internal static partial class AssemblyInfo
                    {
                        public const string Name = "{{name}}";
                    }
                    """));
    }
}
```

## 3. Recommended project configuration

A generator project is usually an SDK-style class library targeting
`netstandard2.0`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>latestMajor</LangVersion>
    <Nullable>enable</Nullable>
    <IsPackable>true</IsPackable>
    <IncludeBuildOutput>false</IncludeBuildOutput>
    <!-- Requires the Microsoft.CodeAnalysis.Analyzers package below; the
         property has no effect on its own. -->
    <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
  </PropertyGroup>

  <ItemGroup>
    <!-- The example pins a recent Roslyn for newest APIs. In real projects,
         pin to the lowest minor that exposes the APIs you actually use
         (e.g. 4.8.* or 4.11.*) to maximize host audience. -->
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.14.*" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.11.*" PrivateAssets="all" />
  </ItemGroup>

  <ItemGroup>
    <None Include="$(OutputPath)$(AssemblyName).dll"
          Pack="true"
          PackagePath="analyzers/dotnet/cs"
          Visible="false" />
  </ItemGroup>
</Project>
```

Guidance:

- Use `PrivateAssets="all"` for Roslyn authoring packages.
- Avoid Workspaces dependencies in the generator assembly unless you are also
  packaging IDE-only code and have validated the host load behavior.
- If the package also ships analyzers, keep generator/analyzer logic in a
  compiler-clean assembly and put code fixes in a separate Workspaces assembly.
- Pin Roslyn to the lowest useful minor. A generator compiled against a newer
  Roslyn cannot load in older hosts.
- If you use APIs such as `ForAttributeWithMetadataName` or
  `AddEmbeddedAttributeDefinition`, document the minimum Roslyn/SDK version.

### Roslyn package version and host compatibility

`Microsoft.CodeAnalysis.CSharp` and related Roslyn packages are compile-time
references for the generator project. At runtime, the host normally loads the
generator against the Roslyn assemblies already present in that compiler or IDE
process. This means the package version you compile against becomes a minimum
host requirement.

Common load-failure diagnostics:

- **`CS9057`** — the generator references a newer compiler version than the
  currently running compiler.
- **`CS8032`** — an instance of a generator/analyzer cannot be created. This is
  a generic load failure and may mean version mismatch, missing dependency, bad
  image, reflection failure, or an exception during activation.

Use wording such as “Requires Roslyn X.Y or later; validated with SDK A.B and
Visual Studio C.D.” Do not assume a NuGet package restore proves the generator
can be loaded by every developer, CI, or IDE host.

Compatibility reference for recent Roslyn versions:

| `Microsoft.CodeAnalysis` minimum version | Minimum .NET SDK | Minimum Visual Studio |
|---|---|---|
| 5.0.0 | 10.0.1xx | VS 2026 18.0 |
| 4.14.0 | 9.0.3xx | VS 2022 17.14 |
| 4.13.0 | 9.0.2xx | VS 2022 17.13 |
| 4.12.0 | 9.0.1xx | VS 2022 17.12 |
| 4.11.0 | 8.0.4xx | VS 2022 17.11 |
| 4.10.0 | 8.0.3xx | VS 2022 17.10 |
| 4.9.2 | 8.0.2xx | VS 2022 17.9 |
| 4.8.0 | 8.0.1xx | VS 2022 17.8 |
| 4.7.0 | 7.0.4xx | VS 2022 17.7 |
| 4.6.0 | 7.0.3xx | VS 2022 17.6 |
| 4.5.0 | 7.0.2xx | VS 2022 17.5 |
| 4.4.0 | 7.0.1xx | VS 2022 17.4 |
| 4.3.1 | 6.0.4xx | VS 2022 17.3 |
| 4.2.0 | 6.0.3xx | VS 2022 17.2 |
| 4.1.0 | 6.0.2xx | VS 2022 17.1 |
| 4.0.1 | 6.0.1xx | VS 2022 RTM (17.0) |
| 3.11.0 | — | VS 2019 16.11 |

Host selection checklist:

1. Check `global.json` for pinned SDK versions.
2. Run `dotnet --info` to see the CLI SDK and MSBuild versions.
3. Check the Visual Studio version and its Roslyn/toolset version.
4. Inspect a binary log (`dotnet build /bl`) for actual analyzer/generator
   assets and targets.
5. Run `dotnet build /p:ReportAnalyzer=true` to verify which generators ran
   and how long they took.
6. Inspect compiler command lines for actual `/analyzer:` paths.

## 4. Minimal generator skeleton

```csharp
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Text;

namespace MyGenerators;

[Generator(LanguageNames.CSharp)]
internal sealed class SampleGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(static postInitializationContext =>
        {
            postInitializationContext.AddSource(
                "SampleGeneratorAttribute.g.cs",
                SourceText.From(
                    """
                    // <auto-generated/>
                    namespace MyGenerators;

                    [System.AttributeUsage(System.AttributeTargets.Class, Inherited = false)]
                    internal sealed class SampleGeneratorAttribute : System.Attribute
                    {
                    }
                    """,
                    Encoding.UTF8));
        });

        var models = context.SyntaxProvider.ForAttributeWithMetadataName(
            fullyQualifiedMetadataName: "MyGenerators.SampleGeneratorAttribute",
            predicate: static (node, _) => node is Microsoft.CodeAnalysis.CSharp.Syntax.TypeDeclarationSyntax,
            transform: static (attributeContext, cancellationToken) =>
            {
                cancellationToken.ThrowIfCancellationRequested();

                var type = (INamedTypeSymbol)attributeContext.TargetSymbol;

                return new TypeToGenerate(
                    NamespaceName: type.ContainingNamespace.IsGlobalNamespace
                        ? string.Empty
                        : type.ContainingNamespace.ToDisplayString(),
                    TypeName: type.Name);
            });

        context.RegisterSourceOutput(
            models,
            static (productionContext, model) =>
            {
                var source = $$"""
                // <auto-generated/>
                namespace {{model.NamespaceName}};

                partial class {{model.TypeName}}
                {
                    public static string GeneratedName => "{{model.TypeName}}";
                }
                """;

                productionContext.AddSource(
                    $"{model.TypeName}.GeneratedName.g.cs",
                    SourceText.From(source, Encoding.UTF8));
            });
    }

    private readonly record struct TypeToGenerate(
        string NamespaceName,
        string TypeName);
}
```

The example intentionally does only three things:

1. emits a marker attribute;
2. discovers attributed types efficiently;
3. converts symbols into a small value-equatable model before generating text.

## 5. Pipeline API essentials

The Roslyn design document explains pipeline primitives in detail:
<https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md>.

### Built-in input providers

Common providers exposed by `IncrementalGeneratorInitializationContext`:

| Provider | Shape | Typical use |
|---|---|---|
| `CompilationProvider` | single value | assembly name, referenced types, compilation options |
| `ParseOptionsProvider` | single value | language version, preprocessor symbols |
| `AnalyzerConfigOptionsProvider` | single value | `.editorconfig`, `.globalconfig`, MSBuild properties and metadata |
| `AdditionalTextsProvider` | many values | non-C# input files |
| `MetadataReferencesProvider` | many values | referenced assemblies and package metadata |
| `SyntaxProvider` | factory for many-value providers | optimized syntax discovery (via `ForAttributeWithMetadataName` / `CreateSyntaxProvider`) |

### `Select`

Use `Select` to map one provider value into another. Keep each transform small
and deterministic.

```csharp
var assemblyName = context.CompilationProvider
    .Select(static (compilation, _) => compilation.AssemblyName ?? string.Empty);
```

### `Where`

Use `Where` to filter multi-value providers as early as possible.

```csharp
var jsonFiles = context.AdditionalTextsProvider
    .Where(static file => file.Path.EndsWith(".json", StringComparison.OrdinalIgnoreCase));
```

### `SelectMany`

Use `SelectMany` when one input produces multiple logical records.

```csharp
var entries = jsonFiles.SelectMany(static (file, cancellationToken) =>
{
    var text = file.GetText(cancellationToken);
    if (text is null)
    {
        return ImmutableArray<Entry>.Empty;
    }

    return ParseEntries(text.ToString());
});
```

Return an explicit `ImmutableArray<T>` (or `IEnumerable<T>`) from both branches
rather than relying on a collection-expression `[]`. The `SelectMany` overload
infers `TResult` from the lambda's return type, and target typing `[]` to an
empty collection can fail or pick an unintended type when the other branch
returns a different concrete sequence.

### `Collect`

Use `Collect` only when a step genuinely needs the full batch. It converts an
`IncrementalValuesProvider<T>` into an `IncrementalValueProvider<ImmutableArray<T>>`.
Overusing it reduces incrementality because one changed item can make the
collected array change.

### `Combine`

Use `Combine` to join providers. Combine late and with reduced values.

Bad pattern:

```csharp
var combined = context.AdditionalTextsProvider.Combine(context.CompilationProvider);
```

This causes every compilation change to be paired with every additional file.
Prefer extracting stable facts first:

```csharp
var assemblyName = context.CompilationProvider
    .Select(static (compilation, _) => compilation.AssemblyName ?? string.Empty);

var combined = context.AdditionalTextsProvider.Combine(assemblyName);
```

### `WithComparer`

Use `WithComparer` when the default equality for a model is not appropriate.
This is especially important for collections, because arrays and many immutable
collections use reference equality by default.

```csharp
var normalizedOptions = context.AnalyzerConfigOptionsProvider
    .Select(static (provider, _) => ReadOptions(provider))
    .WithComparer(GeneratorOptionsComparer.Instance);
```

## 6. Syntax discovery

`SyntaxProvider` exists because syntax scanning can otherwise dominate IDE
performance.

### Prefer `ForAttributeWithMetadataName`

For attribute-driven generators, use
`SyntaxProvider.ForAttributeWithMetadataName`. The Roslyn cookbook and Andrew
Lock's performance article both call this out as the preferred path because it
lets Roslyn index attributes and avoid most syntax work:

- Roslyn cookbook: <https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.cookbook.md>
- Andrew Lock part 9: <https://andrewlock.net/creating-a-source-generator-part-9-avoiding-performance-pitfalls-in-incremental-generators/>

Pattern:

```csharp
var candidates = context.SyntaxProvider.ForAttributeWithMetadataName(
    fullyQualifiedMetadataName: "MyCompany.MyGenerator.GenerateAttribute",
    predicate: static (node, _) => node is Microsoft.CodeAnalysis.CSharp.Syntax.TypeDeclarationSyntax,
    transform: static (attributeContext, cancellationToken) =>
    {
        cancellationToken.ThrowIfCancellationRequested();

        var type = (INamedTypeSymbol)attributeContext.TargetSymbol;
        return TypeModel.FromSymbol(type);
    });
```

### Use `CreateSyntaxProvider` only when attributes are not viable

`CreateSyntaxProvider` is useful when the generator is driven by syntax that
cannot reasonably be marked with an attribute. Keep the predicate purely
syntactic and very cheap; do semantic work only in the transform.

```csharp
var enums = context.SyntaxProvider.CreateSyntaxProvider(
    predicate: static (node, _) => node is Microsoft.CodeAnalysis.CSharp.Syntax.EnumDeclarationSyntax,
    transform: static (syntaxContext, cancellationToken) =>
    {
        cancellationToken.ThrowIfCancellationRequested();

        var symbol = syntaxContext.SemanticModel.GetDeclaredSymbol(
            syntaxContext.Node,
            cancellationToken);

        return symbol is null ? null : EnumModel.FromSymbol(symbol);
    })
    .Where(static model => model is not null);
```

Do not perform broad semantic scans such as “find every type that indirectly
implements this interface.” That forces the generator to inspect too much of the
compilation and cannot be made properly incremental.

## 7. Marker attributes

A marker attribute is the most common way for users to opt into generation.

Design rules:

- Make marker attributes explicit and narrow.
- Use `[AttributeUsage(..., Inherited = false, AllowMultiple = false)]` unless
  multiple applications are part of the contract.
- Prefer sealed attributes.
- Treat attribute constructor parameters and named properties as the public
  configuration contract.
- Provide analyzers for invalid usage where possible.
- Avoid requiring users to manually copy attribute definitions.

### Options for delivering marker attributes

| Approach | Use when | Trade-off |
|---|---|---|
| Generate attribute with `RegisterPostInitializationOutput` | Most generators | Simple and self-contained |
| Ship attributes in a companion runtime/shared DLL | Consumers need to reference the attribute type from multiple assemblies or runtime code | More packaging and versioning complexity |
| Ask users to define the attribute | Rare prototypes only | Easy to get wrong and poor UX |

Andrew Lock covers the marker-attribute trade-offs in parts 7, 8, and 15 of
his series:

- <https://andrewlock.net/creating-a-source-generator-part-7-solving-the-source-generator-marker-attribute-problem-part1/>
- <https://andrewlock.net/creating-a-source-generator-part-8-solving-the-source-generator-marker-attribute-problem-part2/>
- <https://andrewlock.net/exploring-dotnet-10-preview-features-4-solving-the-source-generator-marker-attribute-problem-in-dotnet-10/>

### `EmbeddedAttribute` and duplicate marker attributes

When a generator emits an internal marker attribute into many projects,
`InternalsVisibleTo` or project references can produce duplicate-type warnings
such as CS0436. Newer Roslyn versions provide `AddEmbeddedAttributeDefinition`
and the compiler-recognized `Microsoft.CodeAnalysis.EmbeddedAttribute` pattern
to reduce this problem.

Use this pattern when your minimum Roslyn version supports it.

**Minimum Roslyn:** `AddEmbeddedAttributeDefinition` is available starting in
`Microsoft.CodeAnalysis` **4.12.0** (which ships with the SDK / VS 2022 17.12
band — see the compatibility table in section 3). Earlier hosts do not
recognize the call.

How the two pieces fit together:

1. `AddEmbeddedAttributeDefinition()` injects the compiler-recognized
   `Microsoft.CodeAnalysis.EmbeddedAttribute` type itself into the
   compilation.
2. Marking *your own* generated marker attribute with `[Embedded]` tells the
   compiler this is an embedded, compilation-private attribute, which
   suppresses the duplicate-type warnings (CS0436) when the same generator
   runs in multiple referenced projects.

The order matters: call `AddEmbeddedAttributeDefinition` *before* (or in the
same post-initialization callback as) emitting any source that uses
`[Embedded]`.

```csharp
context.RegisterPostInitializationOutput(static postInitializationContext =>
{
    // 1. Make the EmbeddedAttribute type available in this compilation.
    postInitializationContext.AddEmbeddedAttributeDefinition();

    // 2. Emit your marker attribute and apply [Embedded] to it.
    postInitializationContext.AddSource(
        "GenerateSerializerAttribute.g.cs",
        """
        // <auto-generated/>
        using System;
        using Microsoft.CodeAnalysis;

        namespace MyGenerator;

        [Embedded]
        [AttributeUsage(AttributeTargets.Class, Inherited = false)]
        internal sealed class GenerateSerializerAttribute : Attribute
        {
        }
        """);
});
```

If your supported SDK/Roslyn floor is older than 4.12, use a shared attribute
DLL or manually embed the required attribute support after validating host
compatibility.

## 8. Building cache-friendly models

Incrementality depends on equality. If a pipeline stage produces the same value
as last time, downstream work can be skipped.

Rules:

- Prefer `readonly record struct` (or an immutable `record`) for model values.
  Plain `record struct` is allowed but discouraged because its compiler-generated
  members are mutable, which makes it easier to accidentally violate the
  immutability that incremental caching depends on.
- Use `record struct`, `readonly record struct`, or immutable records for model
  values.
- Extract only the data needed for generation.
- Store symbol display strings, metadata names, enum values, booleans, and small
  option values rather than `ISymbol` objects.
- Do not keep `Compilation`, `SemanticModel`, `ISymbol`, or large syntax nodes
  in model values.
- Avoid arrays and `ImmutableArray<T>` in equality-sensitive model types unless
  you provide a comparer or wrapper.
- Sort values when order is not semantically meaningful.
- Normalize strings that are compared case-insensitively.
- Keep each transform small enough to become a useful cache checkpoint.

Example model:

```csharp
private readonly record struct EnumToGenerate(
    string NamespaceName,
    string TypeName,
    string FullyQualifiedMetadataName,
    EquatableArray<string> MemberNames);
```

`EquatableArray<T>` is **not** a BCL type. It is a small community-pattern
wrapper around `ImmutableArray<T>` that implements value equality (sequence
equality of the underlying elements) so the type is safe to use in pipeline
models without producing false cache-misses caused by reference-equality of
arrays. Reference implementations exist in `CommunityToolkit.Mvvm.SourceGenerators`
and throughout Andrew Lock's series; copy or adapt one of those into your
generator project.

Andrew Lock's parts 9 and 10 are especially useful for this topic:

- Performance pitfalls: <https://andrewlock.net/creating-a-source-generator-part-9-avoiding-performance-pitfalls-in-incremental-generators/>
- Cacheability testing: <https://andrewlock.net/creating-a-source-generator-part-10-testing-your-incremental-generator-pipeline-outputs-are-cacheable/>

## 9. Emitting source

Generated source should be:

- deterministic;
- stable across machines;
- encoded as UTF-8;
- marked with `// <auto-generated/>`;
- nullable-context explicit when needed;
- formatted predictably;
- emitted with unique hint names.

Prefer text generation over syntax tree generation for final output. The Roslyn
cookbook recommends using an indented text writer or `StringBuilder` rather than
constructing `SyntaxNode`s and calling `NormalizeWhitespace`:
<https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.cookbook.md>.

Example:

```csharp
private static string Generate(EnumToGenerate model)
{
    var builder = new StringBuilder();
    builder.AppendLine("// <auto-generated/>");
    builder.AppendLine("#nullable enable");
    builder.AppendLine();

    if (model.NamespaceName.Length > 0)
    {
        builder.Append("namespace ");
        builder.Append(model.NamespaceName);
        builder.AppendLine(";");
        builder.AppendLine();
    }

    builder.Append("partial class ");
    builder.AppendLine(model.TypeName);
    builder.AppendLine("{");
    builder.AppendLine("    public static string Generated() => \"Generated\";");
    builder.AppendLine("}");

    return builder.ToString();
}
```

Hint-name rules:

- Use `.g.cs` suffix.
- Include enough namespace/type context to avoid collisions.
- Replace invalid filename characters.
- Keep hint names deterministic.
- Do not use absolute paths as hint names.

### Marking generated code

Generated source should be identifiable as generated by both tooling and humans.
Two complementary markers exist; they serve different audiences and should
usually be combined.

| Marker | Audience | Effect |
|---|---|---|
| `// <auto-generated/>` header comment | Roslyn, analyzers, IDE | Suppresses most analyzer diagnostics on the file and signals “do not edit” to the IDE. Recognized by the compiler when it appears in the first comment block of the file. |
| `[GeneratedCode(tool, version)]` attribute | Tooling, code-coverage, style checkers, third-party analyzers | Marks individual generated types/members as machine-emitted. Many tools (StyleCop, coverlet, ReSharper, custom analyzers) use this to skip or relax rules. |

Rules:

- Always emit `// <auto-generated/>` as the first line of every generated file.
  This is the primary signal Roslyn uses to suppress analyzer diagnostics on
  generated code. It is documented behavior of
  `GeneratedCodeAnalysisFlags`/`IsGeneratedCode` in Roslyn.
- Apply `[System.CodeDom.Compiler.GeneratedCode("MyGenerator", "1.2.3")]` to
  every top-level type the generator emits (and ideally to generated members
  injected into a user-authored `partial` type). The `tool` argument should be
  a stable identifier — typically the generator's assembly name. The `version`
  argument should be a stable, deterministic string; prefer the generator
  assembly's informational version. Do **not** use `DateTime.Now`,
  `Guid.NewGuid()`, or anything else that would make output non-deterministic
  across builds.
- Do not rely on `[GeneratedCode]` alone for analyzer suppression. Roslyn's
  built-in “is generated” detection primarily inspects the file header
  comment; some analyzers additionally honor `[GeneratedCode]`, but coverage
  is inconsistent.
- For generated members added into a user's `partial` type, place
  `[GeneratedCode]` on the generated member (method, property, nested type),
  not on the user-authored partial declaration.
- Mark generated members the user is not expected to call directly with
  `[System.ComponentModel.EditorBrowsable(EditorBrowsableState.Never)]` to hide
  them from IntelliSense in referencing assemblies.
- Mark internal generated infrastructure with
  `[System.Diagnostics.DebuggerNonUserCode]` when stepping through it during
  debugging would be noise rather than signal.

Example:

```csharp
// <auto-generated/>
#nullable enable

namespace MyApp.Generated;

[global::System.CodeDom.Compiler.GeneratedCode("MyGenerator", "1.2.3")]
[global::System.Diagnostics.DebuggerNonUserCode]
[global::System.ComponentModel.EditorBrowsable(
    global::System.ComponentModel.EditorBrowsableState.Never)]
internal sealed class PersonSerializer
{
    // ...
}
```

Use fully qualified, `global::`-prefixed attribute names in generated code so
the output is robust against `using` aliases, namespace pollution, and
user-defined types that shadow well-known BCL names.

### `file`-scoped generated types

C# 11 introduced the `file` access modifier, which restricts a top-level type
to the single source file in which it is declared. For source generators this
is the cleanest way to emit per-input helper types without leaking names into
the user's assembly or colliding with other generated outputs.

Use `file` for generated types when **all** of the following hold:

- The type is an implementation detail of one generated hint file.
- No other generated file or user code needs to reference it by name.
- The minimum supported language version is C# 11 or later (the generator can
  inspect `ParseOptionsProvider` and degrade gracefully on older targets).

Typical scenarios:

- Per-target helpers emitted once per attributed type (one converter, mapper,
  builder, or visitor per generated file).
- Compile-time lookup tables, switch helpers, or cached
  `ReadOnlySpan<byte>`/`ReadOnlySpan<char>` literals scoped to one file.
- Local glue types that would otherwise need awkward `__GeneratedXyz_<hash>`
  name mangling to avoid collisions across multiple generated files.

Do **not** use `file` when:

- The generated type is part of the public or internal API surface intended
  for the user or other generators to consume.
- Multiple generated files need to share the helper — extract it into a single
  non-`file` generated type instead.
- The consuming project may compile with `<LangVersion>` lower than 11. In
  that case either gate the generator on language version or fall back to a
  unique-name + `internal` strategy.

Example: each attributed type gets its own serializer in its own hint file,
and the helper state machine is `file`-scoped so two generated serializers
in the same assembly cannot collide:

```csharp
// <auto-generated/>
#nullable enable

namespace MyApp.Generated;

[global::System.CodeDom.Compiler.GeneratedCode("MyGenerator", "1.2.3")]
internal static partial class PersonSerializer
{
    public static string Serialize(Person value) => Writer.Write(value);
}

file static class Writer
{
    public static string Write(Person value) => /* ... */;
}
```

Because `Writer` is `file`-scoped, a second generated file can declare its own
`file static class Writer` for a different type without a name clash, and
neither `Writer` is visible from user code.

Language-version guard inside the pipeline:

```csharp
var languageVersion = context.ParseOptionsProvider
    .Select(static (options, _) =>
        ((Microsoft.CodeAnalysis.CSharp.CSharpParseOptions)options).LanguageVersion);
```

Emit `file`-scoped types only when the captured `LanguageVersion` is
`CSharp11` or later; otherwise emit an `internal` type with a deterministic
mangled name.

### Can a generator emit non-code content?

Incremental source generators have exactly one output channel: the
`SourceProductionContext.AddSource(hintName, SourceText)` API (and its
post-initialization counterpart). Every output is added to the **compilation**
as a C# `SyntaxTree`. Generators cannot:

- write arbitrary files to disk;
- add embedded resources to the produced assembly;
- contribute MSBuild items such as `EmbeddedResource`, `Content`, or
  `None`;
- emit `.resx`, `.json`, `.xml`, `.txt`, or binary files that the compiler
  treats as resources;
- run during MSBuild evaluation (they run inside the compiler, after MSBuild
  has already decided which files are resources).

In other words: **the only thing a source generator can emit is C# source**.
This is a deliberate design constraint documented in the Roslyn design notes;
it preserves determinism, IDE incrementality, and the “compiler is a pure
function of its inputs” model.

When you need generated *content* (a real embedded resource, a JSON manifest
on disk, a generated `.cs` file checked in alongside hand-written code), use
one of these alternatives instead of (or in addition to) a source generator:

| Need | Right tool |
|---|---|
| Truly embedded resource in the output assembly | MSBuild target that produces a file under `$(IntermediateOutputPath)` and adds it as `<EmbeddedResource>` before `CoreCompile` |
| Generated file in the project tree, committed or not | MSBuild target (`BeforeTargets="BeforeCompile"`) or a custom `Task` invoked by `.targets` shipped in the NuGet `build/` folder |
| Generated content available at runtime as a file | MSBuild target that emits into the output directory and marks the file `<Content CopyToOutputDirectory="PreserveNewest" />` |
| Compile-time constants, lookup tables, byte arrays, or `ReadOnlySpan<byte>` literals derived from a resource | Source generator that reads the source file via `AdditionalTextsProvider` and emits a C# `static readonly` array or `u8` literal |

The last row is the idiomatic source-generator pattern for resource-shaped
needs: instead of producing a `.bin` resource and reflecting over it at
runtime, read the input through `AdditionalText`, encode it into generated
C# (e.g. a `ReadOnlySpan<byte>` `u8` literal or a `byte[]` initializer), and
let the C# compiler embed it into the assembly. This is both AOT-friendly and
trim-friendly.

Example: turn a JSON manifest declared as `AdditionalFiles` into an embedded
`ReadOnlySpan<byte>`:

```csharp
var manifest = context.AdditionalTextsProvider
    .Where(static f => f.Path.EndsWith("manifest.json", System.StringComparison.OrdinalIgnoreCase))
    .Select(static (f, ct) => f.GetText(ct)?.ToString() ?? string.Empty);

context.RegisterSourceOutput(manifest, static (spc, json) =>
{
    var escaped = System.Text.Encodings.Web.JavaScriptEncoder.Default.Encode(json);

    spc.AddSource("Manifest.g.cs", $$"""
        // <auto-generated/>
        namespace MyApp.Generated;

        internal static class Manifest
        {
            public static System.ReadOnlySpan<byte> Utf8 => "{{escaped}}"u8;
        }
        """);
});
```

If the project genuinely needs a non-`.cs` artifact (e.g. an actual
`.resources` entry, a generated `appsettings.Generated.json` in the output
directory, a generated `.targets` file), implement it as an MSBuild
target/task and ship it via the NuGet `build/` or `buildTransitive/` folder.
Source generators and MSBuild targets compose well: the target produces the
file or `AdditionalFiles` item, and the generator consumes it via
`AdditionalTextsProvider`.

## 10. Diagnostics and analyzers

Generators can report diagnostics from `SourceProductionContext`, but use this
sparingly. Diagnostics are usually better implemented in a companion
`DiagnosticAnalyzer` because analyzers have a richer diagnostic-focused API,
release tracking, code fixes, and clearer separation of concerns.

Use generator diagnostics for:

- invalid generator configuration discovered only during generation;
- malformed `AdditionalFiles` that prevent deterministic output;
- impossible states where generation must stop for a specific input.

Prefer an analyzer for:

- invalid marker attribute usage;
- missing `partial` modifiers;
- unsupported members;
- recommendations and style guidance;
- anything needing a code fix.

If the generator reports diagnostics:

- declare descriptors as static readonly values;
- choose precise source locations;
- include actionable messages;
- never throw from the pipeline to signal user errors;
- test diagnostic IDs, messages, and locations.

## 11. AdditionalFiles

`AdditionalFiles` lets a generator consume non-C# files such as JSON, XML,
YAML, templates, schemas, or domain-specific configuration.

Consumer setup:

```xml
<ItemGroup>
  <AdditionalFiles Include="Schemas\*.json" />
</ItemGroup>
```

Generator pattern:

```csharp
var schemaModels = context.AdditionalTextsProvider
    .Where(static file => file.Path.EndsWith(".json", StringComparison.OrdinalIgnoreCase))
    .Select(static (file, cancellationToken) =>
    {
        var text = file.GetText(cancellationToken);
        if (text is null)
        {
            return SchemaModel.Empty(Path.GetFileName(file.Path));
        }

        return SchemaModel.Parse(Path.GetFileName(file.Path), text.ToString());
    });
```

Rules for generator-side `AdditionalFiles` handling:

- Read files through `AdditionalText.GetText(cancellationToken)`.
- Treat missing text as a recoverable condition.
- Parse into immutable/equatable models as early as possible.
- Do not rely on `AdditionalFiles` enumeration order.
- Define deterministic precedence if multiple files can configure the same
  concept.
- For multitargeted projects, remember the generator runs once per inner build;
  parse configuration per compilation/TFM.
- Match a single well-known file by `Path.GetFileName(...)` when the file may
  move inside the project tree.
- Use path segments or explicit file naming conventions when several files can
  exist for different target frameworks or configurations.
- Prefer `StringComparison.Ordinal` for identifiers and
  `StringComparison.OrdinalIgnoreCase` for file extensions and target framework
  names.
- Keep the parsed result thread-safe. Immutable collections are safer than
  mutable `List<T>` or `HashSet<T>` values captured by output callbacks.

For TFM-specific files, prefer MSBuild filtering when consumers control the
project file:

```xml
<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
  <AdditionalFiles Include="GeneratorConfig\net8.0\schemas.json" />
</ItemGroup>

<ItemGroup Condition="'$(TargetFramework)' == 'net472'">
  <AdditionalFiles Include="GeneratorConfig\net472\schemas.json" />
</ItemGroup>
```

If the generator receives several candidate files, define stable precedence,
for example exact TFM first, then base TFM, then `common`. Do not rely on the
order of `AdditionalTextsProvider` values to resolve ties.

For per-file metadata, combine `AdditionalTextsProvider` with
`AnalyzerConfigOptionsProvider` and read `build_metadata.AdditionalFiles.*`
keys. See the Roslyn cookbook section on consuming MSBuild properties and
metadata:
<https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.cookbook.md>.

Example metadata setup:

```xml
<ItemGroup>
  <CompilerVisibleItemMetadata Include="AdditionalFiles" MetadataName="MyGenerator_Mode" />
  <AdditionalFiles Include="Schemas\person.json" MyGenerator_Mode="Strict" />
</ItemGroup>
```

Example metadata read:

```csharp
var schemaInputs = context.AdditionalTextsProvider
    .Combine(context.AnalyzerConfigOptionsProvider)
    .Select(static (pair, cancellationToken) =>
    {
        var file = pair.Left;
        var optionsProvider = pair.Right;

        optionsProvider.GetOptions(file).TryGetValue(
            "build_metadata.AdditionalFiles.MyGenerator_Mode",
            out var mode);

        var text = file.GetText(cancellationToken);

        return new SchemaInput(
            Path.GetFileName(file.Path),
            text?.ToString() ?? string.Empty,
            mode ?? "Default");
    });
```

## 12. Analyzer config and MSBuild properties

Generators can read:

- `.editorconfig` and `.globalconfig` options;
- globally exposed MSBuild properties;
- metadata attached to items such as `AdditionalFiles`.

Use analyzer config files for human-owned generator policy, MSBuild properties
for project-level build facts, and `AdditionalFiles` for larger structured
inputs. Avoid exposing the same concept through several channels unless the
precedence is documented and tested.

### Global analyzer config option

```csharp
var options = context.AnalyzerConfigOptionsProvider
    .Select(static (provider, _) =>
    {
        provider.GlobalOptions.TryGetValue(
            "my_generator_emit_logging",
            out var rawEmitLogging);

        return new GeneratorOptions(
            EmitLogging: string.Equals(rawEmitLogging, "true", StringComparison.OrdinalIgnoreCase));
    });
```

### MSBuild property through `CompilerVisibleProperty`

Roslyn generators run inside the compiler process and do not have direct access
to MSBuild evaluation. `CompilerVisibleProperty` is the supported bridge from
MSBuild to the compiler. The SDK writes selected properties into a generated
analyzer config input with keys such as `build_property.TargetFramework`.

Package props file:

```xml
<Project>
  <ItemGroup>
    <CompilerVisibleProperty Include="MyGenerator_EnableLogging" />
    <CompilerVisibleProperty Include="TargetFramework" />
  </ItemGroup>
</Project>
```

Generator read:

```csharp
var buildOptions = context.AnalyzerConfigOptionsProvider
    .Select(static (provider, _) =>
    {
        provider.GlobalOptions.TryGetValue(
            "build_property.MyGenerator_EnableLogging",
            out var rawEnableLogging);

        provider.GlobalOptions.TryGetValue(
            "build_property.TargetFramework",
            out var targetFramework);

        return new BuildOptions(
            EnableLogging: string.Equals(rawEnableLogging, "true", StringComparison.OrdinalIgnoreCase),
            TargetFramework: targetFramework ?? string.Empty);
    });
```

Safety rules:

- Treat all option values as untrusted strings.
- Use `TryParse` patterns.
- Prefer safe defaults for missing or malformed values.
- Do not throw from option parsing.
- Document accepted keys and precedence.

The generated analyzer-config representation stores keys in lower case. The
built-in `AnalyzerConfigOptions` implementations happen to perform
case-insensitive lookup today, but the property name component is part of an
undocumented surface and relying on case-insensitivity is fragile across hosts
and testing doubles. **Always query keys in lower case**, e.g.
`build_property.targetframework`, to stay portable.

### `.editorconfig` and `.globalconfig`

Generators see both `.editorconfig` and `.globalconfig` as analyzer config
inputs. Use them for simple user-editable values such as booleans, thresholds,
feature switches, naming options, and severity of companion diagnostics.

| | `.editorconfig` | `.globalconfig` |
|---|---|---|
| Primary purpose | File- and directory-scoped settings | Project- or package-wide settings |
| Scope model | Path sections such as `[*.cs]` or `[src/**/*.cs]` | Global once imported |
| Best for | Tests/generated-code overrides, folder-specific behavior | Organization or repository defaults |
| Path-specific overrides | Yes | No |

Example `.globalconfig` baseline:

```ini
is_global = true

my_generator_emit_logging = false
my_generator_mode = strict
```

Example `.editorconfig` override:

```ini
[**/Tests/*.cs]
my_generator_emit_logging = true
```

Resolution rules to remember:

- within one config file, the later entry for the same key wins;
- between `.editorconfig` files, the deeper file-system path wins;
- between `.globalconfig` files on .NET 6 or later, higher `global_level` wins;
- between `.editorconfig` and `.globalconfig`, `.editorconfig` wins for a
  matching source file;
- command-line and MSBuild warning settings can still override diagnostic
  severity for companion analyzers.

Use `GlobalOptions` for project-wide values and `GetOptions(syntaxTree)` when a
setting is intended to vary by source path. If a generator pipeline needs
per-tree options, extract the option while a syntax node is still available,
then convert it into an equatable model quickly.

### Configuration channel selection

| Need | Preferred channel | Why |
|---|---|---|
| Simple booleans, thresholds, strings, per-folder behavior | `.editorconfig` / `.globalconfig` | Human-editable and IDE/CLI consistent |
| Existing project facts such as TFM, nullable, configuration | `CompilerVisibleProperty` | Avoids duplicating MSBuild facts |
| Per-item metadata for `AdditionalFiles` | `CompilerVisibleItemMetadata` | Keeps metadata attached to the input file |
| Large allow-lists, schemas, templates, many entries | `AdditionalFiles` | Better than encoding structured data in key/value pairs |

Typical precedence when several channels exist:

1. path-scoped analyzer config option;
2. project-wide MSBuild property;
3. per-file metadata or structured `AdditionalFiles` content;
4. built-in generator default.

Andrew Lock part 13 walks through exposing MSBuild properties through generated
editorconfig inputs:
<https://andrewlock.net/creating-a-source-generator-part-13-providing-and-accessing-msbuild-settings-in-source-generators/>.

## 13. Packaging and deployment

Generators are distributed through NuGet analyzer assets: place the generator
DLL under the NuGet `analyzers` folder so the SDK passes it to the compiler as a
build-time extension instead of a runtime reference.

Typical package layout:

```plaintext
analyzers/
  dotnet/
    cs/
      MyGenerator.dll
build/
  MyGenerator.props
```

Project snippet:

```xml
<PropertyGroup>
  <IncludeBuildOutput>false</IncludeBuildOutput>
  <DevelopmentDependency>true</DevelopmentDependency>
</PropertyGroup>

<ItemGroup>
  <None Include="$(OutputPath)$(AssemblyName).dll"
        Pack="true"
        PackagePath="analyzers/dotnet/cs"
        Visible="false" />
  <None Include="build\MyGenerator.props" Pack="true" PackagePath="build" />
</ItemGroup>
```

Packaging rules:

- Do not ship the generator as a normal `lib/` runtime assembly unless it is
  also intentionally a runtime library.
- Use `PrivateAssets="all"` on every Roslyn authoring dependency —
  `Microsoft.CodeAnalysis.CSharp`, `Microsoft.CodeAnalysis.Common` (if
  referenced directly), and `Microsoft.CodeAnalysis.Analyzers`. Otherwise they
  become transitive references for every consumer of the generator package.
- If generation-time dependencies are needed, pack their DLLs next to the
  generator DLL or merge/shade them.
- If generated code needs a runtime dependency, make that dependency visible to
  consumers intentionally.
- Validate the `.nupkg` layout by inspecting it after `dotnet pack`.
- Test package consumption from a separate project, not only through a project
  reference.

### Development-time project references

When testing a generator inside the same solution as a consuming project, add it
as an analyzer asset rather than a normal assembly reference:

```xml
<ItemGroup>
  <ProjectReference Include="..\MyGenerator\MyGenerator.csproj"
                    OutputItemType="Analyzer"
                    ReferenceOutputAssembly="false" />
</ItemGroup>
```

`OutputItemType="Analyzer"` asks the SDK to pass the built DLL to the compiler
as an `<Analyzer>` item. `ReferenceOutputAssembly="false"` prevents the
generator from becoming a runtime dependency of the consuming project.

Manual analyzer items must point to physical DLL paths:

```xml
<ItemGroup>
  <Analyzer Include="path\to\MyGenerator.dll" />
</ItemGroup>
```

If a whole repository should consume a local generator, centralize the reference
in `Directory.Build.props` and condition it so the generator project does not
reference itself.

### Package asset flow

Generator packages are development-time compiler extensions, not normal runtime
libraries.

| Goal | Recommended setting |
|---|---|
| Keep Roslyn authoring packages out of consumers | `PrivateAssets="all"` on Roslyn `PackageReference` items |
| Avoid shipping generator DLL under `lib/` | `<IncludeBuildOutput>false</IncludeBuildOutput>` |
| Mark the package as development-time | `<DevelopmentDependency>true</DevelopmentDependency>` |
| Import props/targets only for direct consumers | Pack into `build/` |
| Import props/targets transitively through intermediate packages | Pack into `buildTransitive/` only when intentional |

Use `buildTransitive/` carefully. A transitive `.targets` file that adds or
removes generators can affect projects that never referenced the generator
package directly.

### External dependencies

Generator-time dependencies are not automatically available to the compiler
host just because the generator project referenced them. If a generator uses a
library while it is running, either avoid the dependency, merge/shade it, or
pack the dependency DLL beside the generator DLL in the same analyzer asset
folder.

Example for a private generation-time dependency:

```xml
<PackageReference Include="Newtonsoft.Json"
                  Version="13.*"
                  PrivateAssets="all"
                  GeneratePathProperty="true" />

<None Include="$(OutputPath)$(AssemblyName).dll"
      Pack="true"
      PackagePath="analyzers/dotnet/cs"
      Visible="false" />

<None Include="$(PkgNewtonsoft_Json)\lib\netstandard2.0\*.dll"
      Pack="true"
      PackagePath="analyzers/dotnet/cs"
      Visible="false" />
```

If the generated source needs a runtime dependency, expose that dependency to
the consumer intentionally as a normal package dependency. Distinguish
generation-time dependencies from generated-code runtime dependencies.

### Combined library + generator packages

If a generator enforces or accelerates correct use of a library, it can be
packed inside the library package:

```plaintext
lib/
  net8.0/
    MyLibrary.dll
analyzers/
  dotnet/
    cs/
      MyLibrary.Generators.dll
build/
  MyLibrary.props
```

Keep the generator in a separate project targeting `netstandard2.0`, reference
it from the library pack project with `ReferenceOutputAssembly="false"`, and
pack its output explicitly into `analyzers/dotnet/cs`. Do not accidentally ship
the generator DLL under `lib/`.

### Language-specific and language-agnostic placement

NuGet uses the path convention `analyzers/{framework}/{language}/{dll}`:

| Path | Loaded for |
|---|---|
| `analyzers/dotnet/cs/MyGenerator.dll` | C# projects |
| `analyzers/dotnet/vb/MyGenerator.dll` | VB projects |
| `analyzers/dotnet/fs/MyGenerator.dll` | F# projects |
| `analyzers/dotnet/MyGenerator.dll` | all analyzer-capable .NET language projects |

Most C# source generators belong under `analyzers/dotnet/cs`. Do not place the
same assembly in both `analyzers/dotnet/` and `analyzers/dotnet/cs/`; a C#
project can load both and produce duplicate output or diagnostics.

Andrew Lock part 3 covers integration testing and NuGet packaging:
<https://andrewlock.net/creating-a-source-generator-part-3-integration-testing-and-packaging/>.

### Supporting multiple Roslyn or SDK versions

If you need both an older Roslyn-compatible generator and a newer optimized
variant, use versioned analyzer folders:

```plaintext
analyzers/
  dotnet/
    roslyn4.0/
      cs/
        MyGenerator.dll
    roslyn4.14/
      cs/
        MyGenerator.dll
```

> The `roslyn{major}.{minor}` folder convention is recognized only by .NET SDK
> 6.0.4xx / NuGet 6.x and later. Older SDKs ignore it entirely and load every
> versioned variant as if all were applicable, which produces duplicate
> generator output. If you must support older SDKs, ship a
> `build/{PackageId}.targets` fallback that manually selects one assembly (see
> the analyzer guide's multi-Roslyn section for a worked example).

Rules:

- Keep `AssemblyName` stable across bands.
- Keep diagnostic IDs and generated public contracts stable.
- Build/test every band independently.
- Do not place the same generator in both unversioned and versioned analyzer
  folders unless additive double-loading is intentional.
- For old SDKs that do not understand `roslyn{version}` folders, ship a
  `build/{PackageId}.targets` fallback only after validating it.

Andrew Lock part 14 discusses splitting projects and tests for multiple SDK
versions:
<https://andrewlock.net/creating-a-source-generator-part-14-supporting-multiple-sdk-versions-in-a-source-generator/>.

## 14. Testing generators

Test at four levels:

1. **Pure model tests**: parse symbols/options into model values and validate
   equality.
2. **Generator output tests**: run the generator against source and verify
   generated files.
3. **Diagnostic tests**: verify user-facing diagnostics.
4. **Integration/package tests**: consume the packed NuGet package in a real
   project.

Recommended packages:

- `Microsoft.CodeAnalysis.CSharp.SourceGenerators.Testing`
- `Microsoft.CodeAnalysis.Testing`
- a test framework such as xUnit, NUnit, or MSTest
- snapshot testing library if your team accepts snapshot baselines

Example with driver-level testing:

```csharp
var syntaxTree = CSharpSyntaxTree.ParseText(
    """
    using MyGenerator;

    [GenerateSerializer]
    public partial class Person
    {
        public string Name { get; set; } = "";
    }
    """);

// Prefer explicit references over scanning the test host's loaded assemblies.
// AppDomain.CurrentDomain.GetAssemblies() is brittle: the set depends on what
// the host has loaded so far, which makes tests order-sensitive and breaks on
// trimmed/single-file hosts. For most analyzer/generator tests, the cleaner
// option is the Basic.Reference.Assemblies NuGet package, which exposes ready
// reference sets such as ReferenceAssemblies.Net80.
var references = new[]
{
    MetadataReference.CreateFromFile(typeof(object).Assembly.Location),
    MetadataReference.CreateFromFile(typeof(System.Linq.Enumerable).Assembly.Location),
    MetadataReference.CreateFromFile(typeof(System.Collections.Generic.List<>).Assembly.Location),
    // ...add any framework or user assemblies your generator needs.
};

var compilation = CSharpCompilation.Create(
    "Tests",
    [syntaxTree],
    references,
    new CSharpCompilationOptions(OutputKind.DynamicallyLinkedLibrary));

var generator = new MyGenerator();
var driver = CSharpGeneratorDriver.Create(generator);

var runDriver = driver.RunGenerators(compilation);
var result = runDriver.GetRunResult();
```

Testing guidance:

- Assert generated source file names.
- Assert generated source content or snapshots.
- Assert the final compilation succeeds.
- Test nullable enabled/disabled if output depends on nullable context.
- Test different language versions if using new syntax.
- Test valid and invalid analyzer config values.
- Test `AdditionalFiles` with in-memory `AdditionalText` implementations.
- Add cacheability tests using generator run results and tracking names where
  possible.

Andrew Lock part 2 covers snapshot testing; part 10 covers cacheability tests:

- <https://andrewlock.net/creating-a-source-generator-part-2-testing-an-incremental-generator-with-snapshot-testing/>
- <https://andrewlock.net/creating-a-source-generator-part-10-testing-your-incremental-generator-pipeline-outputs-are-cacheable/>

## 15. Performance checklist

Generator performance matters most in the IDE because pipelines may run after
small edits.

Checklist:

- Prefer `ForAttributeWithMetadataName` over broad syntax scanning.
- Keep predicates syntactic and cheap.
- Extract symbols into small equatable models early.
- Do not put `ISymbol`, `Compilation`, `SemanticModel`, `SyntaxNode`, or
  `Location` in long-lived model values.
- Avoid reflection in pipeline callbacks.
- Avoid file/network IO except through `AdditionalText`.
- Avoid `Collect` until aggregation is necessary.
- Combine with reduced values, not whole compilations.
- Use deterministic ordering.
- Provide custom equality for collection-heavy models.
- Pass cancellation tokens to Roslyn APIs.
- Use `RegisterImplementationSourceOutput` for implementation-only output when
  semantically appropriate.
- Keep diagnostics out of generators when a companion analyzer is better.
- Measure with large real projects and `/p:ReportAnalyzer=true` when possible.

The Roslyn design document explains caching and equality requirements:
<https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md>.
Andrew Lock part 9 gives practical performance pitfalls:
<https://andrewlock.net/creating-a-source-generator-part-9-avoiding-performance-pitfalls-in-incremental-generators/>.

## 16. AOT, trimming, and runtime-code replacement

Source generators are especially useful for Native AOT and trimming because they
move work from runtime discovery to compile time.

Runtime patterns that are often problematic for trimming/AOT:

- reflection-based member discovery;
- dynamic method generation;
- expression compilation to IL;
- late-bound serialization metadata;
- convention-based DI scanning;
- runtime proxy generation;
- unbounded `Assembly.GetTypes()` scans;
- access to non-public members through reflection.

Generator-friendly replacements:

| Runtime pattern | Generator replacement |
|---|---|
| Reflect over properties for serialization | Generate property readers/writers at compile time |
| Scan assemblies for handlers | Generate a registration table from marked types |
| Build expression delegates at startup | Generate strongly typed methods |
| Use reflection to access private members in trusted scenarios | Generate `UnsafeAccessor` declarations where appropriate |
| Discover route metadata at runtime | Generate endpoint metadata and strongly typed binders |
| Build enum maps lazily | Generate switch expressions or lookup tables |

AOT-oriented generator design rules:

- Generate direct calls instead of reflection when possible.
- Avoid emitting code that itself depends on reflection unless annotated for
  trimming.
- Prefer compile-time metadata tables over runtime scanning.
- Keep generated code linker-friendly and explicit.
- Respect accessibility; only use advanced access mechanisms when the owning
  API intentionally permits it.
- Validate generated code under `PublishAot=true` and trimming warnings enabled.
- Document any runtime features required by the generated output.

Example generated lookup shape:

```csharp
// <auto-generated/>
#nullable enable

namespace MyApp.Generated;

internal static partial class KnownHandlers
{
    public static void Register(IServiceCollection services)
    {
        services.AddSingleton<IHandler, CreateOrderHandler>();
        services.AddSingleton<IHandler, CancelOrderHandler>();
    }
}
```

The value is not just faster startup. The generated code tells the linker and
AOT compiler exactly which members are used.

## 17. Interceptors, UnsafeAccessor, and advanced runtime features

### `UnsafeAccessor`

`UnsafeAccessorAttribute` can expose strongly typed access to otherwise
inaccessible members without reflection in tightly controlled scenarios. A
source generator can produce the required `extern static` declarations from a
known model.

Example shape:

```csharp
// <auto-generated/>
using System.Runtime.CompilerServices;

namespace MyApp.Generated;

internal static partial class PersonAccessors
{
    // For UnsafeAccessorKind.Method, the first parameter is the instance and
    // its type must exactly match the declaring type of the target member.
    // The remaining parameters and the return type must match the target
    // method's signature, and the `Name` value must exactly match the target
    // method's metadata name.
    [UnsafeAccessor(UnsafeAccessorKind.Method, Name = "NormalizeName")]
    public static extern string NormalizeName(Person person);
}
```

`UnsafeAccessorAttribute` is available starting in **.NET 8**
(`System.Runtime.CompilerServices.UnsafeAccessorAttribute`). Guard generated
code with `#if NET8_0_OR_GREATER` (or an explicit target-framework filter in
the pipeline) when the generator may also run for earlier TFMs.

Use this carefully:

- It is unsafe by design and can break if member names or signatures change.
- It should not be used to bypass normal encapsulation casually.
- It is most appropriate for framework/runtime/library scenarios where the
  target member is intentionally part of a generated contract.
- Tests should compile and execute the generated accessor against the exact
  target framework that supports the API.
- Generated code should be conditional on target framework or feature
  availability.

### Interceptors

Interceptors are an advanced source-generator-related feature for redirecting
specific call sites. Andrew Lock part 11 demonstrates implementing one:
<https://andrewlock.net/creating-a-source-generator-part-11-implementing-an-interceptor-with-a-source-generator/>.

Treat interceptors as specialized and version-sensitive:

- require explicit language/compiler support;
- guard generated code by language version and feature availability;
- document the exact SDK/Roslyn requirements;
- keep a non-interceptor fallback when possible;
- validate in a real consuming project.

Do not present interceptors as a general replacement for generator output. Most
generators should remain additive and generate normal source.

## 18. Debugging and generated output

### Debugging

The most reliable debugging workflow is a focused unit test that runs the
generator driver. Debug the test rather than attaching to `devenv.exe` or
compiler server processes.

Other options:

- Use a VSIX experimental instance when developing IDE-specific integration.
- Temporarily call `Debugger.Launch()` in a local branch only.
- Use `dotnet build /p:ReportAnalyzer=true` to see generator/analyzer execution
  timing.
- Use binary logs to inspect analyzer/generator assets passed to the compiler.

### Saving generated files

Consumers can ask the compiler to emit generated files to disk:

```xml
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
  <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>
```

When `EmitCompilerGeneratedFiles` is `true`, the compiler writes every
generated file under
`$(CompilerGeneratedFilesOutputPath)/<GeneratorAssemblyName>/<GeneratorTypeName>/<hintName>`.
If `CompilerGeneratedFilesOutputPath` is not set, files are emitted under
`$(IntermediateOutputPath)generated/` (i.e. inside `obj/`).

Guidance:

- This is useful for debugging and code review.
- Generated files are not normally artifacts and are not automatically checked
  into source control.
- In multitargeted projects, the compiler runs once per inner build, so
  generated files are written under a per-TFM subdirectory. Account for this
  when committing or diffing them.

#### Checking generated files into the repository

Committing generated `.cs` files is a legitimate workflow for code review,
diff-based audits, AOT validation, and reproducible-build investigations. To
do it safely, the files must be emitted into the repo **and excluded from
compilation**, otherwise every type will be defined twice (once by the
generator at compile time and once from the committed `.cs` file).

Recommended setup in the consuming project:

```xml
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
  <!-- Emit into a path under the project, not under obj/. -->
  <CompilerGeneratedFilesOutputPath>$(MSBuildProjectDirectory)\Generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>

<ItemGroup>
  <!-- Remove emitted files from the default compile glob so they are not
       compiled a second time alongside the in-memory generator output. -->
  <Compile Remove="Generated/**/*.cs" />
  <!-- Keep them visible in the IDE/Solution Explorer for review. -->
  <None Include="Generated/**/*.cs" />
</ItemGroup>
```

Rules and trade-offs:

- The path under `CompilerGeneratedFilesOutputPath` must be inside the
  project (or another tracked directory) for Git to see it. Keep it stable
  across machines — do not include `$(Configuration)` or absolute paths in
  the property.
- Always pair `EmitCompilerGeneratedFiles=true` with a `<Compile Remove="…"/>`
  for the same path. Without it you will see `CS0101`/`CS0111`
  duplicate-member errors.
- Generated files inherit hint names, not project-relative paths. Two
  generators that emit the same hint name will collide on disk; choose hint
  names that include the generator's namespace prefix to avoid this.
- For multitargeted projects, generated files are written under
  `Generated/<Tfm>/<GeneratorAssembly>/<GeneratorType>/...`. Either commit
  all TFM variants, or restrict the emit to one canonical TFM:
  ```xml
  <PropertyGroup Condition="'$(TargetFramework)' == 'net8.0'">
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
  </PropertyGroup>
  ```
- Treat committed generated files as **read-only artifacts**. Do not
  hand-edit them. The `// <auto-generated/>` header already tells Roslyn,
  analyzers, and most IDEs to suppress style/quality diagnostics on the
  file.
- For CI, add a job that runs `dotnet build` and then fails if `git diff
  --exit-code Generated/` reports drift. This guarantees that the committed
  files actually match the generator output for the current sources.
- Do not check generated files into a NuGet package's `lib/` output. The
  compiler-in-memory generator output is what ends up in the produced
  assembly; the committed files exist only for review.

Andrew Lock part 6 covers saving generated output and avoiding duplicate
compilation:
<https://andrewlock.net/creating-a-source-generator-part-6-saving-source-generator-output-in-source-control/>.

## 19. Troubleshooting

### Generator not running

Check:

- Is the generator DLL passed as an analyzer asset?
- Is it under `analyzers/dotnet/cs` or the correct `roslyn{version}` folder?
- Does the host Roslyn version meet the generator's referenced Roslyn version?
- Is the generator type decorated with `[Generator]`?
- Does the generator assembly target a loadable TFM such as `netstandard2.0`?
- Are generation-time dependencies packed beside the generator?
- Does `global.json` pin an older SDK than expected?

### Generator runs but emits nothing

Check:

- Is the marker attribute metadata name correct?
- Is the target syntax kind accepted by the predicate?
- Is the marker attribute generated before syntax discovery through
  `RegisterPostInitializationOutput`?
- Are semantic transforms returning `null` and being filtered out?
- Are diagnostics hidden because generation errors were swallowed? In
  particular, check the build output for **`CS8785`** — the canonical signal
  that a generator threw an exception. Roslyn catches the exception so the
  build does not crash, but the generator's source output is discarded for
  that compilation.
- Are `AdditionalFiles` included in the consuming project?

### IDE behavior differs from `dotnet build`

Check:

- Visual Studio version and Roslyn version.
- CLI SDK selected by `global.json`.
- `dotnet --info`.
- Build binary log.
- `/p:ReportAnalyzer=true` output.
- `.editorconfig`, `.globalconfig`, `<NoWarn>`, and `<WarningsAsErrors>`.
- Target framework inner builds in multitargeted projects.

### Generated code fails only under AOT/trimming

Check:

- Does generated code use reflection or dynamic code?
- Are trimming warnings enabled and treated seriously?
- Are all accessed members statically visible to the linker?
- Is the generated output target-framework-specific?
- Does the generated code use APIs unavailable in the target TFM?
- Are `UnsafeAccessor` declarations guarded for supported frameworks?

## 20. Source map to Andrew Lock's series

The Andrew Lock series is practical and example-driven. Use it as follows:

| Part | Article | Use it for |
|---|---|---|
| 1 | <https://andrewlock.net/creating-a-source-generator-part-1-creating-an-incremental-source-generator/> | first generator, enum model, pipeline stages |
| 2 | <https://andrewlock.net/creating-a-source-generator-part-2-testing-an-incremental-generator-with-snapshot-testing/> | snapshot testing generated output and diagnostics |
| 3 | <https://andrewlock.net/creating-a-source-generator-part-3-integration-testing-and-packaging/> | integration testing and NuGet packaging |
| 4 | <https://andrewlock.net/creating-a-source-generator-part-4-customising-generated-code-with-marker-attributes/> | marker attribute parameters and generated code customization |
| 5 | <https://andrewlock.net/creating-a-source-generator-part-5-finding-a-type-declarations-namespace-and-type-hierarchy/> | namespaces, nested types, type hierarchy modeling |
| 6 | <https://andrewlock.net/creating-a-source-generator-part-6-saving-source-generator-output-in-source-control/> | emitted generated files and source control decisions |
| 7 | <https://andrewlock.net/creating-a-source-generator-part-7-solving-the-source-generator-marker-attribute-problem-part1/> | marker attribute delivery trade-offs |
| 8 | <https://andrewlock.net/creating-a-source-generator-part-8-solving-the-source-generator-marker-attribute-problem-part2/> | external/shared marker attribute DLL options |
| 9 | <https://andrewlock.net/creating-a-source-generator-part-9-avoiding-performance-pitfalls-in-incremental-generators/> | performance pitfalls and cache-friendly design |
| 10 | <https://andrewlock.net/creating-a-source-generator-part-10-testing-your-incremental-generator-pipeline-outputs-are-cacheable/> | testing that pipeline outputs are cacheable |
| 11 | <https://andrewlock.net/creating-a-source-generator-part-11-implementing-an-interceptor-with-a-source-generator/> | interceptors and call-site redirection experiments |
| 12 | <https://andrewlock.net/creating-a-source-generator-part-12-reading-compilation-options-and-csharp-version-in-source-generators/> | compilation options and language version checks |
| 13 | <https://andrewlock.net/creating-a-source-generator-part-13-providing-and-accessing-msbuild-settings-in-source-generators/> | MSBuild properties and generator configuration |
| 14 | <https://andrewlock.net/creating-a-source-generator-part-14-supporting-multiple-sdk-versions-in-a-source-generator/> | multiple SDK/Roslyn version support |
| 15 | <https://andrewlock.net/exploring-dotnet-10-preview-features-4-solving-the-source-generator-marker-attribute-problem-in-dotnet-10/> | `EmbeddedAttribute` and .NET 10 marker attribute improvements |

## 21. Pre-publish checklist

Before publishing a generator package:

1. Generator targets `netstandard2.0` or a documented supported host TFM.
2. Roslyn package references are pinned to the lowest required minor.
3. Generator uses `IIncrementalGenerator`.
4. No per-compilation mutable state is stored in fields.
5. Syntax discovery uses `ForAttributeWithMetadataName` when possible.
6. Pipeline model values are equatable and do not retain symbols or
   compilations.
7. Generated source has stable hint names and `// <auto-generated/>`.
8. Marker attributes are generated or packaged deliberately.
9. Configuration keys and precedence are documented.
10. Additional files and MSBuild properties are tested.
11. Generated output compiles under all supported TFMs and language versions.
12. AOT/trimming scenarios are tested if the generator claims AOT support.
13. Package layout contains only intended analyzer/generator assets.
14. Generation-time dependencies are packed beside the generator or avoided.
15. The package is tested from a separate consuming project.
16. Visual Studio live analysis and `dotnet build` both load the generator.
17. Performance is checked on a non-trivial project.
18. Multi-Roslyn bands, if present, are all built and tested independently.

## 22. Key takeaways

- Use `IIncrementalGenerator` for all new generator work.
- Think in pipelines, not in one big execute method.
- Make user intent explicit with marker attributes.
- Prefer `ForAttributeWithMetadataName` for syntax-driven generators.
- Extract small equatable models early.
- Never carry symbols, compilations, or semantic models deeper than necessary.
- Generate deterministic text with stable hint names.
- Put validation diagnostics in a companion analyzer when possible.
- Treat configuration as part of the generator's public contract.
- Package generators under NuGet analyzer asset folders and validate host
  compatibility.
- Source generators are a major AOT enabler because they replace runtime
  reflection and dynamic code with compile-time generated source.
- Use advanced features such as `UnsafeAccessor` and interceptors only with
  explicit target-framework/language-version guards and strong tests.
