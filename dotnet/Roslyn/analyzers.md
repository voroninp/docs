Unfortunately, Microsoft does not care much about providing actualized and extensive documentation about the nitty-gritty 
of the Roslyn analyzers development.
One must search in blog posts (usually by [Andrew Lock](https://andrewlock.net/series/creating-a-source-generator/) or [Gérald Barré](https://www.meziantou.net/writing-a-roslyn-analyzer.htm)), 
or in GitHub Issues and Discussions, or even MRs.

I needed to develop several analyzers for our internal packages, so pulled as much information as I could find and asked several LLMs 
for building and reviewing a detailed guide about desiging analyzers. Here's the outcome of this vibedocumenting. Obviously, use with care.

# Roslyn Analyzer Development Guide

## Table of contents

- [Quick start checklist](#quick-start-checklist)
- [1. Roslyn analyzer fundamentals](#1-roslyn-analyzer-fundamentals)
- [2. Analyzer execution model (critical)](#2-analyzer-execution-model-critical)
- [3. Recommended project configuration](#3-recommended-project-configuration)
- [4. Roslyn package version and host compatibility](#4-roslyn-package-version-and-host-compatibility)
- [5. Creating a new analyzer](#5-creating-a-new-analyzer)
- [6. Roslyn API selection guidance](#6-roslyn-api-selection-guidance)
- [7. Semantic analysis](#7-semantic-analysis)
- [8. Performance and reliability checklist](#8-performance-and-reliability-checklist)
- [9. Diagnostic design](#9-diagnostic-design)
- [10. CodeFixProvider](#10-codefixprovider)
- [11. Incremental analysis](#11-incremental-analysis)
- [12. Working with AdditionalFiles](#12-working-with-additionalfiles)
- [13. Reading MSBuild properties](#13-reading-msbuild-properties)
- [14. Packaging analyzers](#14-packaging-analyzers)
- [15. Deploying analyzers](#15-deploying-analyzers)
- [16. Versioning and rollout strategy](#16-versioning-and-rollout-strategy)
- [17. Release tracking](#17-release-tracking)
- [18. Troubleshooting](#18-troubleshooting)
- [19. Security and governance](#19-security-and-governance)
- [20. Validating analyzer quality with Microsoft.CodeAnalysis.Analyzers](#20-validating-analyzer-quality-with-microsoftcodeanalysisanalyzers)
- [21. Testing Analyzers](#21-testing-analyzers)
- [22. Debugging Analyzers](#22-debugging-analyzers)
- [23. Pre-publish checklist](#23-pre-publish-checklist)
- [24. Key takeaways](#24-key-takeaways)
- [25. Dataflow analysis framework](#25-dataflow-analysis-framework)
- [26. Porting Legacy Rules (FxCop/Binary)](#26-porting-legacy-rules-fxcopbinary)
- [27. Consumer-side suppression](#27-consumer-side-suppression)

This document covers general Roslyn analyzer development practices for
`DiagnosticAnalyzer`, `DiagnosticSuppressor`, and `CodeFixProvider` projects.
Source generators share many host-loading and packaging constraints, but they
have additional design concerns and are mentioned here only where the guidance
overlaps.

Analyzers target `netstandard2.0` so they can run inside different host
environments (Visual Studio, `dotnet build`, CI, and NuGet consumer projects).
The `netstandard2.0` constraint exists because some compiler hosts still run on
.NET Framework; it can be relaxed only after every supported host moves to a
newer runtime.

## Quick start checklist

When creating a new analyzer or code fix, verify the following first:

- Target `netstandard2.0`. 
- Pin Roslyn packages to the lowest minor version that exposes the APIs you
  need. The Roslyn package version defines the minimum host version that can
  load the analyzer. Pinning too high excludes older Visual Studio or SDK hosts
  for no functional benefit and may surface as analyzer load failures.
- Call `ConfigureGeneratedCodeAnalysis(...)` and `EnableConcurrentExecution()`
  in `Initialize`. These are core analyzer hygiene requirements. The first makes
  generated-code behavior explicit, and the second allows Roslyn to run
  callbacks concurrently for better IDE and build performance.
- Prefer `IOperation` or symbol analysis over syntax-only analysis when
  possible. Semantic and operation APIs are usually more robust than syntax
  shape matching. They reduce false positives caused by equivalent syntax forms
  and make the rule easier to maintain.
- Cache expensive state in `RegisterCompilationStartAction`. Compilation-wide
  lookups should be done once, not repeated for every syntax node or operation.
  This improves performance and avoids unnecessary allocations in hot paths.
- Use precise, localizable diagnostic descriptors with stable IDs. Diagnostic
  IDs and messages are part of the public contract of the analyzer. Stable IDs
  protect consumer configuration, while localization and precise wording improve
  usability in the IDE.
- Add release tracking entries in `AnalyzerReleases.Unshipped.md`. This keeps
  the package history synchronized with `SupportedDiagnostics` and helps the
  Roslyn analyzer authoring rules validate your release metadata before
  shipping. Use `Unshipped` during active development because the rule is not
  part of a published package yet. Move those entries to
  `AnalyzerReleases.Shipped.md` only when you actually publish the package
  version that contains them.
- Add positive, negative, and boundary tests before shipping the rule. Analyzer
  bugs usually appear as false positives or missed diagnostics. A minimal test
  matrix is the best protection against noisy rules that users quickly disable.
- Verify whether the rule needs consumer configuration through `.editorconfig`,
  `AdditionalFiles`, MSBuild properties, or a combination of them. Configuration
  is part of the design, not an afterthought. Choosing the right configuration
  channel early keeps the rule understandable, testable, and consistent between
  IDE and CLI hosts.

## 1. Roslyn analyzer fundamentals

A Roslyn analyzer is a .NET assembly that inspects source code using compiler
APIs and reports diagnostics.

Core concepts:

- **`DiagnosticAnalyzer`**: entry point type.
- **`DiagnosticDescriptor`**: metadata for one rule (ID, title, message,
  severity, category).
- **`AnalysisContext`**: registration surface used in `Initialize`.
- **`OperationAnalysisContext` / `SyntaxNodeAnalysisContext`**: callback
  context during analysis.
- **`IOperation` tree**: semantic model abstraction over syntax.
- **`SyntaxNode` tree**: exact parsed source representation.
- **`ISymbol`**: representation of declared entities.

**Diagnostic Suppressors:** A specialized type of analyzer,
`DiagnosticSuppressor`, does not report new warnings but instead
programmatically silences existing compiler or analyzer warnings (e.g., CS8618)
when the code is actually safe according to domain-specific logic. They use
`SuppressionDescriptor` and override `ReportSuppressions`.

### Syntax vs Semantic vs Operation

| Layer | Meaning |
|-------|---------|
| Syntax | "What the user wrote" |
| Semantic | "What it means" |
| Operation | "Normalized semantic structure" |

**Recommendation:** Prefer `IOperation` for most rules.

### Required overrides on `DiagnosticAnalyzer`

A minimal `DiagnosticAnalyzer` subclass must override exactly two members:

- `SupportedDiagnostics` — an `ImmutableArray<DiagnosticDescriptor>` listing
  every descriptor the analyzer can report. The compiler reads this property
  *before* `Initialize` runs to decide whether the analyzer is needed at all,
  so it must be cheap and return a stable, allocation-free value (typically a
  static readonly field).
- `Initialize(AnalysisContext context)` — the one-time registration entry
  point. Inside `Initialize` you call `ConfigureGeneratedCodeAnalysis`,
  `EnableConcurrentExecution`, and the `Register*Action` methods. Do not
  perform real analysis here; only register callbacks. `Initialize` runs once
  per analyzer instance, not once per compilation.

Reporting a diagnostic whose ID is not listed in `SupportedDiagnostics` is a
contract violation and is caught by [RS1005](https://github.com/dotnet/roslyn-analyzers/blob/main/src/Microsoft.CodeAnalysis.Analyzers/Microsoft.CodeAnalysis.Analyzers.md#rs1005-reportdiagnostic-invoked-with-an-unsupported-diagnosticdescriptor).

### Typical analyzer lifecycle

1. Declare one or more `DiagnosticDescriptor` instances.
2. Register callbacks in `Initialize` (operation/syntax/symbol/compilation
   actions).
3. Filter aggressively in callbacks for performance.
4. Create diagnostics with `Diagnostic.Create(...)` at precise locations.

## 2. Analyzer execution model (critical)

Analyzers:

- Run **inside compiler processes**
- Are loaded via reflection looking for the `[DiagnosticAnalyzer]` attribute
  (Code fixes explicitly use [MEF](https://learn.microsoft.com/en-us/dotnet/standard/mef/) via `[ExportCodeFixProvider]`)
- Must not hold **mutable instance or static state**. The analyzer *type* is
  instantiated once and reused across compilations, so per-compilation state
  must live inside `CompilationStartAction` (or `SymbolStartAction` /
  `OperationBlockStartAction`) closures, never in fields. This is what RS1008
  enforces.
- May execute **concurrently** after `EnableConcurrentExecution()` is called
- May run **multiple times per compilation**

### How hosts load analyzers (Attribute vs [MEF](https://learn.microsoft.com/en-us/dotnet/standard/mef/))

1. **`DiagnosticAnalyzer` (and Suppressors)**: The compiler host (`csc.exe`,
   `VBCSCompiler.exe`, or Visual Studio's Roslyn services) discovers analyzers by
   reflecting over assemblies (passed via `/analyzer:Analyzer.dll` command-line
   switches) and looking for types decorated with `[DiagnosticAnalyzer(...)]`
   that inherit from `DiagnosticAnalyzer` or `DiagnosticSuppressor`.
   **Analyzers do NOT use MEF (`[Export]`).**
2. **`CodeFixProvider`**: Code fixes run in the IDE/Workspaces layer to provide
   refactoring actions. They **ARE** loaded via [MEF](https://learn.microsoft.com/en-us/dotnet/standard/mef/). Decorate the provider with
   `[ExportCodeFixProvider(LanguageNames.CSharp, Name = nameof(MyCodeFix))]`
   and normally also `[Shared]`. Keep MEF exports in the code-fix or
   refactoring assembly, not in the compiler-only analyzer assembly.

### Rules

- Do NOT use static mutable state
- Do NOT assume execution order
- Always assume multi-threading

### Correct pattern

Use `CompilationStartAction` to build per-compilation state once and share it
across callbacks:

```csharp
context.RegisterCompilationStartAction(startContext =>
{
    var cachedData = BuildCache(startContext.Compilation);

    startContext.RegisterOperationAction(
        ctx => Analyze(ctx, cachedData),
        OperationKind.Invocation);
});
```

### Build and host pipeline

It helps to separate the responsibilities of the build layers involved in
analyzer loading:

1. **NuGet restore** resolves analyzer package assets and records them in the
   project assets file.
2. **MSBuild evaluation** collects relevant items such as `PackageReference`,
   `ProjectReference`, and explicit `<Analyzer>` items.
3. **SDK/targets logic** maps those items into compiler inputs and decides
   which analyzer DLLs should participate in the build.
4. **Compiler invocation** passes analyzer paths to `csc.exe` or `vbc.exe`
   using `/analyzer:` command-line switches.
5. **Compiler host loading** reflects over the assemblies, discovers
   `DiagnosticAnalyzer` and `DiagnosticSuppressor` types, and executes them
   during compilation.

In the IDE there is an additional layer: Visual Studio typically runs analyzers
through design-time builds and live analysis services, so the same project may
be analyzed by both the command-line SDK and the IDE host. When those hosts use
different Roslyn versions, `dotnet build` and live analysis may disagree even if
the project file is unchanged.

Practical consequence: when diagnosing analyzer activation issues, check **all**
of the following rather than assuming one host represents the other:

- `global.json`
- `dotnet --info`
- the Visual Studio version in use
- the actual `/analyzer:` inputs passed to the compiler

### Multitargeted consumer projects (`TargetFrameworks`)

For SDK-style projects that use `<TargetFrameworks>...</TargetFrameworks>`, 
analyzers execute separately for each target framework during build.

Why:

- .NET SDK multitargeting is implemented as one **outer build** plus multiple
  **inner builds**.
- The outer build works with `TargetFrameworks`, then dispatches one inner
  build per target.
- Each inner build sets a single `TargetFramework` value and performs its own
  compiler invocation.
- Because analyzers run inside the compiler for a specific compilation, the
  analyzer runs once per inner build / per TFM, not once for the multitargeted
  project as a whole.

That means a consumer project like this:

```xml
<PropertyGroup>
  <TargetFrameworks>net8.0;net472</TargetFrameworks>
</PropertyGroup>
```

normally produces **two independent analyzer executions** at build time: one
against the `net8.0` compilation and one against the `net472` compilation.

### What changes between TFMs

The analyzer assembly is the same, but the **compilation input** is different
per TFM. In practice, any of the following may change between runs:

- reference assemblies and available APIs
- resolved symbols from `GetTypeByMetadataName(...)`
- conditional package and project references guarded by
  `Condition="'$(TargetFramework)' == '...'"`
- preprocessor symbols such as `NET8_0`, `NET472`, `WINDOWS`, etc.
- analyzer-config values derived from `build_property.TargetFramework`
- platform-specific TFMs such as `net8.0-windows` vs `net8.0`

So the same rule may legitimately:

- report only for one TFM
- report at different locations or with different messages per TFM
- resolve a well-known type in one TFM and fail to resolve it in another

This is not a special Roslyn exception; it follows directly from the fact that
each TFM is a distinct compilation.

### Design implications for analyzer authors

- Treat every TFM as a separate compilation boundary.
- Keep all cached state inside `CompilationStartAction` closures. That cache is
  valid only for the current inner build.
- Do **not** try to reuse symbol or type caches across TFMs through instance or
  static fields.
- If the rule depends on target framework, read it explicitly from analyzer
  config / MSBuild-visible properties or infer it from the compilation inputs.
- Test at least one multitargeted consumer project when the rule depends on API
  availability, preprocessor symbols, or package references.

### AdditionalFiles and configuration in multitargeted builds

`AdditionalFiles`, `.editorconfig`, and `.globalconfig` entries are generally
available to each inner build, but they are still consumed in a
**per-compilation** context. The file contents may be identical across TFMs while the
effective meaning differs because the compilation, options, references, or
`build_property.TargetFramework` value differ.

Practical consequence: parse shared files once per compilation, not once per
analyzer instance and not once globally for the whole multitargeted project.

### IDE behavior vs build behavior

The build answer is straightforward: **separate execution per TFM**. IDE live
analysis can be less obvious because Visual Studio typically works through
design-time build/project contexts and may not present every target's results in
exactly the same way or at the same time. When TFM-specific behavior matters,
validate with `dotnet build` or an MSBuild binary log instead of relying only on
the live analysis surface.

### How to verify this in practice

- run `dotnet build /bl /p:ReportAnalyzer=true` on a multitargeted consumer
  project
- inspect the binlog and compiler invocations for separate target-framework
  compilations
- compare diagnostics emitted for each output path / TFM

Official references:

- Microsoft Learn, *Target frameworks in SDK-style projects*:
  <https://learn.microsoft.com/dotnet/standard/frameworks#how-to-specify-a-target-framework>
- Microsoft Learn, *Run a target exactly once*:
  <https://learn.microsoft.com/visualstudio/msbuild/run-target-exactly-once?view=visualstudio>

## 3. Recommended project configuration

Project settings:

- Target `netstandard2.0` (see [dotnet/roslyn#47087](https://github.com/dotnet/roslyn/issues/47087) and [dotnet/roslyn#45162](https://github.com/dotnet/roslyn/issues/45162) issues)
- Set `LangVersion` intentionally. `latest` is convenient for analyzer projects
  built only with modern SDKs, but fixed values such as `latestMajor` or a
  specific C# version make CI and multi-Roslyn builds more reproducible.
- Set MSBuild property `EnforceExtendedAnalyzerRules` to `true` (required/best
  practice for analyzer projects)
- Reference `Microsoft.CodeAnalysis.Analyzers` package (analyzer authoring
  quality checks)
- Reference `Microsoft.CodeAnalysis.CSharp` package (Roslyn C# APIs)
- Keep `DiagnosticAnalyzer` / `DiagnosticSuppressor` types in an analyzer-only
  assembly with compiler-provided references (avoid `Workspaces` references in
  this assembly to prevent `RS1038`)
- Put `CodeFixProvider` types in a separate code-fix assembly; reference
  `Microsoft.CodeAnalysis.Workspaces.Common` there (plus language-specific
  Workspaces packages only when needed)

### `Microsoft.CodeAnalysis` vs `Microsoft.CodeAnalysis.CSharp`

`Microsoft.CodeAnalysis.Common` is the base package that contains all
language-agnostic types: `Compilation`, `SemanticModel`, `SyntaxTree`,
`ISymbol`, `IOperation`, `Diagnostic`, `DiagnosticAnalyzer`, `AnalysisContext`,
etc. It defines the shared contracts that every Roslyn language implementation
builds on.

`Microsoft.CodeAnalysis.CSharp` adds everything specific to C#:
`CSharpCompilation`, `CSharpSyntaxTree`, the full `CSharpSyntaxNode` hierarchy
(all the concrete syntax-node types), `SyntaxKind` (the C# enum),
`CSharpParseOptions`, C#-specific symbol extensions, and so on. It takes a
transitive dependency on `Microsoft.CodeAnalysis.Common`, so referencing
`Microsoft.CodeAnalysis.CSharp` is sufficient for a C# analyzer — there is no
need to list the common package separately.

Reference the common package directly only when writing a language-agnostic
analyzer that must not pull in any language-specific types.

### `Microsoft.CodeAnalysis.Workspaces.Common` vs `Microsoft.CodeAnalysis.CSharp.Workspaces`

`Microsoft.CodeAnalysis.Workspaces.Common` provides the cross-language workspace
model: `Workspace`, `Solution`, `Project`, `Document`, `AdditionalDocument`,
`CodeAction`, `CodeFixProvider`, `WorkspaceServices`, and the infrastructure for
document editing and solution mutation. This package belongs in the **code-fix
assembly**, not in the analyzer-only assembly.

`Microsoft.CodeAnalysis.CSharp.Workspaces` adds C#-specific workspace extensions
on top of the common layer: `CSharpSyntaxFormattingOptions`,
`CSharpSimplificationOptions`, and formatting/simplification services that know
about C# syntax. Reference this package when your code fix produces or
transforms C# syntax and needs the built-in C# formatter or simplifier (e.g., to
add `using` directives, simplify qualified names, or reformat changed nodes).

In practice: reference `Microsoft.CodeAnalysis.Workspaces.Common` for all code
fixes; add `Microsoft.CodeAnalysis.CSharp.Workspaces` only when you invoke
C#-specific formatting or simplification services.

### Why split analyzer and code-fix assemblies (`RS1038`)

`RS1038` warns when compiler extension points are implemented in assemblies that
reference non-compiler-provided assemblies. In analyzer projects, this commonly
appears when `DiagnosticAnalyzer` types share an assembly with `Workspaces`
references.

Why this requirement exists:

- **Different host surfaces:** analyzers run in many compiler hosts (IDE live
  analysis, command-line build, CI, design-time builds). Not every host
  guarantees the full Workspaces layer.
- **Analyzer load must be minimal and reliable:** compiler hosts can load
  analyzer assemblies with only compiler-layer Roslyn dependencies available.
  Pulling in extra dependencies increases load-failure risk.
- **A single assembly load failure disables diagnostics:** if the analyzer
  assembly cannot load, the host skips analyzer execution for that assembly and
  diagnostics from it are lost. The failure may appear only as a build warning,
  IDE notification, or log entry.
- **Code fixes are optional at build time:** `CodeFixProvider` functionality is
  primarily an IDE feature; builds and CI require analyzer diagnostics, not
  code-fix infrastructure.
- **Versioning and probing are safer when separated:** compiler and workspace
  assemblies may have different availability/version constraints across hosts.
  Splitting avoids coupling analyzer activation to workspace-specific resolution
  behavior.

Practical consequence: mixing `DiagnosticAnalyzer` and `CodeFixProvider` into
one assembly can make diagnostics disappear in some environments even when the
analyzer logic itself is correct.

Recommended layout:

- **Analyzer assembly** (`netstandard2.0`): analyzers/suppressors +
  compiler-layer Roslyn references (`Microsoft.CodeAnalysis.CSharp`, analyzers
  package, etc.)
- **Code-fix assembly** (`netstandard2.0`): `CodeFixProvider` implementations +
  `Microsoft.CodeAnalysis.Workspaces.Common` (and optionally
  `Microsoft.CodeAnalysis.CSharp.Workspaces`)

This separation avoids unpredictable host loading behavior and keeps analyzer
binaries compatible across compilation scenarios.

### Language-agnostic analyzers

It is possible to design a language-agnostic analyzer:

- Declares multiple language names in the attribute:
  `[DiagnosticAnalyzer(LanguageNames.CSharp, LanguageNames.VisualBasic)]`
- Registers only operation actions or symbol actions — both of which expose
  language-neutral `IOperation` / `ISymbol` APIs.
- References only `Microsoft.CodeAnalysis.Common` (no `CSharp`- or
  `VisualBasic`-specific packages).

Examples from the [dotnet/sdk
repository](https://github.com/dotnet/sdk/tree/main/src/Microsoft.CodeAnalysis.NetAnalyzers/src/Microsoft.CodeAnalysis.NetAnalyzers):

| Rule | Mechanism |
|---|---|
| **CA2007** — Do not directly await a Task | `IAwaitOperation` (operation action) |
| **CA1822** — Mark members as static | `IMethodSymbol` analysis (symbol action) |
| **CA2016** — Forward CancellationToken to methods | `IInvocationOperation` (operation action) |
| **CA1816** — Dispose methods should call SuppressFinalize | Symbol + operation analysis |

These rules run identically on C# and VB.NET projects because `IOperation`
provides a unified semantic view regardless of the surface syntax. Rules that
require syntax-specific checks (such as formatting or whitespace rules) cannot
be made fully language-agnostic.

Keep analyzers:

- deterministic
- side-effect free
- cancellation friendly
- safe for concurrent execution

## 4. Roslyn package version and host compatibility

The `Microsoft.CodeAnalysis.CSharp` and
`Microsoft.CodeAnalysis.Workspaces.Common` packages **are** the Roslyn compiler
APIs. Each host ships its own copy of these assemblies. The NuGet-restored DLLs
are references you compile against; with `PrivateAssets="all"` and analyzer-style
packing, they should not become dependencies of your analyzer package's
consumers. At runtime the host loads your analyzer against its own Roslyn
assemblies.

**This means: the package version you compile against sets a minimum host
requirement.** If the host's Roslyn is older than the version your analyzer was
compiled against, the analyzer will not load.

Two compiler diagnostics are commonly involved, and they have different
meanings:

- **`CS9057`** — "The analyzer assembly references version `X.Y.Z.W` of the
  compiler, which is newer than the currently running version." This is the
  precise version-mismatch warning when the analyzer was compiled against a
  newer Roslyn than the host's.
- **`CS8032`** — generic "An instance of analyzer cannot be created." This
  surfaces for any analyzer load failure: missing dependency, bad image,
  reflection error, etc. A version mismatch can also end up as `CS8032` once
  the analyzer fails to instantiate, but `CS8032` by itself does not
  necessarily indicate a version problem.

Visual Studio may surface either failure through an info bar, ActivityLog
entry, or analyzer error diagnostic.

### Visual Studio compatibility matrix

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
| 3.10.0 | — | VS 2019 16.10 |
| 3.9.0 | — | VS 2019 16.9 |
| 3.8.0 | 5.0.1xx | VS 2019 16.8 |
| 3.7.0 | — | VS 2019 16.7 |
| 3.6.0 | — | VS 2019 16.6 |
| 3.5.0 | — | VS 2019 16.5 |
| 3.4.0 | 3.1.1xx | VS 2019 16.4 |
| 3.3.1 | — | VS 2019 16.3 |
| 3.2.1 | — | VS 2019 16.2 |
| 3.1.0 | — | VS 2019 16.1 |
| 3.0.0 | — | VS 2019 RTM (16.0) |
| 2.10.0 | 2.1.5xx | VS 2017 15.9 |
| 2.9.0 | — | VS 2017 15.8 |
| 2.8.2 | — | VS 2017 15.7 |
| 2.7.0 | — | VS 2017 15.6 |
| 2.6.1 | — | VS 2017 15.5 |
| 2.4.0 | — | VS 2017 15.4 |
| 2.3.2 | — | VS 2017 15.3 |
| 2.2.0 | — | VS 2017 15.2 |
| 2.1.0 | — | VS 2017 15.1 |
| 2.0.0 | — | VS 2017 RTM (15.0) |
| 1.3.2 | — | VS 2015 Update 3 |
| 1.2.2 | — | VS 2015 Update 2 |
| 1.1.1 | — | VS 2015 Update 1 |
| 1.0.1 | — | VS 2015 RTM |

>  Any version targeting a VS 2017 minimum also works in all later VS
> generations. SDK version cells marked `—` indicate that no specific SDK
> mapping has been confirmed for that minor; use the Visual Studio column as the
> authoritative host constraint for those rows. Keep this table aligned with the
> Microsoft Learn Roslyn version-support table and the .NET SDK/MSBuild/Visual
> Studio versioning table when updating it.

The same applies to `dotnet build`: each .NET SDK version ships a specific
Roslyn, so the analyzer will not activate in CLI builds if the SDK's bundled
Roslyn is older than the referenced version.

### Host selection notes

Compatibility tables are necessary but not sufficient because different hosts
may choose different toolsets for the same repository:

- `dotnet build` uses the SDK selected by the current CLI resolution rules,
  including `global.json` if present.
- Visual Studio may run design-time builds and live analysis against its own
  installed toolset, which can differ from the CLI-selected SDK.
- A newer Visual Studio can still invoke an older SDK if the repository pins
  one via `global.json`.
- A package may restore successfully yet still fail to load at analysis time if
  the host Roslyn is older than the package's referenced
  `Microsoft.CodeAnalysis` version.

When documenting supported hosts for an analyzer package, prefer wording such
as: “Requires Roslyn X.Y or later; validated with SDK A.B and Visual Studio
C.D.” That phrasing is more accurate than assuming SDK and Visual Studio always
advance together.

### Pinned versions and audience implications

To maximize the audience of your analyzer, pin to the **lowest minor version**
that provides the APIs you actually use:

```xml
<!-- Supports VS 2022 RTM (17.0) and all later versions -->
<PackageReference Include="Microsoft.CodeAnalysis.CSharp"
                  Version="4.0.*"
                  PrivateAssets="all" />
<PackageReference Include="Microsoft.CodeAnalysis.Workspaces.Common"
                  Version="4.0.*"
                  PrivateAssets="all" />
```

If you rely on APIs introduced in a later minor (e.g. new `IOperation` kinds),
use that minor as the floor, document it, and update the compatibility table
above accordingly.

## 5. Creating a new analyzer

### Step-by-step

1. Add a new `sealed` class inheriting from `DiagnosticAnalyzer`.
2. Add `[DiagnosticAnalyzer(LanguageNames.CSharp)]`.
3. Define a unique diagnostic ID and descriptor.
4. Implement `SupportedDiagnostics`.
5. In `Initialize`:
   - call `ConfigureGeneratedCodeAnalysis(...)` with the appropriate
     `GeneratedCodeAnalysisFlags`.
     The available flags are:
     - `None` — skip all generated code entirely. Correct for most style and correctness rules.
     - `Analyze` — run analysis on generated code but do not report any resulting diagnostics.
       Use when a rule must inspect generated symbols to reason about hand-written code.
     - `ReportDiagnostics` — allow diagnostics to be reported on generated code. Must be combined
       with `Analyze` (`Analyze | ReportDiagnostics`). Reserve for rules where violations inside
       generated code are genuinely actionable (e.g. security or null-safety rules).
     - `All` — shorthand for `Analyze | ReportDiagnostics`.
   - call `EnableConcurrentExecution()`
   - register minimal callbacks (`OperationKind`, `SyntaxKind`, symbol actions,
     etc.)
6. Implement callback logic with fast guard clauses.
7. Report diagnostics only for actionable cases.

### Example skeleton

```csharp
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public sealed class SampleAnalyzer : DiagnosticAnalyzer
{
    public const string DiagnosticId = "SAMPLE0001";

    private static readonly DiagnosticDescriptor Rule = new(
        id: DiagnosticId,
        title: "Rule title",
        messageFormat: "Rule message",
        category: "Usage",
        defaultSeverity: DiagnosticSeverity.Warning,
        isEnabledByDefault: true,
        description: "Rule description.");

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => [Rule];

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterOperationAction(AnalyzeInvocation, OperationKind.Invocation);
    }

    private static void AnalyzeInvocation(OperationAnalysisContext context)
    {
        // filter + report
    }
}
```

## 6. Roslyn API selection guidance

Choose the narrowest API that matches your rule:

- **Operation actions**: preferred for semantic rules (types, conversions,
  symbols, binding).
- **Syntax actions**: useful for formatting/style checks tied to source form.
- **Symbol actions**: for API-surface rules (types/method declarations,
  attributes).
- **Compilation start/end actions**: for expensive setup reused by many
  callbacks.

In many analyzers, combining operation + syntax is useful:

- use `IOperation` for semantic truth
- use `SyntaxNode` for source-shape details (for example, argument naming
  syntax)

### Common `OperationKind` values

| `OperationKind` | `IOperation` type | Typical use |
|---|---|---|
| `Invocation` | `IInvocationOperation` | Method call analysis |
| `ObjectCreation` | `IObjectCreationOperation` | `new T(...)` analysis |
| `Await` | `IAwaitOperation` | Await expression analysis |
| `Return` | `IReturnOperation` | Return statement analysis |
| `SimpleAssignment` | `ISimpleAssignmentOperation` | Assignment analysis |
| `VariableDeclarator` | `IVariableDeclaratorOperation` | Variable declaration analysis |
| `PropertyReference` | `IPropertyReferenceOperation` | Property read/write analysis |
| `FieldReference` | `IFieldReferenceOperation` | Field read/write analysis |
| `Throw` | `IThrowOperation` | Throw expression/statement analysis |
| `Conversion` | `IConversionOperation` | Implicit/explicit cast analysis |
| `BinaryOperator` | `IBinaryOperation` | Binary expression analysis |
| `Conditional` | `IConditionalOperation` | `if`/ternary analysis |
| `Loop` | `ILoopOperation` | Loop analysis |
| `AnonymousFunction` | `IAnonymousFunctionOperation` | Lambda analysis |
| `LocalFunction` | `ILocalFunctionOperation` | Local function analysis |
| `DelegateCreation` | `IDelegateCreationOperation` | Delegate creation analysis |
| `InterpolatedString` | `IInterpolatedStringOperation` | String interpolation analysis |
| `IsNull` | `IIsNullOperation` | Null check analysis |
| `IsPattern` | `IIsPatternOperation` | Pattern matching analysis |

For a complete list see the [`OperationKind` enum](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.operationkind?view=roslyn-dotnet-5.0.0).

**Discovering the right `OperationKind` for a code pattern:**

1. Write a short unit test or temporary diagnostic callback that reaches the
   syntax node you care about.
2. Get the `SemanticModel` for that node's `SyntaxTree` from the test
   compilation, or use the `SemanticModel` already available in syntax/operation
   callbacks.
3. Call `semanticModel.GetOperation(node, cancellationToken)` and inspect
   `.Kind` plus the concrete `IOperation` type in the debugger.
4. Register the confirmed `OperationKind` and remove the temporary probing code.

Use [Syntax Visualizer](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/syntax-visualizer) for getting syntax nodes.

## 7. Semantic analysis

### Symbol comparison

Always use `SymbolEqualityComparer` instead of reference equality:

```csharp
SymbolEqualityComparer.Default.Equals(symbol, other)
```

`SymbolEqualityComparer` exposes two singletons:

| Comparer | Equality includes | Use when |
|---|---|---|
| `SymbolEqualityComparer.Default` | Symbol identity | Most rules. `string` and `string?` compare equal. |
| `SymbolEqualityComparer.IncludeNullability` | Symbol identity + `NullableAnnotation` | Rules that depend on nullable reference type annotations, where `string` and `string?` must compare *unequal*. |

Pick the comparer deliberately. Using `IncludeNullability` unintentionally can
cause symbols obtained from different syntactic positions (where nullability
flows differ) to compare unequal even though they refer to the same type. Using
`Default` in a nullability-sensitive rule can mask real diagnostics.

### `GetSymbolInfo` vs `GetDeclaredSymbol`

Always carefully distinguish between resolving references and definitions:
- Use `semanticModel.GetDeclaredSymbol()` for syntax nodes that **declare** a
  new symbol (e.g., `ClassDeclarationSyntax`, `MethodDeclarationSyntax`,
  `VariableDeclaratorSyntax`).
- Use `semanticModel.GetSymbolInfo()` for syntax nodes that **reference** an
  existing symbol (e.g., `InvocationExpressionSyntax`, `IdentifierNameSyntax`).

### Type lookup

Use `GetTypeByMetadataName` to resolve types by their fully qualified name:

```csharp
compilation.GetTypeByMetadataName("System.Threading.Tasks.Task")
```

**Caveat:** `GetTypeByMetadataName` returns `null` in two distinct cases:

1. **Not found** — no referenced assembly defines a type with that metadata
   name. This is the common case when the consumer project does not reference
   the package that contains the type.
2. **Ambiguous** — more than one referenced assembly defines a type with the
   same metadata name (e.g., a polyfill package duplicating a framework type).
   In that case the method also returns `null` rather than picking a winner.

Always handle the `null` return defensively. When ambiguity is possible, use
`Compilation.GetTypesByMetadataName(...)` (available since Roslyn 4.1) to
enumerate every match and apply your own selection rule.

### Nullable context awareness

When a rule's correctness depends on whether nullable reference types are
enabled, read the nullable context from the semantic model:

```csharp
var nullableContext = context.SemanticModel.GetNullableContext(node.SpanStart);
var annotationsEnabled = nullableContext.AnnotationsEnabled();
```

`NullableContext` is a flags enum; prefer the extension methods
`AnnotationsEnabled()` and `WarningsEnabled()` from `Microsoft.CodeAnalysis`
over direct bit-testing.

When analyzing an `ITypeSymbol`, check `NullableAnnotation` to distinguish
`string` from `string?`:

```csharp
if (typeSymbol.NullableAnnotation == NullableAnnotation.Annotated)
{
    // type is explicitly nullable (e.g. string?)
}
```

Prefer checking nullable annotations on the `IOperation`'s type rather than the
syntax, because the operation layer normalizes implicit casts and conversions.

### Attribute analysis

To inspect attributes on a symbol, use `ISymbol.GetAttributes()`. Each
`AttributeData` provides:

| Member | Description |
|---|---|
| `AttributeClass` | The `INamedTypeSymbol` of the attribute type |
| `ConstructorArguments` | Positional constructor argument values (`ImmutableArray<TypedConstant>`) |
| `NamedArguments` | Named property/field argument values |
| `ApplicationSyntaxReference` | Syntax node for the attribute application (may be `null` for synthesized attributes, i.e. attributes the compiler/runtime model exposes on the symbol even though there is no matching attribute syntax written in source) |

Here, **synthesized attribute** means an attribute-like entry in Roslyn's symbol
model that was not written explicitly in the user's source code. Because there
is no concrete attribute syntax node to point to, `ApplicationSyntaxReference`
can be `null`. Treat `null` here as “present in metadata or compiler-generated”,
not automatically as an error.

Well-known examples include:

- `AsyncStateMachineAttribute` on `async` methods
- `IteratorStateMachineAttribute` on iterator methods using `yield`
- `TupleElementNamesAttribute` for tuple element name metadata
- `DynamicAttribute` for `dynamic` metadata representation
- `NullableAttribute` and `NullableContextAttribute` for nullable reference
  type metadata

Important distinction: `ApplicationSyntaxReference == null` does not mean only
“compiler synthesized”. It can also mean the attribute came from metadata in a
referenced assembly, where the current compilation has no source syntax node for
that attribute application.

Typical pattern — check whether a symbol carries a specific attribute:

```csharp
private static bool HasAttribute(ISymbol symbol, INamedTypeSymbol? attributeType)
{
    if (attributeType is null)
    {
        return false;
    }

    foreach (var attribute in symbol.GetAttributes())
    {
        if (SymbolEqualityComparer.Default.Equals(attribute.AttributeClass, attributeType))
        {
            return true;
        }
    }

    return false;
}
```

Resolve the attribute type once in `RegisterCompilationStartAction` and capture
it in the callback closure:

```csharp
context.RegisterCompilationStartAction(static startContext =>
{
    var obsoleteType = startContext.Compilation
        .GetTypeByMetadataName("System.ObsoleteAttribute");

    startContext.RegisterSymbolAction(
        ctx => AnalyzeMethod(ctx, obsoleteType),
        SymbolKind.Method);
});
```

### Per-compilation caching with `CompilationStartAction`

`CompilationStartAction` is the canonical place to build per-compilation state
(resolved well-known types, option snapshots, parsed `AdditionalFiles`, etc.).
Key properties to rely on:

- It runs **once per compilation**, before any per-symbol/per-operation
  callback.
- The `CompilationStartAnalysisContext` exposes `Compilation`, `Options`, and a
  `CancellationToken` — enough to do all expensive lookups up front.
- Nested registrations (`RegisterOperationAction`, `RegisterSymbolAction`,
  `RegisterCompilationEndAction`, etc.) capture the resolved state in their
  closures, so per-callback work stays cheap.
- The captured state is scoped to *that* compilation. A new compilation gets a
  fresh `CompilationStartAction` invocation and a fresh closure, which is why
  this pattern is safe even though the analyzer instance is shared.

Guidelines:

- Bail out of the `CompilationStartAction` early (return without registering
  anything) if a required well-known type is missing. The analyzer then
  contributes zero per-operation cost to that compilation.
- Do **not** store the resolved values in instance or static fields of the
  analyzer — that re-introduces cross-compilation state and trips RS1008.
- Prefer `static` lambdas plus parameter passing over closures that capture
  the outer `this`, to make the data flow explicit and avoid accidental field
  capture.

To read a constructor argument value:

```csharp
if (attribute.ConstructorArguments.Length > 0
    && attribute.ConstructorArguments[0].Value is string message)
{
    // use message
}
```

For attributes that can appear on return values or parameters, use
`IMethodSymbol.GetReturnTypeAttributes()` or iterate
`IMethodSymbol.Parameters[i].GetAttributes()`.

## 8. Performance and reliability checklist

- Return early with guard clauses.
- Avoid allocations in hot paths.
- Avoid repeated symbol lookups if they can be cached in compilation-start
  state.
- Avoid expensive work for disabled rules. If a rule is very costly, prefer
  putting it in its own `DiagnosticAnalyzer` type so hosts can skip it when all
  supported diagnostics are disabled. `Rule.GetEffectiveSeverity(context.Compilation.Options)`
  is only a coarse compilation-level hint. Do not use it to skip registering or
  running analysis when the rule can be re-enabled for specific files through
  `.editorconfig` path sections. In per-node callbacks, prefer checking the
  effective options for the current syntax tree before skipping expensive work,
  and let Roslyn handle final diagnostic suppression.

  ```csharp
  private static bool IsRuleDisabledForTree(
      AnalyzerOptions options,
      SyntaxTree syntaxTree,
      string diagnosticId)
  {
      var treeOptions = options.AnalyzerConfigOptionsProvider.GetOptions(syntaxTree);

      return treeOptions.TryGetValue(
              $"dotnet_diagnostic.{diagnosticId}.severity",
              out var severity)
          && string.Equals(severity, "none", StringComparison.OrdinalIgnoreCase);
  }
  ```
- Never throw from callbacks — Roslyn catches analyzer exceptions to prevent
  compiler crashes, but they are not truly silent: Visual Studio may show an
  info-bar notification, and `/reportanalyzer` surfaces exception details in
  build output. Regardless, the analyzer's diagnostics will not be reported for
  that compilation, so guard every code path.
- Honor cancellation tokens when doing heavier work.
- Avoid file/network IO inside analyzer callbacks.

### Practical heuristics from Roslyn team guidance

The Roslyn team shared a concise set of analyzer performance heuristics in
[dotnet/roslyn issue #25259, comment 376116587](https://github.com/dotnet/roslyn/issues/25259#issuecomment-376116587).
The guidance is older, but the core advice still maps well to modern analyzer
authoring:

- **Enable concurrent execution.** Call `EnableConcurrentExecution()` in
  `Initialize` so the host can analyze in parallel when it is safe to do so.
  This guide already treats that as required analyzer hygiene, but it is worth
  repeating because it has a direct impact on IDE and build throughput.
- **Prefer syntax-first filtering before semantic work.** Syntax APIs are often
  cheaper than semantic APIs. When the rule allows it, do a quick syntactic
  filter first, then perform binding or symbol checks only for the reduced set
  of candidate nodes. In practice this often means using syntax to reject most
  cases early, then using `IOperation`, symbols, or the semantic model only for
  the small number of remaining actionable candidates.
- **Prefer stateless analysis when possible.** Per-compilation state built in
  `RegisterCompilationStartAction` is sometimes necessary and is the correct
  pattern when you need shared cached lookups. However, if a rule can be
  expressed without stateful compilation-wide aggregation, the simpler
  stateless form is often faster and easier for the IDE to run incrementally.
- **Treat compilation-wide actions as the most expensive bucket.** Favor the
  narrowest callback that expresses the rule accurately.

The same Roslyn guidance gives a useful rough cost hierarchy for callback
shapes:

- **Fastest:** `SyntaxNodeAction`, `SyntaxTreeAction`
- **Intermediate:** `SymbolAction`, `SemanticModelAction`, `OperationAction`,
  `CodeBlockAction`, `OperationBlockAction`
- **Slowest:** `CompilationAction`

Use the hierarchy as a heuristic, not as an absolute rule. A narrowly filtered
operation action can still be the right design when semantic correctness
matters more than a purely syntax-based approximation.

For measurement and profiling, the same Roslyn comment also points to the
historical `StyleCopTester` tool from the StyleCop Analyzers repository and to
profiling analyzers under Visual Studio. Treat those as supplementary inputs
alongside the benchmark and end-to-end measurement guidance below.

### Caching well-known types

Resolving commonly used framework types (e.g. `Task`, `IDisposable`,
`CancellationToken`) on every callback is expensive. Cache them once in
`RegisterCompilationStartAction` and capture the result in callbacks via
closures:

```csharp
context.RegisterCompilationStartAction(static startContext =>
{
    var types = WellKnownTypes.TryCreate(startContext.Compilation);
    if (types is null)
    {
        return; // required types unavailable — skip all analysis for this compilation
    }

    startContext.RegisterOperationAction(
        ctx => Analyze(ctx, types),
        OperationKind.Invocation);
});
```

Encapsulate all well-known type lookups in a dedicated internal record:

```csharp
private sealed record WellKnownTypes(
    INamedTypeSymbol Task,
    INamedTypeSymbol CancellationToken)
{
    public static WellKnownTypes? TryCreate(Compilation compilation)
    {
        var task = compilation.GetTypeByMetadataName("System.Threading.Tasks.Task");
        var ct   = compilation.GetTypeByMetadataName("System.Threading.CancellationToken");

        return task is null || ct is null ? null : new WellKnownTypes(task, ct);
    }
}
```

Rules:
- Always guard against `null` returns from `GetTypeByMetadataName` and bail out
  early.
- Treat cached type symbols as immutable — never write to them after creation.
- Keep cached state inside the `CompilationStartAction` closure, never in
  static or instance fields (that would violate the stateless analyzer contract
  and trigger RS1008).

### Measuring analyzer performance

For a rigorous performance discipline, use a two-layer strategy:

1. **Micro-benchmarks**: Measure analyzer compute cost (diagnostic and
   non-diagnostic paths) using
   [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) (see [Roslyn
   Analyzer Performance
   Guidelines](https://github.com/dotnet/roslyn-analyzers/blob/main/docs/Performance.md)).
2. **End-to-end build throughput**: Test against large real-world projects to
   capture total build impact.

Track at least:
- Analyzer execution time (`/p:reportanalyzer=true`)
- Task-level build timings (recorded in msbuild `.binlog`)
- Total build time

Integrate checks locally, on pull requests, and in periodic trend monitoring to
catch regressions early.

### Cancellation

Analyzer callbacks are cooperative and must honour cancellation, especially in
IDE hosts where the user can retype quickly and invalidate in-flight analysis.

- Every analysis context (`OperationAnalysisContext`,
  `SyntaxNodeAnalysisContext`, `SymbolAnalysisContext`,
  `CompilationStartAnalysisContext`, …) exposes a `CancellationToken`. Pass it
  to every API that accepts one (e.g., `GetOperation`, `GetSymbolInfo`,
  `SemanticModel.GetDiagnostics`, `AdditionalText.GetText`).
- In loops over symbols, syntax nodes, or operations, call
  `cancellationToken.ThrowIfCancellationRequested()` periodically. The compiler
  treats `OperationCanceledException` matching the supplied token as a clean
  cancellation, not an analyzer failure.
- Do **not** catch `OperationCanceledException` and convert it into a
  diagnostic; let it propagate. Swallowing it makes the host think the
  analyzer succeeded with empty results, which can mask real failures and
  defeat IDE responsiveness.
- Long-running setup inside `CompilationStartAction` should also check the
  token before registering nested actions.

## 9. Diagnostic design

A good diagnostic is:

- **Precise**: points to the exact offending syntax node, not the whole
  statement.
- **Actionable**: the message tells the developer what to fix and why.
- **Stable**: the diagnostic ID never changes meaning; retire old IDs rather
  than repurpose them.

### Diagnostic metadata

Beyond title, message, category, and default severity, consider the rest of the
`DiagnosticDescriptor` contract carefully:

- **`helpLinkUri`**: point to stable documentation for the rule, ideally a
  permanent repository or documentation URL.
- **`customTags`**: use `WellKnownDiagnosticTags` only when the behavior is
  intentional and understood by hosts.
  - `Unnecessary` marks dead or redundant code and enables fading in the IDE.
  - `Telemetry` allows the host to include the rule in telemetry pipelines
    where supported.
  - `NotConfigurable` prevents consumer severity changes and should be rare.
- **`isEnabledByDefault`**: set to `false` only when the rule is experimental,
  expensive, noisy, or intended for staged rollout.

Be conservative with metadata that changes IDE behavior. Once consumers depend
on a diagnostic's shape, category, and configurability, changing those semantics
becomes a breaking experience even if the diagnostic ID stays the same.

### Localization

Instead of hardcoding strings in the `DiagnosticDescriptor`, use
`LocalizableResourceString` mapped to a `.resx` framework. This allows the host
IDE to dynamically switch the analyzer's display language natively.

```csharp
private static readonly LocalizableString Title = new LocalizableResourceString(
    nameof(Resources.AnalyzerTitle), Resources.ResourceManager, typeof(Resources));

private static readonly DiagnosticDescriptor Rule = new(
    DiagnosticId, Title, MessageFormat, Category, DiagnosticSeverity.Warning, isEnabledByDefault: true);
```

>  **Note:** When using `.resx` for localization, you must ensure that the
> localized satellite assemblies (e.g.,
> `analyzers/dotnet/cs/zh-Hans/YourAnalyzer.resources.dll`) are packaged and
> shipped alongside your analyzer DLL. Without them, the fallback strings will
> be used. Alternatively, consider embedding the resources if satellite
> assemblies are not desired.

### Reporting a diagnostic and `messageFormat` arguments

`DiagnosticDescriptor.MessageFormat` is a composite format string. Supply the
placeholder values when constructing the diagnostic:

```csharp
private static readonly DiagnosticDescriptor Rule = new(
    id: "SAMPLE0002",
    title: "Avoid awaiting Task directly",
    messageFormat: "Method '{0}' should call ConfigureAwait(false) on awaited Task",
    category: "Reliability",
    defaultSeverity: DiagnosticSeverity.Warning,
    isEnabledByDefault: true);

var diagnostic = Diagnostic.Create(
    Rule,
    awaitOperation.Syntax.GetLocation(),
    methodSymbol.Name);

context.ReportDiagnostic(diagnostic);
```

Notes:

- The order and count of `{0}`, `{1}`, … in `messageFormat` must match the
  order and count of arguments passed to `Diagnostic.Create`. Mismatches
  produce `FormatException` at runtime.
- Format arguments must be values cheap to render to string (symbol names,
  numbers, simple strings). Avoid passing entire syntax nodes or large object
  graphs.
- Use **`additionalLocations`** to attach secondary locations the IDE can
  navigate to (e.g., the conflicting declaration). Code fixes can read these.
- Use the **`properties`** bag (`ImmutableDictionary<string, string?>`) to
  carry analyzer-to-code-fix metadata. The code-fix provider receives it via
  `Diagnostic.Properties` and can branch on it without re-running expensive
  semantic analysis.

### Choosing the right diagnostic location

Choose the narrowest source location that explains the problem clearly:

- report on the **offending token** when a single keyword, identifier, or
  punctuation token is the problem
- report on the **symbol name** when the rule is about how a declaration is
  named or declared
- report on the **argument expression** when a specific argument value or
  argument shape is invalid
- report on the **whole declaration or statement** only when there is no
  smaller location that still communicates the issue

When a primary location is not enough, attach **additional locations** or
**diagnostic properties** to help code fixes and suppressions reason about the
result without re-deriving expensive context.

### Suppressors and suppression strategy

Use a `DiagnosticSuppressor` only when you need to silence an existing compiler
or analyzer diagnostic because domain-specific knowledge proves the code is
safe. Prefer a normal analyzer when you are introducing new guidance rather than
overriding existing guidance.

When authoring suppressors:

- suppress only diagnostics you fully understand
- use the narrowest suppression condition possible
- document why the suppression is safe and when it applies
- add dedicated tests proving both the suppression case and the non-suppression
  case

Suppressing compiler diagnostics is especially sensitive because an incorrect
suppression can hide a real bug while leaving the user with no visible warning.

### Rule documentation and release hygiene

For each diagnostic ID, publish a dedicated rule reference page and wire
`DiagnosticDescriptor.HelpLinkUri` to a stable URL. Keep one page per rule ID
even if multiple IDs are produced by one analyzer type.

Recommended per-rule page structure:
- Cause
- Rule description
- How to fix violations
- When to suppress warnings
- Example of a violation
- Example of a fix
- Related rules

When severity, message wording, or fix behavior changes, document that change in
release notes so consumers understand whether they need to update configuration
or CI baselines.

## 10. CodeFixProvider

Code fixes produce a **new `Solution`** — they never mutate source in place.

Key rules:

- Register a `CodeAction` via `context.RegisterCodeFix(...)`. The action's
  delegate produces a new `Document` (e.g. `document.WithSyntaxRoot(newRoot)`)
  or a new `Solution` when the fix touches `AdditionalDocument` entries.
- Return the updated `Solution` (not just `Document`) when the fix touches
  `AdditionalDocument` entries.
- Register fix equivalence keys so the IDE can batch-apply the fix across the
  solution.

### Code fix authoring details

In `RegisterCodeFixesAsync`:

- filter to the diagnostic IDs owned by the provider
- use stable, deterministic `equivalenceKey` values so Fix All can group
  identical fixes correctly
- preserve trivia whenever possible so the result respects user formatting and
  comments
- annotate new syntax with `Formatter.Annotation` and `Simplifier.Annotation`
  only when the fix actually relies on Roslyn formatting or simplification
  services

#### `Title` vs `equivalenceKey`

`CodeAction.Title` and `CodeAction.EquivalenceKey` look similar but serve
different purposes:

- **`Title`** is user-facing. It is what the IDE displays in the light-bulb
  menu. Make it short, imperative, and localizable. Two distinct fixes for the
  *same* diagnostic should have different titles.
- **`EquivalenceKey`** is a stable identifier used by `FixAllProvider` to
  decide which fixes can be batched together. Two `CodeAction` instances with
  the same `EquivalenceKey` are treated as the "same fix" for the purpose of
  Fix All. It must therefore depend only on the *kind* of fix, never on the
  specific symbol/location.

A common pitfall is to set `EquivalenceKey = Title` and then localize the
title, which silently breaks Fix All in non-English locales. Use a fixed
culture-independent string (e.g., `nameof(MyCodeFix) + ".AddUsing"`) for
`EquivalenceKey` and a `LocalizableResourceString`-backed value for `Title`.

Code fixes should make the smallest safe change. Avoid rewriting unrelated
syntax or normalizing an entire document when only one node needs to change.

### Custom refactorings without diagnostics

Use a `CodeRefactoringProvider` when the source code is already valid and no
diagnostic should be reported. A `CodeFixProvider` is diagnostic-driven: it is
invoked for diagnostics that already exist and receives those diagnostics in
`RegisterCodeFixesAsync`. A `CodeRefactoringProvider` is selection- and
caret-driven: Visual Studio asks it whether one or more `CodeAction` entries are
available at the current location, even when the code has no squiggle.

Official references:

- Microsoft Learn, [`CodeRefactoringProvider`](https://learn.microsoft.com/dotnet/api/microsoft.codeanalysis.coderefactorings.coderefactoringprovider?view=roslyn-dotnet-5.0.0):
  inherit this type to provide source code refactorings and export the provider
  so the host can offer refactorings in the UI.
- Microsoft Learn, [Quick Actions in Visual Studio](https://learn.microsoft.com/visualstudio/ide/find-and-fix-code-errors?view=visualstudio#use-quick-actions-to-fix-or-refactor-code):
  Quick Actions and Refactorings are surfaced through the light bulb / screwdriver
  UI and the `Ctrl+.` menu.

Typical shape:

```csharp
[ExportCodeRefactoringProvider(LanguageNames.CSharp, Name = nameof(MyRefactoringProvider))]
[Shared]
internal sealed class MyRefactoringProvider : CodeRefactoringProvider
{
    public override async Task ComputeRefactoringsAsync(CodeRefactoringContext context)
    {
        var document = context.Document;
        var cancellationToken = context.CancellationToken;
        var root = await document
            .GetSyntaxRootAsync(cancellationToken)
            .ConfigureAwait(false);

        if (root is null)
        {
            return;
        }

        var node = root.FindNode(context.Span);
        if (!CanRefactor(node))
        {
            return;
        }

        var action = CodeAction.Create(
            "Apply custom refactoring",
            ct => ApplyRefactoring(document, node, ct),
            equivalenceKey: nameof(MyRefactoringProvider));

        context.RegisterRefactoring(action);
    }

    private static bool CanRefactor(SyntaxNode node)
    {
        // Check the selection, syntax shape, and semantic constraints.
        return true;
    }

    private static Task<Document> ApplyRefactoring(
        Document document,
        SyntaxNode node,
        CancellationToken cancellationToken)
    {
        // Return a changed Document or Solution, just like a code fix.
        throw new NotImplementedException();
    }
}
```

Practical rules:

- Put refactoring providers in the Workspaces/IDE-facing assembly, not in the
  compiler-only analyzer assembly. They use MEF (`ExportCodeRefactoringProvider`)
  and `Microsoft.CodeAnalysis.Workspaces.Common`, the same broad host surface as
  code fixes.
- Register a `CodeAction` with `context.RegisterRefactoring(...)`; do not create
  a hidden diagnostic just to make a code fix appear. Hidden diagnostics are
  appropriate only when there is still an analyzer rule and a diagnostic contract
  to configure or suppress.
- Keep `ComputeRefactoringsAsync` cheap. It may be called often while the user
  moves the caret or changes the selection, so filter quickly by span and syntax
  before doing semantic work.
- Use stable, culture-independent `equivalenceKey` values for actions that can
  participate in refactor-all or preview grouping; keep the user-facing action
  title localizable.
- Return updated `Document` or `Solution` values from the action delegate. Like
  code fixes, refactorings do not mutate the editor buffer directly.

#### Keyboard shortcuts for custom refactorings in Visual Studio

Visual Studio exposes custom `CodeRefactoringProvider` actions through the
general **Quick Actions and Refactorings** UI. Users can assign keyboard
shortcuts to Visual Studio commands through **Tools** > **Options** >
**Environment** > **Keyboard** (see Microsoft Learn, [Identify and customize
keyboard shortcuts in Visual Studio](https://learn.microsoft.com/visualstudio/ide/identifying-and-customizing-keyboard-shortcuts-in-visual-studio?view=visualstudio)),
but an individual Roslyn `CodeAction` from a `CodeRefactoringProvider` is not a
standalone Visual Studio command with its own stable command ID.

Consequences:

- Users can bind or rebind the shortcut that opens the Quick Actions menu (for
  example, the default `Ctrl+.` command shown in the Visual Studio Quick Actions
  documentation), then choose the custom refactoring from that menu.
- Users generally cannot bind a shortcut directly to one specific custom Roslyn
  refactoring action shipped only as a `CodeRefactoringProvider`.
- If a dedicated shortcut is required, ship a separate Visual Studio extension
  command (for example, a VSPackage / VisualStudio.Extensibility command) and
  bind that command through VS command infrastructure. The command can then call
  shared transformation code, but that is a Visual Studio extension feature, not
  analyzer-package metadata.

### Fix All in Document/Project/Solution

Your `CodeFixProvider` should heavily prioritize overriding
`GetFixAllProvider()` to support fixing all occurrences at once.

- **`WellKnownFixAllProviders.BatchFixer`**: the framework-supplied
  `FixAllProvider` that batches independent `CodeAction` edits. **It is not the
  default** — `CodeFixProvider.GetFixAllProvider()` returns `null` unless you
  override it, which means no Fix All UI is shown to the user. To opt in,
  override `GetFixAllProvider()` and return `WellKnownFixAllProviders.BatchFixer`
  (or a custom provider). For most simple syntax transformations (like adding a
  `using` statement or rewriting an invocation), `BatchFixer` is sufficient:
  the IDE collects all diagnostics, sequences your registered `CodeAction`
  edits, and applies them cleanly in one batch.
- **Custom `FixAllProvider`**: Sometimes `BatchFixer` is insufficient or
  unsafe. You must implement a custom `FixAllProvider` if your fix modifies
  non-source files (like `AdditionalDocuments` where concurrent modifications
  aren't automatically merged) or if your fixes overlap and conflict in a single
  syntax tree.
  - *Example*: The `LiteralArgumentNamedParameterCodeFixProvider` provides a
    custom `LiteralArgumentNamedParameterFixAllProvider.Instance`. Because the
    code fix appends suppressed method configurations to a `.txt`
    `AdditionalDocument`, triggering the `BatchFixer` over a hundred warnings
    would lead to multiple parallel updates to the exact same text
    document—often resulting in lost entries or corrupted string merges. The
    custom provider solves this by collecting all diagnostics into a single
    `Dictionary<ProjectId, HashSet<string>>`, computing the combined set of
    distinct lines, and invoking a single
    `solution.WithAdditionalDocumentText(...)` update operation.

Minimal custom `FixAllProvider` skeleton for `AdditionalDocument` edits. A real
provider must respect `fixAllContext.Scope`; this sketch shows the project /
solution-shaped aggregation pattern used when the fix writes one shared file per
project.

  ```csharp
  internal sealed class MyFixAllProvider : FixAllProvider
  {
      public static readonly MyFixAllProvider Instance = new();

      public override async Task<CodeAction?> GetFixAsync(FixAllContext fixAllContext)
      {
          // Collect all diagnostics across the requested scope (Document/Project/Solution)
          var diagnosticsByProject = new Dictionary<ProjectId, List<Diagnostic>>();

          foreach (var project in fixAllContext.Solution.Projects)
          {
              var diagnostics = await fixAllContext
                  .GetAllDiagnosticsAsync(project)
                  .ConfigureAwait(false);

              var diagnosticsList = diagnostics.ToList();
              if (diagnosticsList.Count > 0)
              {
                  diagnosticsByProject[project.Id] = diagnosticsList;
              }
          }

          return CodeAction.Create(
              "Apply all fixes",
              ct => ApplyAllFixesAsync(fixAllContext.Solution, diagnosticsByProject, ct),
              fixAllContext.CodeActionEquivalenceKey);
      }

      private static async Task<Solution> ApplyAllFixesAsync(
          Solution solution,
          Dictionary<ProjectId, List<Diagnostic>> diagnosticsByProject,
          CancellationToken cancellationToken)
      {
          foreach (var (projectId, diagnostics) in diagnosticsByProject)
          {
              var project = solution.GetProject(projectId)!;
              // compute combined change once and apply a single WithAdditionalDocumentText call
              solution = await ApplyProjectFixAsync(
                  solution, project, diagnostics, cancellationToken)
                  .ConfigureAwait(false);
          }

          return solution;
      }

      private static Task<Solution> ApplyProjectFixAsync(
          Solution solution,
          Project project,
          List<Diagnostic> diagnostics,
          CancellationToken cancellationToken)
      {
          // ... build new AdditionalDocument text, return updated solution ...
          throw new NotImplementedException();
      }
  }
  ```

### Syntax rewriting with `SyntaxEditor` and `DocumentEditor`

Most code fixes need to modify several nodes in the same document. Use
`DocumentEditor` (which wraps `SyntaxEditor`) rather than building and splicing
syntax trees manually:

```csharp
protected override async Task<Document> FixAsync(
    Document document,
    Diagnostic diagnostic,
    CancellationToken cancellationToken)
{
    var editor = await DocumentEditor
        .CreateAsync(document, cancellationToken)
        .ConfigureAwait(false);

    var root = await document
        .GetSyntaxRootAsync(cancellationToken)
        .ConfigureAwait(false);

    var nodeToReplace = root!.FindNode(diagnostic.Location.SourceSpan);
    var newNode = SyntaxFactory./* build replacement */ ...
        .WithAdditionalAnnotations(Formatter.Annotation);

    editor.ReplaceNode(nodeToReplace, newNode);

    return editor.GetChangedDocument();
}
```

Key `SyntaxEditor` / `DocumentEditor` methods:

| Method | Use |
|---|---|
| `ReplaceNode(old, new)` | Replace one node in the tree |
| `InsertBefore(node, newNodes)` | Insert siblings before a node |
| `InsertAfter(node, newNodes)` | Insert siblings after a node |
| `RemoveNode(node)` | Remove a node (with configurable trivia behavior) |
| `AddAttribute(declaration, attribute)` | Add an attribute to a member declaration |
| `AddMember(declaration, member)` | Add a member to a type declaration |
| `SetModifiers(declaration, modifiers)` | Replace the modifier list |
| `SetAccessibility(declaration, accessibility)` | Change the access modifier |

`DocumentEditor.GetChangedDocument()` returns the new `Document` with all edits
applied atomically. Prefer it over chaining `.WithSyntaxRoot(newRoot)` calls,
which makes it easier to lose edits or accidentally base later changes on an
outdated root.

Annotate replacement nodes with `Formatter.Annotation` and
`Simplifier.Annotation` only when the fix actually relies on Roslyn formatting
or simplification services — adding them unconditionally can cause unexpected
trivia changes in surrounding code.

## 11. Incremental analysis

IDE hosts try to run analyzers incrementally, but the invalidation scope depends
on what changed. A syntax-only edit may re-analyze one document, while semantic
changes, global options, additional files, or project references can trigger
broader re-analysis. Command-line builds are a separate compilation pass.
Therefore:

- Avoid whole-compilation scans inside per-operation or per-syntax callbacks.
- Cache expensive lookups inside `CompilationStartAction`, which runs once per
  compilation.
- Keep callbacks fast; prefer early-exit guard clauses over complex logic.

### What triggers re-analysis

Approximate invalidation scopes used by typical IDE hosts (Visual Studio,
Roslyn LSP, VS Code via OmniSharp):

| Change | Likely scope |
|---|---|
| Edit inside a method body (no public surface change) | Re-analyze the edited document |
| Rename / signature change of a public member | Re-analyze the project (and downstream projects that reference it) |
| Edit to an `AdditionalFiles` entry | Re-run `CompilationStartAction` and dependent callbacks for the owning project |
| Edit to `.editorconfig` / `.globalconfig` | Re-evaluate options; affected projects re-analyze |
| MSBuild property change (`<Property>`) reaching the analyzer | Project-level re-analysis |
| Reference / package version change | Project-level re-analysis (and likely downstream too) |
| Save / build on the command line | Full compilation pass; no incremental state is carried over from a previous run |

Design implications:

- Per-callback work must be cheap *and* idempotent — it may run many times
  per editing session.
- State captured in `CompilationStartAction` must be valid for the whole
  compilation; if it depends on inputs that can change (`AdditionalFiles`,
  options), the host will rebuild the closure for you on the next compilation,
  so you do not need (and must not have) a custom invalidation mechanism.
- Never assume the previous run's data is still in memory. Each compilation is
  independent from the analyzer's perspective.

## 12. Working with AdditionalFiles

`AdditionalFiles` is the standard Roslyn mechanism for passing arbitrary
configuration files into an analyzer at analysis time. The files are **read-only
text** — they are never compiled into the assembly being analyzed.

### Consumer-side setup (project that uses the analyzer)

Declare any configuration file as an `AdditionalFiles` item in the consuming
project's `.csproj`:

```xml
<ItemGroup>
  <AdditionalFiles Include="YourAnalyzer.Exceptions.txt" />
</ItemGroup>
```

The file path is relative to the project file. It can be anywhere on disk so
long as the path resolves at build time.

### TFM-specific `AdditionalFiles` in multitargeted consumer projects

In a multitargeted consumer project, `AdditionalFiles` can also be made
**target-framework-specific**. This matters when one analyzer package needs
different configuration for `net8.0`, `net472`, `net8.0-windows`, and so on.

There are two common patterns:

1. **MSBuild selects different files per TFM** using `Condition` on
   `$(TargetFramework)`.
2. **The analyzer receives multiple files and selects the matching one** based
   on path or filename conventions.

Prefer the first pattern when possible. It keeps the analyzer simpler because
each inner build receives only the files that apply to its `TargetFramework`.

#### Pattern 1: include different files per TFM with MSBuild conditions

Because SDK-style multitargeting runs one inner build per target framework, you
can condition `AdditionalFiles` items on `$(TargetFramework)` just like package
references or other build inputs:

```xml
<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
  <AdditionalFiles Include="AnalyzerConfig/net8.0/YourAnalyzer.Exceptions.txt" />
</ItemGroup>

<ItemGroup Condition="'$(TargetFramework)' == 'net472'">
  <AdditionalFiles Include="AnalyzerConfig/net472/YourAnalyzer.Exceptions.txt" />
</ItemGroup>
```

This is usually the cleanest approach for a folder layout such as:

```plaintext
AnalyzerConfig/
  net8.0/
    YourAnalyzer.Exceptions.txt
  net472/
    YourAnalyzer.Exceptions.txt
```

You can also vary the filename instead of the directory:

```xml
<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
  <AdditionalFiles Include="AnalyzerConfig/YourAnalyzer.net8.0.exceptions.txt" />
</ItemGroup>

<ItemGroup Condition="'$(TargetFramework)' == 'net472'">
  <AdditionalFiles Include="AnalyzerConfig/YourAnalyzer.net472.exceptions.txt" />
</ItemGroup>
```

In these forms, the analyzer usually does **not** need TFM-aware selection
logic at all, because the build already filtered the item list.

#### Pattern 2: include multiple files and select the matching one in the analyzer

Sometimes you intentionally include several configuration files for the same
project and let the analyzer choose the best match. This is useful when:

- a shared props/targets file cannot easily predict the consumer's exact layout
- you want one fallback file plus one or more TFM-specific overrides
- the file naming convention is part of the analyzer package contract

Example folder-per-TFM layout:

```plaintext
AnalyzerConfig/
  net8.0/
    YourAnalyzer.Exceptions.txt
  net472/
    YourAnalyzer.Exceptions.txt
  common/
    YourAnalyzer.Exceptions.txt
```

Example filename-per-TFM layout:

```plaintext
AnalyzerConfig/
  YourAnalyzer.Exceptions.txt
  YourAnalyzer.net8.0.Exceptions.txt
  YourAnalyzer.net472.Exceptions.txt
```

In this model, the analyzer should read the current target framework once per
compilation and apply a deterministic match rule, for example:

1. exact filename or path match for the current TFM
2. shared fallback such as `common` or a filename without TFM suffix
3. otherwise no config file

Do **not** rely on `Options.AdditionalFiles` enumeration order to break ties.
Always define and document precedence explicitly.

#### Reading the current TFM inside the analyzer

If the analyzer itself needs to choose among several `AdditionalFiles`, read the
current target framework from analyzer config / MSBuild-visible properties as
described in [Section 13](#13-reading-msbuild-properties):

```csharp
startContext.Options.AnalyzerConfigOptionsProvider.GlobalOptions
    .TryGetValue("build_property.TargetFramework", out var targetFramework);
```

Then compare that value against either:

- a directory segment such as `...\net8.0\...`
- a filename convention such as `YourAnalyzer.net8.0.Exceptions.txt`

Normalize comparisons carefully:

- compare TFM strings with `StringComparison.OrdinalIgnoreCase`
- use `Path.GetFileName(...)`, `Path.GetDirectoryName(...)`, and path-segment
  parsing rather than raw substring checks when practical
- document whether platform-specific TFMs such as `net8.0-windows` require an
  exact match or may fall back to `net8.0`

#### Recommended precedence rules

When supporting TFM-specific files, choose one precedence model and keep it
stable. Reasonable examples:

- **Exact TFM first, then base TFM, then common fallback**
  - `net8.0-windows` → `net8.0-windows` → `net8.0` → `common`
- **Exact TFM first, then unqualified filename fallback**
  - `YourAnalyzer.net8.0.Exceptions.txt` → `YourAnalyzer.Exceptions.txt`

Be explicit about whether `net8.0-windows` should match only
`net8.0-windows`, or may reuse `net8.0`, and whether `net48` is distinct from
`net481` for your rule's purposes. Those are analyzer-contract decisions, not
something Roslyn decides for you.

#### Practical guidance for folder vs filename conventions

- Use **folder-per-TFM** when each target framework has a larger config set or
  multiple related files.
- Use **filename-per-TFM** when there is only one small file per rule and you
  want the directory to stay flat.
- Use a **single common file** when the content is truly TFM-independent.
- Prefer MSBuild filtering over analyzer-side filtering when consumer project
  control is available.

#### Testing TFM-specific additional files

If the rule supports TFM-specific `AdditionalFiles`, add tests for:

- exact-match TFM file exists
- only fallback file exists
- both exact-match and fallback files exist
- multiple equally plausible matches exist and precedence is enforced
- platform-specific TFMs such as `net8.0-windows`

Official references:

- Microsoft Learn, *Target frameworks in SDK-style projects*:
  <https://learn.microsoft.com/dotnet/standard/frameworks#how-to-specify-a-target-framework>
- Microsoft Learn, *Run a target exactly once*:
  <https://learn.microsoft.com/visualstudio/msbuild/run-target-exactly-once?view=visualstudio>

### Analyzer-side access

Additional files are exposed via `AnalysisContext.Options.AdditionalFiles` (an
`ImmutableArray<AdditionalText>`). Each `AdditionalText` provides:

| Member | Description |
|---|---|
| `Path` | Path supplied by the host. MSBuild normally provides a full path, but tests and custom hosts may provide a relative path. |
| `GetText(CancellationToken)` | Returns a `SourceText` (line-aware, encoding-aware). May return `null` if the file cannot be read. |

`.editorconfig` path sections apply to source files, not to `AdditionalFiles`.
If an additional file needs analyzer-config options, read them with
`context.Options.AnalyzerConfigOptionsProvider.GetOptions(additionalText)` or
put project-wide options in `.globalconfig` / `GlobalOptions`.

For a single well-known config file, matching by **filename only**
(`Path.GetFileName`) keeps the file movable inside the project tree. If your
configuration allows multiple files with the same name, define and document a
deterministic path or precedence rule instead of relying on enumeration order:

```csharp
private static ImmutableHashSet<string> ParseExceptions(
    ImmutableArray<AdditionalText> additionalFiles,
    CancellationToken cancellationToken)
{
    var builder = ImmutableHashSet.CreateBuilder(StringComparer.Ordinal);

    foreach (var file in additionalFiles)
    {
        if (!string.Equals(
                Path.GetFileName(file.Path),
                ExceptionsFileName,
                StringComparison.OrdinalIgnoreCase))
        {
            continue;
        }

        var sourceText = file.GetText(cancellationToken);
        if (sourceText is null)
        {
            continue;
        }

        foreach (var textLine in sourceText.Lines)
        {
            var line = textLine.ToString().Trim();
            if (line.Length == 0 || line.StartsWith("#", StringComparison.Ordinal))
            {
                continue;
            }

            builder.Add(line);
        }
    }

    return builder.ToImmutable();
}
```

### Parse once per compilation — not per callback

Parse `AdditionalFiles` inside `RegisterCompilationStartAction`, not inside
per-operation or per-syntax callbacks. This avoids re-reading and re-parsing the
file for every node:

```csharp
public override void Initialize(AnalysisContext context)
{
    context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
    context.EnableConcurrentExecution();

    context.RegisterCompilationStartAction(static startContext =>
    {
        // Parsed once; captured by all per-operation callbacks for this compilation.
        var exceptions = ParseExceptions(
            startContext.Options.AdditionalFiles,
            startContext.CancellationToken);

        startContext.RegisterOperationAction(
            ctx => AnalyzeInvocation(ctx, exceptions),
            OperationKind.Invocation);
    });
}
```

### Thread safety

Per-operation callbacks run concurrently. The parsed result must be thread-safe.
Prefer `ImmutableHashSet<string>` (or another immutable collection) over
`HashSet<string>`.

### Format conventions

There are no platform constraints on the file format — define whatever suits the
rule. Common choices:

- One entry per line.
- `#` for comments (skip lines starting with `#`).
- Blank lines ignored.
- Case sensitivity matches the domain (use `StringComparer.Ordinal` for
  identifiers).

Example format for a method-exception list:

```plaintext
# Format for methods:      <namespace>.<class>.<method>:<parameter>
# Format for constructors:  <namespace>.<class>:<parameter>
MyApp.Services.EmailService.Send:retryCount
MyApp.Models.Point:x
```

### AdditionalFiles in a CodeFixProvider

A `CodeFixProvider` works with the Workspace model (`Solution`, `Project`,
`AdditionalDocument`) rather than the analyzer's `AdditionalText` model.

| Scenario | API |
|---|---|
| Check whether the file exists | `project.AdditionalDocuments.FirstOrDefault(f => Path.GetFileName(f.Name) == FileName)` |
| Read existing content | `await additionalDocument.GetTextAsync(cancellationToken)` |
| Create a new file | `solution.AddAdditionalDocument(DocumentId.CreateNewId(project.Id, FileName), FileName, content, filePath: ...)` |
| Update existing content | `solution.WithAdditionalDocumentText(additionalDocument.Id, newSourceText)` |

The code fix must return the modified `Solution` (not `Document`) when touching
additional files, because `AdditionalDocument` changes live at the solution
level.

### Testing with AdditionalFiles

In unit tests use a custom `AdditionalText` implementation to inject file
content without touching disk:

```csharp
private sealed class InMemoryAdditionalText(string path, string content) : AdditionalText
{
    public override string Path { get; } = path;

    public override SourceText? GetText(CancellationToken cancellationToken = default)
        => SourceText.From(content, Encoding.UTF8);
}
```

Pass it via `AnalyzerOptions`:

```csharp
var options = new AnalyzerOptions(
    additionalFiles: [new InMemoryAdditionalText("YourAnalyzer.Exceptions.txt", fileContent)]);
```

## 13. Reading MSBuild properties

Analyzers can read arbitrary MSBuild properties from the project being analyzed
— not just file content — through the [`CompilerVisibleProperty`
mechanism](https://learn.microsoft.com/en-us/dotnet/core/project-sdk/msbuild-props#compilervisibleproperty)
and `AnalyzerConfigOptionsProvider`.

### How it works

**Step 1 — expose the property.** Add a `CompilerVisibleProperty` item for each
MSBuild property you want to surface. This is typically placed in a `.props`
file that ships alongside the analyzer (e.g. inside the NuGet package's `build/`
folder), so it is imported automatically by every consuming project:

```xml
<!-- build/YourAnalyzer.props, imported automatically from the NuGet package -->
<Project>
  <ItemGroup>
    <CompilerVisibleProperty Include="TargetFramework" />
    <CompilerVisibleProperty Include="Nullable" />
  </ItemGroup>
</Project>
```

The .NET SDK translates each declared property into an entry in the global
`.editorconfig` that is passed to the compiler:

```ini
build_property.TargetFramework = net8.0
build_property.Nullable = enable
```

**Step 2 — read it in the analyzer.** Use
`AnalyzerOptions.AnalyzerConfigOptionsProvider.GlobalOptions.TryGetValue` with
the `build_property.<PropertyName>` key:

```csharp
context.RegisterCompilationStartAction(static startContext =>
{
    startContext.Options.AnalyzerConfigOptionsProvider.GlobalOptions
        .TryGetValue("build_property.TargetFramework", out var targetFramework);
    // targetFramework == "net8.0", or null when the property is not set / not declared visible
});
```

The entire key (including the property name) is stored in **lower case** in the
underlying `.editorconfig` representation (e.g.
`build_property.targetframework`). However, `AnalyzerConfigOptions.TryGetValue`
uses **case-insensitive comparison**, so querying with
`"build_property.TargetFramework"` works correctly.

### Why `CompilerVisibleProperty` and not direct MSBuild evaluation

Roslyn analyzers run inside the compiler process and have no access to the
MSBuild evaluation context. The `CompilerVisibleProperty` mechanism is the only
supported bridge: MSBuild writes the property values into the `.editorconfig`
stream it passes to the compiler, making them available as analyzer options at
analysis time.

### Safety of `build_property` values

The approach is **not adversarially safe**, but the threat model is constrained:
the consumer controls their own build, so they already have full authority over
any value injected into it. There is no privilege escalation concern. The real
risks are **accidental misconfiguration** and **defensive coding failures**.

Concrete risks and mitigations:

| Risk | Mitigation |
|---|---|
| Value is `null` (property not declared visible, or not set) | Always guard with `if (value is null)` and apply a safe default. |
| Value is an unexpected string (typo, wrong environment) | Use `TryParse` / `Enum.TryParse` — never `Parse` or direct enum cast. |
| Analyzer throws from a callback | Roslyn catches exceptions from analyzer callbacks to prevent compiler crashes; diagnostics are not reported for the failed analyzer. Visual Studio may show an info-bar and `/reportanalyzer` surfaces details in build output. Guard every parse path and never let exceptions escape. |
| Incorrect diagnostics reported (false positives/negatives) | Document accepted values; add a diagnostic or no-op when the value is unrecognized rather than silently falling back to an incorrect behavior. |

Example of a safe read:

```csharp
context.RegisterCompilationStartAction(static startContext =>
{
    startContext.Options.AnalyzerConfigOptionsProvider.GlobalOptions
        .TryGetValue("build_property.Configuration", out var configuration);

    // Treat missing or unrecognized values as the safe default.
    var isRelease = string.Equals(configuration, "Release", StringComparison.OrdinalIgnoreCase);

    startContext.RegisterOperationAction(
        ctx => Analyze(ctx, isRelease),
        OperationKind.Invocation);
});
```

The key rule: **never trust the string value; always validate and default
safely**.

### Comparison with the preprocessor-symbol approach

Both approaches are suitable for reading the target framework. The table below
summarizes the trade-offs:

| | `build_property.TargetFramework` | `CSharpParseOptions.PreprocessorSymbolNames` |
|---|---|---|
| Value delivered | Exact TFM string (`net8.0`) | Symbols like `NET8_0` — requires conversion |
| Requires project opt-in | Yes — `CompilerVisibleProperty` in a `.props` file | No — SDK emits symbols automatically |
| Works in unit tests | Only with a mocked `AnalyzerConfigOptions` | Yes — `CSharpParseOptions.WithPreprocessorSymbols` |
| Multiple values available | Any MSBuild property you declare | Only what the SDK emits as preprocessor defines |

The preprocessor-symbol approach requires no consumer-side opt-in and integrates
cleanly with in-process test helpers
(`CSharpParseOptions.WithPreprocessorSymbols`).

### Testing with MSBuild properties

Because `AnalyzerConfigOptionsProvider` is abstract, supply a test double when
unit-testing property-dependent logic:

```csharp
private sealed class TestAnalyzerConfigOptions(
    IReadOnlyDictionary<string, string> values) : AnalyzerConfigOptions
{
    public override bool TryGetValue(string key, out string value)
        => values.TryGetValue(key, out value!);
}
```

Then pass it via a custom `AnalyzerConfigOptionsProvider` when constructing
`AnalyzerOptions`.

When using `Microsoft.CodeAnalysis.Testing`, you can also inject the generated
analyzer-config text directly:

```csharp
test.TestState.AnalyzerConfigFiles.Add((
    "/.globalconfig",
    "is_global = true\nbuild_property.TargetFramework = net8.0"));
```

### `.editorconfig` and `.globalconfig` guidance

For consumer-facing analyzer configuration, prefer analyzer config files
(`.editorconfig` or `.globalconfig`) as the primary surface. They are the most
natural place for severity control and small textual options, and they work
consistently in both the IDE and command-line builds.

In the normal model, these files are **owned by the consuming repository**, not
by the analyzer author. The consumer decides which rules are enabled, how severe
they are, and which folders or files should use different settings.

Although both formats feed analyzer config options into Roslyn, they are used
in different ways:

| | `.editorconfig` | `.globalconfig` |
|---|---|---|
| Primary purpose | File- and directory-scoped code style / analyzer settings | Project- or package-wide analyzer settings |
| Scope model | Applies by directory and file pattern sections such as `[*.cs]` or `[src/**/*.cs]` | Applies to all source files in the project once imported; no path sections |
| Best for | Different settings for tests, generated code, samples, or specific folders | A single baseline policy shared across many projects |
| Typical location | In the repo tree near the code it governs | At repo root or shared build infrastructure |
| Path-specific overrides | Yes — this is its main strength | No — use `.editorconfig` for path scoping or include different global configs conditionally from MSBuild |
| Typical owner | Consumer repository | Consumer repository |

Practical rule:

- use `.editorconfig` when settings should vary by folder, project area, or
  file pattern
- use `.globalconfig` when you want one default policy for the whole repository
  or organization
- use both when you want a central baseline from `.globalconfig` and local
  overrides in `.editorconfig`

### How Roslyn sees them

Roslyn ultimately treats both files as **analyzer config** inputs. That means
custom options such as `dotnet_code_quality.YourAnalyzer.option_name` and
severity settings such as `dotnet_diagnostic.YOUR_RULE_ID.severity` can come
from either file type.

The important difference is not the key syntax, but the **intended scoping
model**:

- `.editorconfig` is discovered through the source tree and is ideal for
  settings that should vary by path.
- `.globalconfig` is imported as a global analyzer config and is ideal for
  establishing project-wide defaults. It must not use `[*.cs]`-style section
  headers for path scoping.

Neither file type is textually merged into another file. Roslyn receives
multiple analyzer-config inputs and computes the **effective value** for each
key from those separate inputs according to its config resolution rules.

In practice, many teams use the following layering model:

1. `.globalconfig` establishes organization- or repository-wide defaults.
2. `.editorconfig` refines or overrides those defaults for specific directories
   or file sets.

Example baseline in `.globalconfig`:

```ini
is_global = true

dotnet_diagnostic.MY0001.severity = warning
dotnet_code_quality.MyAnalyzers.require_named_arguments = true
dotnet_code_quality.MyAnalyzers.max_depth = 5
```

Example local override in `.editorconfig`:

```ini
[**/Tests/*.cs]
dotnet_diagnostic.MY0001.severity = suggestion
dotnet_code_quality.MyAnalyzers.max_depth = 2

[**/*.Generated.cs]
dotnet_diagnostic.MY0001.severity = none
```

### Resolution and precedence model

Think in terms of **layers**, not file merging:

1. broad defaults from `.globalconfig`
2. path-based refinements from `.editorconfig`
3. the effective value seen by the analyzer for a given syntax tree

That means:

- consumers do **not** need to copy settings out of `.globalconfig` into
  `.editorconfig`
- a consumer override usually works by defining the same key in a more specific
  config layer
- analyzer authors should design option defaults so that consumer-owned config
  can override them cleanly

Important precedence rules:

- within one config file, the later entry for the same key wins
- between `.editorconfig` files, the deeper file-system path wins
- between `.globalconfig` files on .NET 6 or later, the **higher**
  `global_level` wins; equal levels for conflicting keys produce a compiler
  warning and neither value wins. `global_level` is an integer; SDK-generated
  global configs use `100` by default, so consumer-authored global configs that
  must override SDK defaults typically set a value greater than `100` (e.g.,
  `global_level = 200`). Configs without an explicit `global_level` are treated
  as level `0`.
- between `.editorconfig` and `.globalconfig`, the `.editorconfig` entry wins
- for severity, command-line / MSBuild settings such as `-nowarn` and
  `-warnaserror` override analyzer config severity

For analyzer authors, the safe mental model is: “consumer config is
authoritative; any package- or infrastructure-provided defaults must behave like
defaults, not forced policy, unless the package is explicitly intended to
enforce policy.”

### Package-provided defaults and why they are unusual

In some ecosystems, a package or shared build infrastructure may contribute an
additional global analyzer-config input. When that happens, it is still **not**
merged into the consumer's existing `.globalconfig` or `.editorconfig` as text.
Instead, it becomes one more analyzer-config layer that Roslyn evaluates
alongside the consumer-owned files.

This is a specialized pattern, typically for internal engineering systems or
organization-wide policy packages. It is **not** the default expectation for
public analyzer packages.

If you do provide package- or infrastructure-level defaults, be conservative:

- document every imported rule and option clearly
- avoid surprising severity escalations to `error` unless the package is
  explicitly policy-enforcing
- expect consumers to override broad defaults locally with their own
  `.editorconfig`
- test the precedence between the provided defaults and consumer-owned
  overrides

In most public analyzer packages, the better guidance is simply:

- let the consumer create and own `.editorconfig` / `.globalconfig`
- document the supported keys and expected values clearly
- keep package behavior reasonable even when no custom config file is present

Typical uses:

- severity configuration via `dotnet_diagnostic.<ID>.severity`
- per-rule options such as naming, allow-lists, mode switches, or thresholds
- path-scoped options that vary by directory or file pattern

Use MSBuild-backed properties when the option is naturally a project-level build
setting, and use `AdditionalFiles` when you need richer structured or
line-oriented data that would be awkward in an analyzer config key.

### Choosing the right configuration channel

Use the smallest configuration surface that matches the shape and scope of the
option:

| Need | Preferred channel | Why |
|---|---|---|
| Severity, simple booleans, thresholds, per-folder overrides | `.editorconfig` / `.globalconfig` | Natural fit for analyzer policy; supports path scoping and works in IDE + CLI |
| Project-level build facts already known to MSBuild | `CompilerVisibleProperty` | Keeps project-wide settings in the build layer and avoids duplicating values in config files |
| Larger structured data, allow-lists, exception lists, many entries | `AdditionalFiles` | Better for line-oriented or structured content that would be awkward to encode as key/value pairs |
| Temporary test-only overrides | Test harness configuration | Keeps test-specific behavior out of the shipping package contract |

Rules of thumb:

- Prefer `.editorconfig` when human editing and path scoping matter.
- Prefer MSBuild properties when the value already exists naturally in the
  build, such as target framework, configuration, or custom project
  capabilities.
- Prefer `AdditionalFiles` when the configuration is too large or too
  structured for analyzer config keys.
- Avoid exposing the same concept through multiple channels unless you document
  and test a stable precedence order.

Example `.editorconfig` patterns:

```ini
[*.cs]
dotnet_diagnostic.MY0001.severity = warning
dotnet_code_quality.MyAnalyzers.require_named_arguments = true

[**/Tests/*.cs]
dotnet_diagnostic.MY0001.severity = suggestion

[**/*.Generated.cs]
dotnet_diagnostic.MY0001.severity = none
```

### Configuration precedence and option design

When a rule supports multiple configuration channels, document a clear
precedence order and keep it stable. A common and understandable model is:

1. path-scoped analyzer config option (`.editorconfig` / `.globalconfig`)
2. project-wide MSBuild property exposed through `CompilerVisibleProperty`
3. `AdditionalFiles` content for larger structured configuration
4. built-in analyzer default

The exact order may differ for your analyzer, but it must be documented and
tested. Consumers need to know which source of truth wins when multiple knobs
are set.

For custom options:

- use stable, namespaced keys such as
  `dotnet_code_quality.YourAnalyzer.option_name`
- validate untrusted string inputs with `TryParse`-style logic
- choose safe defaults for missing or malformed values
- prefer ignoring bad values over throwing exceptions from callbacks
- document accepted values, defaults, and unsupported combinations
- For expensive analyses (especially dataflow/interprocedural analysis), expose
  tuning knobs (e.g., max method call chain length, interprocedural analysis
  kind) so consumers can opt out if build throughput degrades.

When reading analyzer config options, remember that settings may be defined per
tree rather than globally. Use the syntax tree's options when a rule is intended
to vary by file path or folder.

### Per-tree (file-scoped) analyzer config options

`.editorconfig` settings are often scoped to file patterns (e.g.
`[*.Tests.cs]`). To read options that may vary by file, query the per-tree
`AnalyzerConfigOptions` instead of `GlobalOptions`:

```csharp
// Inside a RegisterSyntaxTreeAction callback:
var treeOptions = treeContext.Options.AnalyzerConfigOptionsProvider
    .GetOptions(treeContext.Tree);

treeOptions.TryGetValue("dotnet_code_quality.MY0001.max_depth", out var rawValue);
var maxDepth = int.TryParse(rawValue, out var parsed) ? parsed : 5;
```

Or inside a `RegisterOperationAction` callback, derive the tree from the
operation's syntax node:

```csharp
var options = context.Options.AnalyzerConfigOptionsProvider
    .GetOptions(context.Operation.Syntax.SyntaxTree);
```

Guideline: use `GlobalOptions` for project-wide settings (e.g.
`build_property.*`) and per-tree options for rules that should vary by directory
or file-name pattern.

## 14. Packaging analyzers

Analyzer assemblies are usually shipped as NuGet packages under the `analyzers`
folder.

### Typical package layout

- `analyzers/dotnet/cs/YourAnalyzerAssembly.dll`

If packaging manually, ensure your `.nupkg` includes analyzer assets in that
path. If using SDK pack conventions, configure package metadata and analyzer
inclusion in the project file.

Useful package metadata to add when publishing:

- `PackageId`
- `Version`
- `Authors`
- `Description`
- `RepositoryUrl`
- `PackageReadmeFile`
- `PackageLicenseExpression` or `PackageLicenseFile`

### Specifying packaging entirely in the `.csproj` file (recommended default)

For SDK-style analyzer projects, keep packaging configuration in the project
file unless you have a specific reason to split it out. This keeps build, pack,
metadata, and analyzer asset layout in one place and usually produces the
lowest-maintenance setup.

Typical responsibilities handled directly in `.csproj`:

- package metadata (`PackageId`, `Version`, `Authors`, `Description`,
  `RepositoryUrl`, license/readme/icon)
- analyzer placement (`IncludeBuildOutput`, explicit `PackagePath` where
  needed)
- bundled MSBuild assets (`build/*.props`, `build/*.targets`, optionally
  `buildTransitive/*`)
- additional package files (rule docs, changelog/readme, sample configs)

Example (single analyzer assembly, SDK-driven packing):

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <IncludeBuildOutput>false</IncludeBuildOutput>

    <PackageId>YourAnalyzer</PackageId>
    <Version>1.0.0</Version>
    <Authors>Your Team</Authors>
    <Description>Analyzer package description.</Description>
    <PackageReadmeFile>README.md</PackageReadmeFile>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <RepositoryUrl>https://github.com/yourorg/yourrepo</RepositoryUrl>
  </PropertyGroup>

  <ItemGroup>
    <None Include="$(OutputPath)$(AssemblyName).dll"
          Pack="true"
          PackagePath="analyzers/dotnet/cs"
          Visible="false" />
    <None Include="build\YourAnalyzer.props" Pack="true" PackagePath="build\" />
    <None Include="build\YourAnalyzer.targets" Pack="true" PackagePath="build\" />
    <None Include="README.md" Pack="true" PackagePath="\" />
  </ItemGroup>
</Project>
```

Prefer `.csproj`-only packing when:

- you package one analyzer project (or a small, straightforward set)
- package inputs are produced by the same build
- you want the simplest CI pipeline (`dotnet pack`) and minimal duplication

### Package asset flow and transitivity

Analyzer packages behave differently from normal library packages, so it helps
to understand which metadata controls which flow:

- **`PrivateAssets="all"` on analyzer authoring dependencies** keeps
  compile-time Roslyn packages such as `Microsoft.CodeAnalysis.CSharp` from
  flowing transitively to consumers of *your* analyzer package.
- **`<DevelopmentDependency>true</DevelopmentDependency>` on the analyzer
  project being packed** sets the `developmentDependency` flag in the produced
  `.nuspec`. Modern NuGet clients honor this flag to mark the package as
  development-time and suppress some forms of transitive flow, but the behavior
  is weaker and more client-dependent than `PrivateAssets="all"` on a
  `PackageReference`. Treat it as a hint, not a guarantee, and continue to use
  `PrivateAssets="all"` on the analyzer's own dependencies to control
  compile-time transitive flow.
- **`build/` assets** are imported into the directly consuming project.
- **`buildTransitive/` assets** flow through an intermediate package into
  downstream consumers.

Use `buildTransitive/` only when you intentionally want package-provided MSBuild
logic to affect projects that reference another package which in turn references
your analyzer package. For most analyzer packages, `build/` is the safer default
because it limits surprise transitive behavior.

Typical intent matrix:

| Goal | Recommended setting |
|---|---|
| Keep Roslyn compile-time packages out of downstream consumers | `PrivateAssets="all"` on those `PackageReference` items |
| Mark the produced analyzer package as development-time only | `<DevelopmentDependency>true</DevelopmentDependency>` |
| Import `.props` / `.targets` only for direct consumers | Pack into `build/` |
| Import `.props` / `.targets` through intermediate packages | Pack into `buildTransitive/` |

Be conservative with transitivity. A `.targets` file that adds or removes
analyzers can be surprising when it suddenly affects projects that never
referenced your package directly.

### Packing an analyzer as part of a library package

Sometimes the goal is not to publish a standalone analyzer package, but to ship
an analyzer **inside the same NuGet package as a library** so that every
consumer of the library automatically gets the analyzer as soon as they add a
`PackageReference` to the library package.

This pattern is common for framework or SDK-style libraries that want to ship
usage guidance, guardrails, or API-specific diagnostics together with the API
surface they validate.

#### When this pattern is appropriate

Use a combined library + analyzer package when:

- the analyzer is specifically about correct usage of that library
- consumers should get the diagnostics automatically with no extra package
  reference
- the analyzer and the library are intended to version together

Prefer a separate analyzer package when:

- the analyzer is broadly useful beyond one library
- consumers may reasonably want the library without the analyzer
- the analyzer has an independent release cadence

#### Package layout for a combined package

The package contains **both** normal library assets and analyzer assets:

```plaintext
lib/
  net8.0/
    YourLibrary.dll
analyzers/
  dotnet/
    cs/
      YourLibrary.Analyzers.dll
build/
  YourLibrary.props
  YourLibrary.targets
```

The `lib/` folder exposes the runtime/reference assembly for the library, while
the `analyzers/` folder causes the compiler and IDE to load the analyzer for
projects that reference the package. The consumer does not need a second
`PackageReference` just for the analyzer.

#### SDK-style packing from the library project

One straightforward approach is to keep the library project packable and pack
the analyzer output from a separate analyzer project into the same `.nupkg`.

Example:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <PackageId>YourLibrary</PackageId>
    <IsPackable>true</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\YourLibrary.Analyzers\YourLibrary.Analyzers.csproj"
                      ReferenceOutputAssembly="false"
                      PrivateAssets="all" />
  </ItemGroup>

  <ItemGroup>
    <None Include="..\YourLibrary.Analyzers\bin\$(Configuration)\netstandard2.0\YourLibrary.Analyzers.dll"
          Pack="true"
          PackagePath="analyzers/dotnet/cs"
          Visible="false" />
  </ItemGroup>
</Project>
```

Key points:

- keep the analyzer in its own analyzer project targeting `netstandard2.0`
- do **not** reference the analyzer assembly as a normal runtime dependency of
  the library
- pack the analyzer DLL explicitly into `analyzers/dotnet/cs`
- continue to keep the analyzer project's Roslyn package dependencies marked
  with `PrivateAssets="all"`

`ReferenceOutputAssembly="false"` is important here because the library should
not compile against or ship the analyzer assembly as a runtime dependency. The
analyzer is a build-time asset for consumers, not part of the library's runtime
API surface.

#### Dedicated pack project for a combined package

If the library and analyzer are built by different projects or pipelines, a
dedicated pack project can be clearer than packing from the library project
itself.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <NoBuild>true</NoBuild>
    <IncludeBuildOutput>false</IncludeBuildOutput>
    <PackageId>YourLibrary</PackageId>
    <IsPackable>true</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <None Include="..\src\YourLibrary\bin\$(Configuration)\net8.0\YourLibrary.dll"
          Pack="true"
          PackagePath="lib/net8.0" />
    <None Include="..\src\YourLibrary.Analyzers\bin\$(Configuration)\netstandard2.0\YourLibrary.Analyzers.dll"
          Pack="true"
          PackagePath="analyzers/dotnet/cs"
          Visible="false" />
  </ItemGroup>
</Project>
```

This approach is often easier when the package is an aggregate of multiple
outputs, or when release engineering wants one packaging entry point.

#### Code fixes and companion dependencies

If the bundled analyzer also ships a separate code-fix assembly, pack it next
to the analyzer assembly inside the same `analyzers/dotnet/cs` folder:

```xml
<None Include="..\YourLibrary.CodeFixes\bin\$(Configuration)\netstandard2.0\YourLibrary.CodeFixes.dll"
      Pack="true"
      PackagePath="analyzers/dotnet/cs"
      Visible="false" />
```

Any non-framework dependencies required by the analyzer or code-fix assembly
must also be packed alongside them if the host needs them at analyzer load time.
Do not assume that runtime dependencies of the library under `lib/` will also
serve as analyzer-load dependencies.

#### Consumer experience

For the consumer, the combined package looks like a normal library package:

```xml
<ItemGroup>
  <PackageReference Include="YourLibrary" Version="1.2.3" />
</ItemGroup>
```

After restore/build:

- the project references the library normally for compilation/runtime use
- the analyzer is loaded automatically by the compiler and IDE
- diagnostics from the analyzer appear without requiring a separate analyzer
  package reference

This is usually the best distribution model when the analyzer expresses the
correct usage contract of the library itself.

#### Validation checklist for combined packages

When packing analyzers into a library package, verify all of the following:

- the library DLL is present under the expected `lib/{tfm}/` path
- the analyzer DLL is present under `analyzers/dotnet/...`
- the analyzer DLL is **not** accidentally included as a runtime assembly under
  `lib/`
- analyzer-only dependencies are available beside the analyzer if required
- `dotnet build` loads the analyzer when the package is consumed by a test
  project
- Visual Studio live analysis also loads the analyzer and, if applicable, the
  bundled code fix

### Packaging with a `.nuspec` (when and why)

Use a `.nuspec` when you need packaging control beyond what is practical in
project metadata, or when packaging must be decoupled from building a single
SDK-style project.

`.nuspec` is often preferable when:

- the package aggregates outputs from multiple projects/repositories
- you need strict file-by-file inclusion rules from prebuilt artifacts
- you package non-SDK or mixed assets in one `.nupkg` and want an explicit file
  manifest
- release engineering owns packaging separately from the analyzer build itself

Trade-offs:

- **Pros:** explicit package manifest, strong control over file mapping, useful
  for composite packages
- **Cons:** duplicated metadata risk (project vs nuspec), more maintenance
  overhead, easier drift over time

Minimal analyzer-oriented `.nuspec` sketch:

```xml
<?xml version="1.0"?>
<package>
  <metadata>
    <id>YourAnalyzer</id>
    <version>1.0.0</version>
    <authors>Your Team</authors>
    <description>Analyzer package description.</description>
  </metadata>
  <files>
    <file src="bin\Release\netstandard2.0\YourAnalyzer.dll" target="analyzers\dotnet\cs" />
    <file src="build\YourAnalyzer.props" target="build" />
    <file src="build\YourAnalyzer.targets" target="build" />
  </files>
</package>
```

Practical guidance:

- start with `.csproj` packaging by default
- introduce `.nuspec` only when you hit a concrete packaging boundary
  (aggregation, non-SDK assets, or release-process separation)
- if both are present, define one source of truth for package metadata to avoid
  divergence

### Supporting Multiple Roslyn Versions

Sometimes an analyzer needs to use newer Roslyn APIs (like
`IIncrementalGenerator` or `FileScopedNamespaceDeclarationSyntax` introduced in
Roslyn 4.0), but still wants to provide a fallback analyzer for users on older
compilers (like VS 2019). Alternatively, you may want to ensure your analyzer
only loads on compatible compiler versions to prevent `CS8032` warnings when a
user installs your package on an older toolchain that cannot load the newer
`Microsoft.CodeAnalysis.dll`.

The .NET SDK supports a `roslyn{version}` folder structure inside the
`analyzers/` directory (e.g. `roslyn4.0`). The SDK will automatically choose the
highest valid `roslyn{version}` folder that is less than or equal to the current
compiler's Roslyn version. To learn more, see the original design proposal:
[dotnet/sdk#20355](https://github.com/dotnet/sdk/issues/20355).

Example layout for an analyzer supporting a Roslyn 3.11 fallback and a Roslyn
4.0 optimized version:

```plaintext
analyzers/
  dotnet/
    roslyn3.11/
      cs/
        YourAnalyzer.dll
    roslyn4.0/
      cs/
        YourAnalyzer.dll
```

Selection and fallback rules:

- The SDK selects the highest `roslyn{version}` folder that is less than or
  equal to the current compiler's Roslyn version.
- Assets outside `roslyn{version}` folders are treated as universally
  applicable and are loaded **in addition to** matching versioned assets. Do
  **not** place the same analyzer both under `analyzers/dotnet/cs` and under
  `analyzers/dotnet/roslyn{version}/cs` unless you intentionally want both
  loaded.
- If you need a lowest-common-denominator fallback, place it in the lowest
  supported `roslyn{version}` folder (for example `roslyn1.0`) instead of
  duplicating it outside versioned folders.

**Note for earlier SDK versions:** If the user is on an older .NET SDK (before
.NET 6), it will not understand the `roslyn{version}` structure by default. You
can ship a `.targets` file inside the `build/` folder of your NuGet package that
manually selects the correct analyzer assembly when
`$(SupportsRoslynComponentVersioning)` is absent. The fallback must target the
lowest Roslyn version you actually support; if the fallback is `roslyn3.11`, SDKs
or Visual Studio versions with Roslyn older than 3.11 are outside the supported
range.

Example `build/YourAnalyzer.targets` for a package that ships both a
`roslyn3.11` and a `roslyn4.0` variant:

```xml
<Project>
  <!--
    On SDKs that support roslyn{version} folder selection natively
    ($(SupportsRoslynComponentVersioning) == 'true'), the SDK already
    picks the right assembly from the versioned analyzers/ folder, so
    nothing needs to be done here.

    On older SDKs that do not set SupportsRoslynComponentVersioning,
    remove any versioned analyzer assets from this package that NuGet already
    resolved, then inject the Roslyn 3.11 assembly manually because it is the
    lowest common denominator.

    The language subfolder (cs / vb / fs) is derived from the project's
    $(Language) property so that the correct assembly is injected
    regardless of the consuming project's language.
  -->
  <PropertyGroup>
    <_YourAnalyzerPackageRoot>$(MSBuildThisFileDirectory)..\</_YourAnalyzerPackageRoot>

    <!-- Map MSBuild $(Language) values to NuGet analyzer subfolder names. -->
    <_YourAnalyzerLanguageFolder Condition="'$(Language)' == 'C#'">cs</_YourAnalyzerLanguageFolder>
    <_YourAnalyzerLanguageFolder Condition="'$(Language)' == 'VB'">vb</_YourAnalyzerLanguageFolder>
    <_YourAnalyzerLanguageFolder Condition="'$(Language)' == 'F#'">fs</_YourAnalyzerLanguageFolder>
  </PropertyGroup>

  <Target Name="_YourAnalyzerSelectFallbackAnalyzer"
          Condition="'$(SupportsRoslynComponentVersioning)' != 'true' and '$(_YourAnalyzerLanguageFolder)' != ''"
          AfterTargets="ResolvePackageDependenciesForBuild;ResolveNuGetPackageAssets">
    <ItemGroup>
      <_YourAnalyzerVersionedItem Include="@(Analyzer)"
                                  Condition="'%(Analyzer.NuGetPackageId)' == 'YourAnalyzer' and $([System.String]::Copy('%(Analyzer.Identity)').IndexOf('roslyn')) &gt;= 0" />
      <Analyzer Remove="@(_YourAnalyzerVersionedItem)" />
      <Analyzer Include="$(_YourAnalyzerPackageRoot)analyzers\dotnet\roslyn3.11\$(_YourAnalyzerLanguageFolder)\YourAnalyzer.dll"
                Condition="Exists('$(_YourAnalyzerPackageRoot)analyzers\dotnet\roslyn3.11\$(_YourAnalyzerLanguageFolder)\YourAnalyzer.dll')" />
    </ItemGroup>
  </Target>
</Project>
```

Pack the targets file alongside the analyzer assets:

```xml
<ItemGroup>
  <None Include="build\YourAnalyzer.targets" Pack="true" PackagePath="build\" />
</ItemGroup>
```

NuGet auto-imports `build/{PackageId}.targets` for every consuming project, so
the fallback activates without any consumer-side configuration.

##### Remove-based pattern for three or more Roslyn bands

The fallback example above removes all versioned assets and injects one specific
fallback DLL. When a package ships **three or more** Roslyn bands, the pattern
used by `System.Text.Json` can be easier to maintain: NuGet resolves all
versioned analyzer items first, and the targets file strips out only the bands
that are too new for the current SDK.

```xml
<Project>
  <!--
    Collect all analyzer items belonging to this package so later targets can
    filter them without touching items from unrelated packages.
  -->
  <Target Name="_YourAnalyzerGatherAnalyzers">
    <ItemGroup>
      <_YourAnalyzerItem Include="@(Analyzer)"
                         Condition="'%(Analyzer.NuGetPackageId)' == 'YourAnalyzer'" />
    </ItemGroup>
  </Target>

  <!--
    On older SDKs that do not set $(SupportsRoslynComponentVersioning), remove
    every band that is newer than the lowest common denominator (roslyn3.11 here).
    The lowest band stays loaded and acts as the fallback.
  -->
  <Target Name="_YourAnalyzerMultiTargeting"
          Condition="'$(SupportsRoslynComponentVersioning)' != 'true'"
          AfterTargets="ResolvePackageDependenciesForBuild;ResolveNuGetPackageAssets"
          DependsOnTargets="_YourAnalyzerGatherAnalyzers">
    <ItemGroup>
      <!-- Strip roslyn4.0 and any future bands — keep only the roslyn3.11 assembly. -->
      <Analyzer Remove="@(_YourAnalyzerItem)"
                Condition="$([System.String]::Copy('%(_YourAnalyzerItem.Identity)').IndexOf('roslyn4')) &gt;= 0" />
    </ItemGroup>
  </Target>
</Project>
```

Key differences from the fallback pattern:

| | Fallback add/remove | Band-pruning remove |
|---|---|---|
| How it works | Removes package versioned assets, then injects one known fallback DLL path | Removes only newer-band items that NuGet already resolved |
| Best for | One fallback band | Two or more bands above the fallback |
| Maintenance | Must update the hardcoded path when bands are added | Add one `Remove` condition per new band |
| Real-world example | Simple single-fallback packages | `System.Text.Json`, packages with 3+ Roslyn bands |

Both patterns are correct; choose based on the number of bands your package
ships.

#### Building and packing multi-Roslyn analyzers

The .NET SDK build system natively dimensions builds by Target Framework (TFM),
but does not have a built-in dimension for "Roslyn Version". To properly build
and pack an analyzer targeting multiple Roslyn versions without complex MSBuild
hacks, the most robust approach is:

1. **Keep logic in a shared location:** Place your common code in a Shared
   Project (`.shproj`) or a standard library folder and link the source files
   using `<Compile Include="..\Shared\**\*.cs" />`.
2. **Create a project per Roslyn version:** Create separate `.csproj` files for
   each baseline Roslyn version you target (e.g.,
   `YourAnalyzer.Roslyn311.csproj` and `YourAnalyzer.Roslyn40.csproj`).
   - **Crucial:** Set `<AssemblyName>YourAnalyzer</AssemblyName>` uniformly
     across all projects so they produce identically named `.dll` files. The
     compiler and NuGet expect the core assembly identity and diagnostic IDs to
     remain completely consistent regardless of which folder it is loaded from.
   - Keep assembly identity stable across variants where possible: same
     assembly name, same strong-name key if used, compatible assembly versioning
     strategy, same package identity, and the same diagnostic IDs.
   - Pin the `Microsoft.CodeAnalysis.CSharp` package reference in each project
     to the respective Roslyn version.
   - Target `netstandard2.0` in all projects.
   - Use preprocessor directives (e.g.,
     `<DefineConstants>$(DefineConstants);ROSLYN4_0_OR_GREATER</DefineConstants>`)
     to #ifdef compiler-specific APIs like `IIncrementalGenerator` or
     `FileScopedNamespaceDeclarationSyntax`.
3. **Mirror the split for code fixes when needed:** If the analyzer ships with
   a separate code-fix assembly, create matching code-fix projects for the same
   Roslyn baselines and pack each code-fix assembly next to the corresponding
   analyzer assembly for that Roslyn band.

Example pack-project snippet showing both analyzer and code-fix DLLs placed per
Roslyn band:

   ```xml
   <ItemGroup>
     <!-- Roslyn 3.11 band -->
     <None Include="..\src\YourAnalyzer.Roslyn311\bin\$(Configuration)\netstandard2.0\YourAnalyzer.dll"
           Pack="true"
           PackagePath="analyzers\dotnet\roslyn3.11\cs\" />
     <None Include="..\src\YourAnalyzer.CodeFixes.Roslyn311\bin\$(Configuration)\netstandard2.0\YourAnalyzer.CodeFixes.dll"
           Pack="true"
           PackagePath="analyzers\dotnet\roslyn3.11\cs\" />

     <!-- Roslyn 4.0 band -->
     <None Include="..\src\YourAnalyzer.Roslyn40\bin\$(Configuration)\netstandard2.0\YourAnalyzer.dll"
           Pack="true"
           PackagePath="analyzers\dotnet\roslyn4.0\cs\" />
     <None Include="..\src\YourAnalyzer.CodeFixes.Roslyn40\bin\$(Configuration)\netstandard2.0\YourAnalyzer.CodeFixes.dll"
           Pack="true"
           PackagePath="analyzers\dotnet\roslyn4.0\cs\" />
   </ItemGroup>
   ```

The SDK selects the entire contents of the winning `roslyn{version}/cs/` folder,
so placing the code-fix DLL next to the analyzer DLL ensures both are loaded or
neither is. Never mix code-fix assemblies across bands (for example, placing the
Roslyn 3.11 code-fix in the Roslyn 4.0 folder) because the code-fix host must be
able to load both assemblies with the same Roslyn runtime.
4. **Aggregate and Pack:** Disable packing on the individual source projects
   (`<IsPackable>false</IsPackable>`) and use a `.nuspec` or a dedicated
   entry-point `.csproj` to aggregate the compiled assets into the precise
   `roslyn{version}` folder hierarchy.

**Example `.nuspec` file layout:**
```xml
<files>
  <!-- Fallback for older MSBuild / compilers -->
  <file src="src\YourAnalyzer.Roslyn311\bin\Release\netstandard2.0\YourAnalyzer.dll" target="analyzers\dotnet\roslyn3.11\cs\" />

  <!-- Optimized/Modern analyzer for newer SDKs -->
  <file src="src\YourAnalyzer.Roslyn40\bin\Release\netstandard2.0\YourAnalyzer.dll" target="analyzers\dotnet\roslyn4.0\cs\" />
</files>
```

If you prefer to pack completely via `.csproj` without a `.nuspec`, use a
dedicated pack project that references the Roslyn-specific build projects and
packs their outputs explicitly.

**Example pack project (`YourAnalyzer.Package.csproj`):**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <NoBuild>true</NoBuild>
    <IncludeBuildOutput>false</IncludeBuildOutput>
    <PackageId>YourAnalyzer</PackageId>
    <IsPackable>true</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <None Include="..\src\YourAnalyzer.Roslyn311\bin\$(Configuration)\netstandard2.0\YourAnalyzer.dll"
          Pack="true"
          PackagePath="analyzers\dotnet\roslyn3.11\cs\" />
    <None Include="..\src\YourAnalyzer.Roslyn40\bin\$(Configuration)\netstandard2.0\YourAnalyzer.dll"
          Pack="true"
          PackagePath="analyzers\dotnet\roslyn4.0\cs\" />
  </ItemGroup>
</Project>
```

**Build orchestration:** Build every Roslyn-specific project first, then pack
from the dedicated pack project or `.nuspec` entry point.

```powershell
dotnet build src/YourAnalyzer.Roslyn311/YourAnalyzer.Roslyn311.csproj -c Release
dotnet build src/YourAnalyzer.Roslyn40/YourAnalyzer.Roslyn40.csproj -c Release
dotnet pack build/YourAnalyzer.Package.csproj -c Release
```

In CI, treat package creation as a separate step after all Roslyn-specific
analyzer and code-fix projects have built successfully.

**Validate the package layout:** After packing, inspect the `.nupkg` and verify
that:

- the expected `analyzers/dotnet/roslyn{version}/...` folders exist,
- only the intended analyzer and code-fix assets are present,
- no duplicate copies exist under both versioned and unversioned analyzer
  folders unless that additive behavior is intentional.

#### Release tracking in multi-project layouts

When the analyzer is split across multiple Roslyn-version projects, maintain
**one shared pair** of release tracking files rather than one pair per project.
Place `AnalyzerReleases.Shipped.md` and `AnalyzerReleases.Unshipped.md` in the
directory that is shared across all Roslyn-specific projects — typically the
root of the shared source folder or the pack project directory.

Wire both files into each Roslyn-version project via a relative path:

```xml
<!-- In YourAnalyzer.Roslyn311.csproj and YourAnalyzer.Roslyn40.csproj -->
<ItemGroup>
  <AdditionalFiles Include="..\Shared\AnalyzerReleases.Shipped.md" />
  <AdditionalFiles Include="..\Shared\AnalyzerReleases.Unshipped.md" />
</ItemGroup>
```

Because diagnostic IDs must be consistent across all Roslyn bands (the same ID
with the same severity appears in every band's `SupportedDiagnostics`), a single
release tracking file accurately represents the public contract of the whole
package. Using separate files per project risks them drifting out of sync and
confusing RS2008 validation.

When all Roslyn-version projects reference the same files, RS2008 validation
runs identically in every build and any diagnostic mismatch is caught regardless
of which band is being compiled.

### One analyzer assembly vs multiple assemblies

Both approaches are valid. Choose based on ownership boundaries, release
cadence, and dependency isolation.

#### Single assembly with multiple analyzers (default for most packages)

Prefer one assembly when analyzers are part of one cohesive rule set and are
versioned/released together.

Benefits:

- simpler packaging (`analyzers/dotnet/cs/YourAnalyzers.dll`)
- one dependency graph to manage
- fewer load/probing edges in host compiler processes
- easier shared infrastructure (helpers, options parsing, common symbols)

Use this model when:

- all rules target the same language(s)
- rules share common helpers or configuration files
- all rules should be updated together in one package version

#### Multiple assemblies in one package

Split analyzers into separate assemblies when they have different lifecycles or
technical constraints.

Use multiple assemblies when:

- parts of the rule set are language-specific vs language-agnostic
- one subset has optional/heavier dependencies you do not want to impose on all
  rules
- teams own different rule groups and release them at different speeds
- you need stricter isolation for experimental vs stable rules

Trade-offs:

- more packaging and version-management complexity
- higher risk of duplicated utilities across assemblies
- more assembly-load surface during analysis

#### Practical recommendation

Start with **one assembly per cohesive analyzer family**. Split only when you
have a concrete boundary (language split, ownership split, dependency split, or
release split) that provides clear operational value.

If you split:

- keep diagnostic IDs globally unique across all assemblies
- avoid shipping the same analyzer in multiple paths that can both load for one
  project
- keep shared code in a common internal assembly/source package to avoid
  divergence

### Creating the package with the correct layout

The SDK pack targets do not automatically move your build output into the
`analyzers/` folder. Suppress normal `lib/` output with `IncludeBuildOutput` and
explicitly pack the analyzer DLL into the analyzer asset path.

Minimal project configuration:

```xml
<PropertyGroup>
  <!-- Optional: mark as development dependency so NuGet treats it as dev-only -->
  <DevelopmentDependency>true</DevelopmentDependency>
  <!-- Recommended: suppress lib/ output so the analyzer DLL does not also
       ship as a runtime reference under lib/. You can omit this if you pack
       the analyzer manually and accept that the build output may also appear
       under lib/{tfm}/ in the package. -->
  <IncludeBuildOutput>false</IncludeBuildOutput>
</PropertyGroup>

<ItemGroup>
  <None Include="$(OutputPath)$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
</ItemGroup>
```

To ship `.props` / `.targets` files that are auto-imported by consuming
projects, place them in a `build/` folder and pack them:

```xml
<ItemGroup>
  <!-- auto-imported by MSBuild from the NuGet package -->
  <None Include="build\YourAnalyzer.props" Pack="true" PackagePath="build\" />
  <None Include="build\YourAnalyzer.targets" Pack="true" PackagePath="build\" />
  <None Include="README.md" Pack="true" PackagePath="\" />
</ItemGroup>
```

The resulting package should contain:

```plaintext
analyzers/
  dotnet/
    cs/
      YourAnalyzer.dll
build/
  YourAnalyzer.props
  YourAnalyzer.targets
```

If you ship code fixes in a separate assembly, pack the code-fix DLL alongside
the analyzer DLL so IDE hosts can discover its MEF exports, while keeping the
`DiagnosticAnalyzer` types themselves in the compiler-clean analyzer assembly:

```xml
<None Include="$(OutputPath)YourAnalyzer.CodeFixes.dll"
      Pack="true"
      PackagePath="analyzers/dotnet/cs"
      Visible="false" />
```

Validate the packed package in both the IDE and `dotnet build`; code-fix
assemblies often carry Workspaces dependencies, so missing companion DLLs can
still show up as host load failures.

### External Dependencies

Analyzers load directly into the compiler host (`csc.exe`, `VBCSCompiler.exe`,
or Visual Studio's Roslyn services).
Normal NuGet package dependencies of the analyzer project do **not** become
runtime dependencies of the compiler host. If your analyzer depends on an
external library such as `Newtonsoft.Json`, restoring that package in the
consumer project is not enough.

Rule of thumb: avoid external dependencies in analyzers if possible. If they are
unavoidable, either merge/shade them into the analyzer assembly (for example with
`ILRepack`) or pack the dependent DLLs alongside the analyzer DLL in the same
analyzer asset folder so the analyzer loader receives them as dependency
locations. Test both `dotnet build` and Visual Studio, because dependency
probing and shadow-copy behavior has historically differed across hosts.

### Packaging a language-agnostic analyzer

NuGet uses the path convention `analyzers/{framework}/{language}/{dll}` to
decide which assemblies to load. The `{language}` segment is optional:

| Path | Loaded for |
|------|------------|
| `analyzers/dotnet/cs/YourAnalyzer.dll` | C# projects only |
| `analyzers/dotnet/vb/YourAnalyzer.dll` | VB.NET projects only |
| `analyzers/dotnet/fs/YourAnalyzer.dll` | F# projects only |
| `analyzers/dotnet/YourAnalyzer.dll` | All .NET languages |

A language-agnostic analyzer (see [Section
3](#3-recommended-project-configuration)) must be placed in `analyzers/dotnet/`
**without** a language subfolder. **Do not place the same assembly in both
`dotnet/` and a language subfolder** (e.g. `dotnet/cs/`) — NuGet resolves both
paths for a matching project, so the analyzer would be loaded twice per
compilation, producing duplicate diagnostics. Placing separate copies in
`dotnet/cs/` and `dotnet/vb/` does not cause double-loading within a single
project (NuGet loads only the matching language folder), but it is redundant and
wasteful — use the language-neutral `dotnet/` path instead.

For a language-agnostic analyzer, pack the DLL manually with an explicit
`PackagePath`:

```xml
<PropertyGroup>
  <!-- Suppress runtime lib output -->
  <IncludeBuildOutput>false</IncludeBuildOutput>
  <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
</PropertyGroup>

<ItemGroup>
  <!-- Place in the language-neutral folder so NuGet loads it for all .NET languages -->
  <None Include="$(OutputPath)$(AssemblyName).dll"
        Pack="true"
        PackagePath="analyzers/dotnet/"
        Visible="false" />
</ItemGroup>
```

The resulting package layout for a language-agnostic analyzer:

```plaintext
analyzers/
  dotnet/
    YourAnalyzer.dll   ← selected for all analyzer-capable .NET language projects
build/
  YourAnalyzer.props
  YourAnalyzer.targets
```

The analyzer still only runs for the languages declared in its
`[DiagnosticAnalyzer(...)]` attribute. For example, an analyzer declared for C#
and Visual Basic is not magically an F# analyzer merely because it is placed in
the language-neutral folder.

If a package needs to ship both language-agnostic shared rules and
language-specific rules, use separate assemblies. This is the pattern used by
`System.Runtime.Analyzers` in the .NET SDK:

```plaintext
analyzers/
  dotnet/
    YourAnalyzer.Common.dll          ← language-agnostic rules
    cs/
      YourAnalyzer.CSharp.dll        ← C#-specific rules
    vb/
      YourAnalyzer.VisualBasic.dll   ← VB.NET-specific rules
```

Each assembly is a separate project. The C#- and VB.NET-specific projects pack
with their respective language folder, while the shared project uses the manual
`PackagePath` approach above.

### SDK-style vs classic project systems

Most modern guidance in this document assumes **SDK-style projects** with
`PackageReference`. In older project systems, the analyzer asset pipeline can
differ:

- **SDK-style projects**: analyzer assets from NuGet and `ProjectReference
  OutputItemType="Analyzer"` generally work as documented in both CLI and IDE
  builds.
- **Classic .csproj / .vbproj projects**: support depends more heavily on the
  installed Visual Studio/MSBuild toolset, and package-based analyzer behavior
  may differ from SDK-style projects.
- **`packages.config`-based projects**: analyzer asset flow is less predictable
  than `PackageReference`; verify actual compiler inputs rather than assuming
  NuGet conventions are identical.
- **.NET Framework consumer projects**: can still consume analyzers
  successfully, but host compatibility is determined by the Visual
  Studio/MSBuild toolset that loads them, not by the target framework of the
  consumer alone.

If you need to support both SDK-style and classic projects, validate at least
one real project of each type. Do not infer compatibility from a single modern
sample project.

## 15. Deploying analyzers

Common deployment options:

1. **NuGet package (recommended)**
   - publish package to internal/public feed
   - consumers add `PackageReference`
   - analyzer runs in IDE and build

2. **Project reference (development)**
   - reference analyzer project from a solution consuming project
   - useful during local rule development

3. **VSIX extension (IDE-focused)**
   - for Visual Studio-only delivery
   - not ideal when build-server enforcement is required

### Referencing an analyzer project in the same solution

When the analyzer project and the consuming project are in the same solution,
reference the analyzer project as an **analyzer** (not as a normal assembly
reference).

In the consuming project's `.csproj`:

```xml
<ItemGroup>
  <ProjectReference Include="..\Analyzers\Analyzers.csproj"
                    OutputItemType="Analyzer"
                    ReferenceOutputAssembly="false" />
</ItemGroup>
```

Why these metadata values matter:

- `OutputItemType="Analyzer"` tells the .NET SDK to wrap the resulting DLL
  output in an `<Analyzer>` item instead of a `<Reference>`.
- `ReferenceOutputAssembly="false"` prevents the analyzer assembly from
  becoming a runtime dependency of the consuming project.

#### What is the `<Analyzer>` MSBuild item?

The `<Analyzer>` item is not the same kind of built-in assembly reference
concept as `<Reference>`. In practice its behavior comes from the imported
compiler targets/toolchain — most commonly the **.NET SDK** plus standard
managed build targets such as `Microsoft.Common.targets`,
`Microsoft.CSharp.targets`, and `Microsoft.VisualBasic.targets`.

When you define an `<Analyzer Include="path\to\analyzer.dll" />` item:
1. The SDK collects all items in that group during the build.
2. It passes them to the compiler task (`csc.exe` or `vbc.exe`) via the
   `/analyzer:path\to\analyzer.dll` command-line switch (or `/a`).
3. The host compiler process then loads those assemblies reflectively and
   executes any supported compiler extension points, such as
   `DiagnosticAnalyzer`, `DiagnosticSuppressor`, `ISourceGenerator`, or
   `IIncrementalGenerator`.

Because the item is SDK/compiler-specific rather than MSBuild-native, you must
provide full paths to `.dll` files. `ProjectReference` handles this path
resolution automatically when combined with `OutputItemType="Analyzer"`, while
manual `<Analyzer>` items (like those in `.targets` files inside NuGet packages)
must explicitly point to the physical binary.

Metadata commonly seen on analyzer items includes:

| Metadata | Meaning |
|---|---|
| `Identity` | The physical analyzer path passed to the compiler |
| `NuGetPackageId` | The package that contributed the analyzer asset; useful when filtering analyzer items in `.targets` files |
| `Visible` | UI/display-oriented metadata; does not control whether the compiler loads the analyzer |

If you need to prove what the build is actually doing, inspect a binary log or
the compiler command line instead of reasoning only from the project file.

Use this approach for inner-loop development because it gives immediate feedback
in the IDE and in `dotnet build` without packing/publishing a NuGet package
first.

If many projects in the solution should consume the same local analyzer, you can
centralize the reference in `Directory.Build.props` and condition it to avoid
applying it to the analyzer project itself.

### Build-time enforcement

To treat diagnostics as errors in consuming projects, configure `.editorconfig`
or project-level warning settings.

Example `.editorconfig`:

```ini
[*.cs]
# escalate one analyzer rule to error
dotnet_diagnostic.YOUR_RULE_ID.severity = error
```

Alternatively, escalate rules via MSBuild `<WarningsAsErrors>` in the consuming
project:

```xml
<PropertyGroup>
  <WarningsAsErrors>$(WarningsAsErrors);YOUR_RULE_ID</WarningsAsErrors>
</PropertyGroup>
```

### CI baseline files

When introducing a new rule into an existing codebase, violations may already
exist across many files. Rather than blocking the build immediately, establish a
baseline so that CI only fails on *new* violations introduced after the baseline
was captured.

**SARIF baseline approach:**

```powershell
# generate a SARIF report of current compiler and analyzer diagnostics
dotnet build /p:ErrorLog=baseline.sarif
```

Commit `baseline.sarif` only if your CI has tooling to compare subsequent SARIF
output against the baseline and fail on new diagnostics. `dotnet format --report`
is useful for formatting/code-style reporting, but it is not a general analyzer
SARIF baseline mechanism.

**Suppression file approach:** For team-agreed suppressions, use a
`.editorconfig` with `dotnet_diagnostic.<ID>.severity = none` for directories
that are out of scope, and progressively tighten as violations are resolved.

**`<NoWarn>` approach:** For short-term bulk suppression during a rollout:

```xml
<PropertyGroup>
  <!-- Temporary: remove entries as violations are fixed -->
  <NoWarn>$(NoWarn);YOUR_RULE_ID</NoWarn>
</PropertyGroup>
```

Document the intended removal date of every `<NoWarn>` entry in a comment so it
does not accumulate silently.

## 16. Versioning and rollout strategy

- Use stable diagnostic IDs (do not repurpose existing IDs).
- Introduce new rules as warning/suggestion first.
- Provide code fixes where practical.
- Document false-positive suppression strategy.
- Roll out in CI before enforcing locally as errors.

## 17. Release tracking

> **Official Documentation**: For the authoritative guide, see
> [`ReleaseTrackingAnalyzers.Help.md`](https://github.com/dotnet/roslyn-analyzers/blob/main/src/Microsoft.CodeAnalysis.Analyzers/ReleaseTrackingAnalyzers.Help.md)
> in the dotnet/roslyn-analyzers repository.

The `Microsoft.CodeAnalysis.Analyzers` package ships analyzers that validate two
special `AdditionalFiles` to help you track which rules your package has shipped
and what is pending for the next release.

### Overview

Release tracking enables third-party analyzer packages to define analyzer
releases with associated versions. Each release can track:

1. **Additions**: Set of new analyzer rules that shipped for the first time in
   this release.
2. **Removals**: Set of old analyzer rules that shipped in an earlier release,
   but are removed starting this release.
3. **Changes**: Set of existing analyzer rules where one of the following
   attributes changed:
   - Category of the diagnostic
   - Default severity of the diagnostic
   - Enabled by default status

### Files

Add two markdown files to your analyzer project as `AdditionalFiles`:

```xml
<ItemGroup>
  <AdditionalFiles Include="AnalyzerReleases.Shipped.md" />
  <AdditionalFiles Include="AnalyzerReleases.Unshipped.md" />
</ItemGroup>
```

Alternatively, when `Microsoft.CodeAnalysis.Analyzers` is installed the IDE
offers a light-bulb fix that creates and wires up both files automatically.

- **`AnalyzerReleases.Shipped.md`** — tracks all shipped analyzer releases.
  This file only records analyzer rules for shipped releases. It cannot be
  changed while work is in progress for the upcoming release. When a new
  analyzer package is released, create a new release section with the shipped
  package version at the bottom of this file, and move all entries from the
  unshipped file under the new shipped release.
- **`AnalyzerReleases.Unshipped.md`** — tracks the upcoming or unshipped
  analyzer release. This file starts empty at the beginning of each release.
  While working on the upcoming release, it tracks additions/removals/changes to
  analyzer rules. After each package publish, move all entries from this file to
  `Shipped.md` and leave this file empty for the next release cycle.

### Why `Unshipped` first?

Use `AnalyzerReleases.Unshipped.md` while the rule or diagnostic change is still
in development because it represents the next package version that does not
exist yet. During that time the rule may still change severity, wording,
category, fix behavior, or even be removed before release. Keeping those entries
in `Unshipped` makes it clear that the public package contract is not final.

`AnalyzerReleases.Shipped.md` is the historical ledger of package versions that
were actually published. Once an entry is moved there, it should describe a
released package, not a planned one.

### When release tracking is not worth enabling

For a **real analyzer package** that will be published, versioned, and consumed
by other repositories or teams, release tracking is generally worth enabling
from the start. It documents the public diagnostic contract and keeps
`SupportedDiagnostics` aligned with the package history.

The main cases where it is usually **not** worth the extra ceremony are:

- a throwaway or one-off analyzer used only inside one repository during a
  migration, cleanup, or audit
- an experimental prototype where diagnostic IDs, severities, and even the rule
  set are expected to churn heavily before any package is published
- a local development setup that is consumed only through `ProjectReference` or
  ad-hoc IDE testing, with no intention to maintain release history

In those cases, the analyzer has no meaningful published package contract yet,
so maintaining `AnalyzerReleases.Shipped.md` / `AnalyzerReleases.Unshipped.md`
can be overhead without much value.

However, as soon as the analyzer is expected to ship as a reusable NuGet
package, be consumed outside the current solution, or maintain stable
diagnostic IDs across releases, enable release tracking. Adding it early is
usually easier than reconstructing package history later.

If you intentionally skip release tracking for a prototype while still using
`Microsoft.CodeAnalysis.Analyzers`, treat that as a temporary decision and
avoid normalizing it for shippable packages; otherwise `RS2008` will correctly
push you back toward the release-tracking workflow.

### File format

The release tracking files use a specific markdown table format. Each release in
`AnalyzerReleases.Shipped.md` is grouped by version and can contain three
sections:

- **New Rules**: Rules added in this release
- **Removed Rules**: Rules removed in this release
- **Changed Rules**: Rules whose category, severity, or enabled-by-default
  status changed in this release

#### Example `AnalyzerReleases.Shipped.md`:

```md
## Release 1.0

### New Rules

Rule ID | Category | Severity | Notes
--------|----------|----------|---------
CA1000  |  Design  |  Warning | CA1000_AnalyzerName, [Documentation](CA1000_Documentation_Link)
CA2000  | Security |  Info    | CA2000_AnalyzerName, [Documentation](CA2000_Documentation_Link)
CA3000  |  Usage   | Disabled | CA3000_AnalyzerName, [Documentation](CA3000_Documentation_Link)


## Release 2.0

### New Rules

Rule ID | Category | Severity | Notes
--------|----------|----------|---------
CA4000  |  Design  |  Warning | CA4000_AnalyzerName, [Documentation](CA4000_Documentation_Link)

### Removed Rules

Rule ID | Category | Severity | Notes
--------|----------|----------|---------
CA3000  |  Usage   | Disabled | CA3000_AnalyzerName, [Documentation](CA3000_Documentation_Link)

### Changed Rules

Rule ID | Category | Severity | Notes
--------|----------|----------|---------
CA2000  | Security | Disabled | CA2000_AnalyzerName, [Documentation](CA2000_Documentation_Link)
```

#### Example `AnalyzerReleases.Unshipped.md`:

`AnalyzerReleases.Unshipped.md` contains the same table structure for `New
Rules`, `Removed Rules`, and `Changed Rules`, but normally without a `##
Release` heading because the package version has not shipped yet:

```md
### New Rules

Rule ID | Category | Severity | Notes
--------|----------|----------|---------
CA5000  | Security |  Warning | CA5000_AnalyzerName
CA6000  |  Design  |  Warning | CA6000_AnalyzerName
```

The `Notes` column typically contains the analyzer class name and an optional
documentation link. You can include markdown-formatted links in the `Notes`
column for better documentation.

### Multi-project scenarios

Note that each analyzer project that contributes an assembly to the analyzer
NuGet package requires these additional files. When you have [multiple
Roslyn-version builds](#supporting-multiple-roslyn-versions), maintain **one
shared pair** of release tracking files rather than one pair per project. Place
them in a shared directory and link them into each Roslyn-version project:

```xml
<!-- In YourAnalyzer.Roslyn311.csproj and YourAnalyzer.Roslyn40.csproj -->
<ItemGroup>
  <AdditionalFiles Include="..\Shared\AnalyzerReleases.Shipped.md" />
  <AdditionalFiles Include="..\Shared\AnalyzerReleases.Unshipped.md" />
</ItemGroup>
```

Because diagnostic IDs must be consistent across all Roslyn bands, a single
release tracking file accurately represents the public contract of the whole
package.

### Release workflow

The typical workflow for managing release tracking files:

1. **During development**: Edit only `AnalyzerReleases.Unshipped.md`. Add new
   rules, record removed rules, or document rule changes in the appropriate
   table sections.
2. **At release time**: When publishing a new package version, create a new `##
   Release <version>` section at the bottom of `AnalyzerReleases.Shipped.md`
   and move all entries from the unshipped file into it.
3. **After release**: Leave `AnalyzerReleases.Unshipped.md` empty for the next
   release cycle.

#### Detailed example workflow

Consider the example files shown above where:

- Version 1.0 shipped three rules: CA1000, CA2000, and CA3000
- Version 2.0 changed CA2000's severity from `Info` to `Disabled`, removed
  CA3000, and added CA4000
- The upcoming release (work in progress) will add CA5000 and CA6000

When version 3.0 is ready to ship, you would:

1. Create a new `## Release 3.0` section at the end of
   `AnalyzerReleases.Shipped.md`
2. Move the entries from `AnalyzerReleases.Unshipped.md` under that new section
3. Clear `AnalyzerReleases.Unshipped.md` (leave it empty)

The updated `AnalyzerReleases.Shipped.md` would then contain all three releases:

```md
## Release 1.0
<!-- ... earlier content ... -->

## Release 2.0
<!-- ... earlier content ... -->

## Release 3.0

### New Rules

Rule ID | Category | Severity | Notes
--------|----------|----------|---------
CA5000  | Security |  Warning | CA5000_AnalyzerName
CA6000  |  Design  |  Warning | CA6000_AnalyzerName
```

And `AnalyzerReleases.Unshipped.md` would be empty, ready for the next
development cycle.

### When to add to `AnalyzerReleases.Shipped.md`

Add entries to `AnalyzerReleases.Shipped.md` only as part of a real package
release. Typical cases:

- a new analyzer package version is being packed and published
- a previously unshipped rule is now included in the released package
- a rule severity/category/message change is part of the released package
- a rule removal is part of the released package

Do **not** add entries to `Shipped` merely because code was merged to `main` or
`master`, because a merge does not guarantee that the change has been published
to consumers yet.

### Automated validation and code fixes

The `Microsoft.CodeAnalysis.Analyzers` package reports diagnostics when the
files are missing, malformed, or out of sync with the rules declared in
`SupportedDiagnostics`, and provides code fixes to keep the files consistent
automatically.

Release tracking analyzer provides diagnostics and code fixes to help you
maintain the shipped and unshipped files. You only need to perform two manual
tasks:

1. **Initial setup**: Create these two files and mark them as additional files
   for your analyzer project (the IDE light-bulb fix can do this
   automatically).
2. **Release management**: Move all entries from the unshipped file to the
   shipped file after each package release.

The analyzer handles the rest, including:

- Warning when a new rule in `SupportedDiagnostics` is not tracked in
  `Unshipped.md`
- Warning when tracked rules don't match the actual diagnostic descriptors
- Providing code fixes to automatically add missing entries
- Validating the format and structure of both files

### Creating the files

You can create the release tracking files in two ways:

#### Using the IDE light bulb (recommended)

When `Microsoft.CodeAnalysis.Analyzers` is installed and the tracking files are
missing, the IDE will offer a light-bulb code fix to create and wire them up
automatically. This is the easiest approach.

#### Manual creation

Alternatively, you can manually create the files:

1. **In Visual Studio**: Right-click the project in Solution Explorer, choose
   "Add → New Item...", select "Text File", and create both
   `AnalyzerReleases.Shipped.md` and `AnalyzerReleases.Unshipped.md`. Then
   right-click each file, select "Properties", and choose "C# analyzer
   additional file" for "Build Action".

2. **In the project file**: Create the two files at your desired location, then
   add to your `.csproj`:

   ```xml
   <ItemGroup>
     <AdditionalFiles Include="AnalyzerReleases.Shipped.md" />
     <AdditionalFiles Include="AnalyzerReleases.Unshipped.md" />
   </ItemGroup>
   ```

### Initial file content

When starting from scratch:

- `AnalyzerReleases.Shipped.md` should be empty if you haven't shipped any
  releases yet, or should contain the history of all previously shipped releases
- `AnalyzerReleases.Unshipped.md` should start empty at the beginning of each
  release cycle

The release tracking analyzer will guide you to add entries for your current
rules once the files are properly configured.

## 18. Troubleshooting

- Analyzer not running:
  - confirm package/reference is restored
  - verify analyzer DLL is in `analyzers/dotnet/cs`
  - check host supports target framework (`netstandard2.0` is safest)
  - check that the host's Roslyn version meets the minimum required by the
    compiled analyzer
  - look for `CS9057` ("analyzer assembly references version X.Y.Z.W of the
    compiler, which is newer than the currently running version"). This is
    the precise version-mismatch warning.
  - look for `CS8032` ("an instance of analyzer cannot be created"). This is
    a generic load failure — it may indicate a version mismatch, a missing
    dependency, a bad image, or a reflection failure during activation.
  - look for `AD0001` in the build/IDE output. Roslyn raises `AD0001` when an
    analyzer throws during execution and includes the inner exception details;
    a single thrown exception silently disables that analyzer for the rest of
    the compilation.
- Rule never reports:
  - validate callback registration kind
  - verify guard clauses are not over-filtering
  - inspect operation/syntax tree shape in debugger
  - confirm analyzer callbacks are not throwing; Roslyn catches analyzer
    exceptions to protect the host, but the analyzer's diagnostics are lost and
    the failure is usually surfaced as `AD0001`, `CS8032`, an info bar, or
    `/p:ReportAnalyzer=true` output
- Rule reports too often:
  - add semantic checks (`ArgumentKind`, `IsImplicit`, symbol comparisons)
  - tighten parent-operation context checks
- IDE reports diagnostic but `dotnet build` does not (or vice versa):
  - IDE and CLI host different Roslyn versions; verify the minimum version
    constraint covers both.
  - Check `.editorconfig` severity overrides — a
    `dotnet_diagnostic.<ID>.severity = none` entry
    scoped to a directory applies in both hosts, but an IDE-only `.editorconfig` that is not on the
    MSBuild include path only affects the IDE.
  - Check `<NoWarn>` in project files — this suppresses diagnostics at the
    MSBuild layer and only
    applies to `dotnet build`/`msbuild`, not to IDE live analysis.
  - Check `<WarningsAsErrors>` — this escalates warnings to errors only during
    `dotnet build`, so
    a warning-level rule may appear as a warning in the IDE and as an error in CI.
  - The severity hierarchy includes `DiagnosticDescriptor.DefaultSeverity`,
    `.globalconfig` / `.editorconfig`, rule-set files in older projects, and
    command-line/MSBuild settings such as `<NoWarn>` and `<WarningsAsErrors>`.
    Command-line/MSBuild warning settings can override analyzer-config severity
    during build, so identify which layer is active in each host before changing
    the rule.
  - For rules that must fire identically in both hosts, avoid host-specific
    suppression and prefer
    `.editorconfig` as the single source of truth for severity configuration.

### Host mismatch checklist

When analyzer behavior differs between developers, CI, `dotnet build`, and
Visual Studio, check the host inputs in this order:

1. `global.json` — does the repository pin an older SDK than expected?
2. `dotnet --info` — which SDK and MSBuild versions does the CLI actually use?
3. Visual Studio version — which Roslyn/targets ship with the IDE host?
4. Binary log (`/bl`) — which analyzer assets and targets executed during the
   build?
5. Analyzer report (`/p:ReportAnalyzer=true`) — which analyzers ran and how
   long did they take?
6. Compiler command line — which `/analyzer:` paths were passed to `csc.exe` or
   `vbc.exe`?

If the wrong analyzer DLL is being loaded, debugging the analyzer code itself is
usually premature; fix the host/input mismatch first.

### Inspecting actual analyzer inputs

Useful commands when troubleshooting packaging and host activation:

```powershell
dotnet --info
dotnet build /bl
dotnet build /p:ReportAnalyzer=true
```

The binary log helps answer “which targets ran and which items existed?”, while
`ReportAnalyzer` helps answer “which analyzers actually executed?”. Those are
different questions and often reveal different classes of issue.

## 19. Security and governance

- Do not read secrets or environment-sensitive files in analyzers.
- Keep diagnostics free from sensitive data.
- Prefer deterministic logic to avoid host/environment-dependent behavior.
- Keep user-facing text clear and localizable when scaling rule sets.

## 20. Validating analyzer quality with Microsoft.CodeAnalysis.Analyzers

`Microsoft.CodeAnalysis.Analyzers` is a NuGet package that ships Roslyn
analyzers targeting analyzer authors. Installing it in your analyzer project
enables diagnostic rules (RS1xxx / RS2xxx) that catch common mistakes and
enforce Roslyn best practices at compile time.

### Adding the package

```xml
<PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.11.0" PrivateAssets="all" />
```

Use `PrivateAssets="all"` so the package is a development-time dependency only
and does not flow to consumers of your NuGet package. Pin an explicit version
that is compatible with the SDK/Visual Studio version used to build the analyzer
project; newer major versions of this authoring-analyzer package may require a
newer Roslyn host.

### Key rules enforced

| Rule | Description |
|---|---|
| RS1008 | Avoid storing per-compilation data in analyzer fields (thread-safety) |
| RS1012 | `RegisterCompilationStartAction` with no registered follow-up actions |
| [RS1022](https://github.com/dotnet/roslyn-analyzers/blob/main/docs/rules/RS1022.md) | Do not use `Workspaces` types from analyzer/compiler-extension implementations |
| RS1024 | Compare symbols for equality, not reference |
| RS1025 | `ConfigureGeneratedCodeAnalysis` must be called in `Initialize` |
| RS1026 | `EnableConcurrentExecution` must be called in `Initialize` |
| RS1030 | Do not call `Compilation.GetSemanticModel()` inside a callback |
| [RS1038](https://github.com/dotnet/roslyn-analyzers/blob/main/docs/rules/RS1038.md) | Keep compiler extension implementations in assemblies with compiler-provided references |
| [RS1041](https://github.com/dotnet/roslyn-analyzers/blob/main/docs/rules/RS1041.md) | Keep compiler extension implementations in `netstandard2.0` assemblies |
| RS2008 | Enable analyzer release tracking (`AnalyzerReleases.*.md` files required) |

### Practical remediation guidance for common RS rules

The warnings above are not style-only checks; they protect analyzer reliability,
host compatibility, and deterministic execution. Use the following remediation
patterns:

- **RS1008**: move compilation-specific state out of analyzer instance/static
  fields and build it in `RegisterCompilationStartAction`; capture immutable
  state in nested callbacks.
- **RS1012**: if you register `CompilationStartAction`, ensure it registers at
  least one `OperationAction` / `SyntaxNodeAction` / `SymbolAction` /
  `CompilationEndAction`.
- **RS1022**, **RS1038**, and **RS1041** form a unified compatibility policy:
  - `RS1041` ensures target-framework compatibility across all hosts (must be
    `netstandard2.0`).
  - `RS1038` and `RS1022` ensure runtime load reliability by forbidding
    `Workspaces` APIs and non-compiler dependencies in the core analyzer
    assembly. Place your analyzer and code-fix logic in separate assemblies.
- **RS1024**: compare symbols with `SymbolEqualityComparer.Default` (or
  `IncludeNullability` when needed), never by reference equality.
- **RS1025 + RS1026**: call both `ConfigureGeneratedCodeAnalysis(...)` and
  `EnableConcurrentExecution()` in every analyzer `Initialize` implementation.
- **RS1030**: avoid `Compilation.GetSemanticModel()` in callbacks; prefer
  semantic data already available from operation/symbol actions, or restructure
  the analyzer so the host supplies the `SemanticModel` through an appropriate
  context.
- **RS2008**: keep `AnalyzerReleases.Unshipped.md` in sync during development
  and move entries to `AnalyzerReleases.Shipped.md` only at actual package
  release time.

Quick verification pass before shipping:

1. Build analyzer and code-fix projects with analyzer warnings enabled.
2. Confirm no RS10xx/RS20xx warnings remain.
3. Validate release-tracking files match `SupportedDiagnostics`.
4. Run analyzer tests (positive, negative, boundary, and
   configuration-dependent cases).

### Release tracking integration

RS2008 validates the release tracking markdown files described in [Section
17](#17-release-tracking). When both files are present and
`SupportedDiagnostics` is consistent with their contents, the IDE offers
light-bulb fixes to keep the files synchronized automatically.

### IDE support

When the package is installed, the IDE provides:

- Code fixes to create and wire up `AnalyzerReleases.Shipped.md` and
  `AnalyzerReleases.Unshipped.md`.
- Refactoring support to update release entries when rules are added, removed,
  or changed.

## 21. Testing Analyzers

A dedicated test project validates analyzer behavior with focused unit tests.

### Testing goals for Roslyn analyzers

Analyzer tests should prove:

- diagnostics are reported for true violations
- diagnostics are not reported for valid code (false-positive protection)
- diagnostic IDs/messages/locations are correct
- behavior is stable across language constructs and edge cases

Tests compile inline source and run analyzers through
`Compilation.WithAnalyzers(...)`.

### Modern test harness architecture

The modern industry standard for testing analyzers is the
[`Microsoft.CodeAnalysis.Testing`
suite](https://github.com/dotnet/roslyn-sdk/tree/main/src/Microsoft.CodeAnalysis.Testing)
(e.g., `Microsoft.CodeAnalysis.CSharp.Analyzer.Testing` and
`Microsoft.CodeAnalysis.CSharp.CodeFix.Testing`).

Use the language-specific **generic** packages (no test-framework suffix).
Legacy packages such as `*.MSTest`, `*.NUnit`, and `*.XUnit` are obsolete.

#### Recommended verifier types

Prefer the built-in verifier helpers:

- `CSharpAnalyzerVerifier<TAnalyzer, DefaultVerifier>`
- `CSharpCodeFixVerifier<TAnalyzer, TCodeFix, DefaultVerifier>`
- `VisualBasicAnalyzerVerifier<TAnalyzer, DefaultVerifier>`
- `VisualBasicCodeFixVerifier<TAnalyzer, TCodeFix, DefaultVerifier>`

A common pattern is to alias the verifier in each test file:

```csharp
using Verify = Microsoft.CodeAnalysis.CSharp.Testing.CSharpCodeFixVerifier<
    MyAnalyzer,
    MyCodeFix,
    Microsoft.CodeAnalysis.Testing.DefaultVerifier>;
```

To express expected diagnostics, use the verifier's `Diagnostic(...)` factory
and the fluent `DiagnosticResult` builder methods (`WithLocation`,
`WithArguments`, `WithSeverity`, etc.):

```csharp
var expected = Verify.Diagnostic("MY0001")
    .WithLocation(0)
    .WithArguments("someSymbolName");
```

`DiagnosticResult` is a struct, not a static class — `using static` on it does
not import anything useful. The factory methods are accessed through the
verifier alias (e.g., `Verify.Diagnostic(...)`) or through
`DiagnosticResult.CompilerError("CSxxxx")` / `CompilerWarning("CSxxxx")` when
you need to assert compiler-emitted diagnostics.

#### Package/version alignment

The testing packages may pull older Roslyn dependencies. If analyzer and test
projects conflict, pin the test project's workspace package to the same Roslyn
version used by the analyzer project:

- C#: `Microsoft.CodeAnalysis.CSharp.Workspaces`
- VB: `Microsoft.CodeAnalysis.VisualBasic.Workspaces`

Using these verification packages eliminates most boilerplate. A typical test
defines source code with markup indicating where diagnostics should appear:

```csharp
[Fact]
public async Task Reports_diagnostic_on_violation()
{
    var source = @"
using System.Threading.Tasks;

class C
{
    void M()
    {
        [|Task.Delay(1)|];
    }
}";
    await Verify.VerifyAnalyzerAsync(source);
}
```

#### Markup syntax in test sources

`Microsoft.CodeAnalysis.Testing` supports inline markup inside test source
strings to declare expected diagnostic locations.

Common forms:

- `[| ... |]` — marks one expected diagnostic span (for the analyzer's default
  diagnostic).
- `{|RuleId: ... |}` — marks one expected diagnostic span tied to a specific
  diagnostic ID.
- `{|#0: ... |}` — marks a named location (`0`) that you can reference from
  expected diagnostics with `.WithLocation(0)`.

Example using an indexed location:

```csharp
var source = """
class C
{
    void M(C instance)
    {
        {|#0:instance?.DoWorkAsync();|}
    }
}
""";

await Verify.VerifyAnalyzerAsync(source, Verify.Diagnostic().WithLocation(0));
```

In this example, `{|#0: ... |}` means: “create a diagnostic location labeled `0`
for this exact span”, and `WithLocation(0)` asserts that the reported diagnostic
is at that labeled span.

#### Advanced configuration and testing code fixes

Prefer `VerifyCodeFixAsync` when the analyzer has a code fix. It verifies
diagnostics, verifies the fix output, and validates Fix All behavior when
supported. Use `VerifyAnalyzerAsync` mainly for no-diagnostic scenarios or
analyzers without code fixes.

When a diagnostic is expected but no fix should be offered, pass the same source
as both input and fixed code.

When the standard `VerifyAnalyzerAsync` or `VerifyCodeFixAsync` is not enough,
you can instantiate the test class directly. This allows you to configure
reference assemblies, add additional files, or specify compiler options:

```csharp
[Fact]
public async Task Test_With_Custom_Configuration()
{
    var test = new CSharpCodeFixTest<MyAnalyzer, MyCodeFix, DefaultVerifier>
    {
        TestCode = "class C { }",
        FixedCode = "class C { /* fixed */ }",
        ReferenceAssemblies = ReferenceAssemblies.Net.Net80
    };

    // Add expected diagnostics manually if markup `[|...|]` is not used
    test.ExpectedDiagnostics.Add(Verify.Diagnostic("MY0001").WithLocation(1, 7));

    await test.RunAsync();
}
```

You can also inject custom analyzer options or `.editorconfig` content directly
into the test state:

```csharp
var test = new CSharpAnalyzerTest<MyAnalyzer, DefaultVerifier>
{
    TestCode = "class C { }",
};

test.TestState.AnalyzerConfigFiles.Add(("/.editorconfig", "[*.cs]\ndotnet_diagnostic.MY0001.severity = error"));

await test.RunAsync();
```

#### Adding NuGet Packages and Custom Assemblies

If the code you are analyzing depends on external NuGet packages or custom
assemblies that aren't part of the predefined `ReferenceAssemblies.Net.Net80`
set, you can easily add them to your test configuration.

> **Version note:** the `ReferenceAssemblies.Net.Net80` (and similar `NetXY`)
> properties require a recent enough `Microsoft.CodeAnalysis.Testing` package.
> Older versions expose only `ReferenceAssemblies.NetCore.NetCoreApp31`,
> `ReferenceAssemblies.NetFramework.Net472.Default`, etc. If the property does
> not resolve, upgrade the testing packages or use the older naming.

**Adding a NuGet Package:** You can append packages to the baseline reference
assemblies using `AddPackages`:

```csharp
var test = new CSharpAnalyzerTest<MyAnalyzer, DefaultVerifier>
{
    TestCode = "class C { Newtonsoft.Json.Linq.JObject obj; }",
    ReferenceAssemblies = ReferenceAssemblies.Net.Net80.AddPackages(
        ImmutableArray.Create(new PackageIdentity("Newtonsoft.Json", "13.0.3")))
};
```

**Adding Custom Assemblies:** If you need to reference a specific DLL from disk
or an assembly loaded in your current application domain, add it directly to
`TestState.AdditionalReferences`:

```csharp
var test = new CSharpAnalyzerTest<MyAnalyzer, DefaultVerifier>
{
    TestCode = "class C { MyCustomLibrary.MyType obj; }",
    ReferenceAssemblies = ReferenceAssemblies.Net.Net80
};

// Reference the assembly containing 'MyType'
test.TestState.AdditionalReferences.Add(
    MetadataReference.CreateFromFile(typeof(MyCustomLibrary.MyType).Assembly.Location));
```

These packages handle:
- Automatic workspaces and project `Compilation` creation.
- Seamless reference assembly fetch targeting (e.g., pulling `net8.0` SDK
  references).
- Asserting correct diagnostic spans and `FixAll` completion behavior.
- Verification of `CodeFixProvider` iterations, fixed output, and Fix All output.

While manual `AdhocWorkspace` and `CSharpCompilation` setups are occasionally
useful for highly specialized, specific testing environments,
`Microsoft.CodeAnalysis.Testing` packages are strongly recommended for almost
all scenarios.

### Testing across Roslyn versions

When the analyzer ships separate builds per Roslyn version (see [Supporting
Multiple Roslyn Versions](#supporting-multiple-roslyn-versions)), the test suite
must exercise each band independently. A single shared test project that pulls
in one Roslyn version is not sufficient: a regression or false positive can
exist only in the Roslyn 4.0 build and be invisible to tests that only run
against the Roslyn 3.11 build.

#### Per-band test project layout

Mirror the production project split with a matching test project per Roslyn
band:

```plaintext
tests/
  YourAnalyzer.Tests.Roslyn311/
    YourAnalyzer.Tests.Roslyn311.csproj   ← references YourAnalyzer.Roslyn311
  YourAnalyzer.Tests.Roslyn40/
    YourAnalyzer.Tests.Roslyn40.csproj    ← references YourAnalyzer.Roslyn40
  Shared/
    AnalyzerTests.cs                      ← linked into both test projects
```

Share test source the same way as production source: link the `.cs` files from a
`Shared/` folder or use a shared project. This avoids duplicating the test
matrix for rules that behave identically across bands.

```xml
<!-- In YourAnalyzer.Tests.Roslyn311.csproj -->
<ItemGroup>
  <Compile Include="..\Shared\**\*.cs" />
</ItemGroup>

<ItemGroup>
  <ProjectReference Include="..\..\src\YourAnalyzer.Roslyn311\YourAnalyzer.Roslyn311.csproj" />
</ItemGroup>
```

#### Pinning Roslyn references in test projects

Each test project must pin its `Microsoft.CodeAnalysis.CSharp.Workspaces`
reference to the same Roslyn version used by the corresponding analyzer project.
Mismatched versions cause the testing framework to compile test sources against
one Roslyn runtime while the analyzer was built against another, leading to load
errors or misleading test results.

```xml
<!-- In YourAnalyzer.Tests.Roslyn311.csproj -->
<PackageReference Include="Microsoft.CodeAnalysis.CSharp.Workspaces" Version="3.11.*" />
<PackageReference Include="Microsoft.CodeAnalysis.CSharp.Analyzer.Testing" Version="1.*" />
```

```xml
<!-- In YourAnalyzer.Tests.Roslyn40.csproj -->
<PackageReference Include="Microsoft.CodeAnalysis.CSharp.Workspaces" Version="4.0.*" />
<PackageReference Include="Microsoft.CodeAnalysis.CSharp.Analyzer.Testing" Version="1.*" />
```

#### Using preprocessor directives in shared tests

When a test covers behavior that only exists in one Roslyn band, guard it with
the same preprocessor constant used in the production code:

```csharp
#if ROSLYN4_0_OR_GREATER
[Fact]
public async Task Reports_diagnostic_for_file_scoped_namespace()
{
    // FileScopedNamespaceDeclarationSyntax is only available in Roslyn 4.0+
    var source = "namespace N;";
    await Verify.VerifyAnalyzerAsync(source);
}
#endif
```

Define the constant in the test project the same way as in the production
project:

```xml
<PropertyGroup>
  <DefineConstants>$(DefineConstants);ROSLYN4_0_OR_GREATER</DefineConstants>
</PropertyGroup>
```

#### Running all band tests in CI

In CI, run all band-specific test projects as part of the same test step so a
failure in any band blocks the build:

```powershell
dotnet test tests/YourAnalyzer.Tests.Roslyn311/YourAnalyzer.Tests.Roslyn311.csproj -c Release
dotnet test tests/YourAnalyzer.Tests.Roslyn40/YourAnalyzer.Tests.Roslyn40.csproj -c Release
```

### Testing suppressors

`DiagnosticSuppressor` tests need a slightly different mindset from normal
analyzer tests. You are not asserting that a new diagnostic appears; you are
asserting that an existing compiler or analyzer diagnostic is suppressed only in
the cases that are actually safe.

For each suppressor, add at least:

- a case where the original diagnostic is present without the suppressor logic
  applying
- a case where the original diagnostic is present and the suppressor should
  suppress it
- a nearby non-suppression case that proves the suppressor is not over-broad
- an assertion of the exact suppressed diagnostic ID(s)

Guidance:

- keep suppression conditions narrow and test each precondition independently
- test the compiler/analyzer diagnostic that is being suppressed, not just the
  final absence of warnings
- include regression tests for every known false suppression and missed
  suppression

An incorrect suppressor is usually worse than a noisy analyzer because it can
hide a real problem without leaving any replacement diagnostic behind.

### What to test for every analyzer rule

For each diagnostic rule, include at least:

- **Positive case**: violation should report exactly expected diagnostic count.
- **Negative case**: compliant code should report nothing.
- **Boundary case**: nearest non-violation/violation edge.
- **Shape variants**: equivalent syntax forms (parentheses, casts,
  expression-bodied members, etc.).
- **Context variants**: async/non-async, generic/non-generic, local/member
  scope differences where relevant.

For rules targeting specific type or API patterns, also verify:

- all relevant type variations (generic, value type, and custom
  implementations)
- all relevant usage contexts (e.g., assigned, discarded, returned, or composed
  in expressions)

### Assertion best practices

Use precise assertions:

- assert exact diagnostic count when deterministic
- assert diagnostic ID
- assert stable message fragments where placeholders are expected
- assert location when location is part of rule contract

Prefer one behavior per test method.

### Test naming guidance

Keep behavior-oriented names using natural underscore-separated words, for
example:

- `Reports_diagnostic_for_bare_task_invocation`
- `Does_not_report_diagnostic_for_awaited_invocation`

Maintain symmetry between `Reports_*` and `Does_not_report_*` tests.

### Handling Roslyn API specifics in tests

Roslyn behavior can differ based on references and language version. Ensure
tests include:

- all required framework references for target constructs
- source snippets that compile cleanly for the intended syntax
- analyzer-specific dependencies (e.g., custom awaiter interfaces)

If a test unexpectedly returns no diagnostics, inspect:

- whether source compilation produced semantic model errors
- whether callback kind matches operation shape in that snippet
- whether metadata references include required types

### False-positive and false-negative strategy

For every new rule or modification:

- add at least one regression test for the original bug/requirement
- add one nearby non-violation test to prevent over-reporting
- add one variant syntax test to reduce tree-shape fragility

For every bug fix, keep the test that reproduces the bug permanently.

### Test matrix for configuration and environment differences

When analyzer behavior depends on environment or configuration, expand the test
matrix beyond the basic positive and negative cases. Consider testing:

- different reference assemblies or target frameworks when API shape affects
  semantic analysis
- nullable enabled vs disabled when null-state analysis influences the rule
- different C# language versions when syntax shape or binding behavior differs
- generated-code enabled vs disabled behavior
- valid and invalid custom configuration values from analyzer config options,
  MSBuild properties, and `AdditionalFiles`
- cancellation behavior for analyzers that perform heavier work or iterate
  large symbol sets

These tests help catch false positives that only appear under one host
configuration and reduce surprises when the analyzer is consumed across multiple
SDK or IDE versions.

### Running tests

From solution root:

```powershell
dotnet test YourProject.Tests/YourProject.Tests.csproj
```

Run targeted tests during development:

```powershell
dotnet test YourProject.Tests/YourProject.Tests.csproj --filter "FullyQualifiedName~YourAnalyzerTests"
```

### CI recommendations

In CI pipelines:

1. Restore packages
2. Build solution
3. Run analyzer tests
4. Fail pipeline on any test failure

Optional quality gates:

- collect test coverage
- enforce deterministic SDK via `global.json`
- run tests on multiple SDK patch versions if needed

### Validation against real-world repositories

Beyond automated unit tests, validate new or heavily-changed analyzers against
large real repositories to find false positives and real-world scalability
issues:
- Run against runtime-style repos (e.g.,
  [`dotnet/runtime`](https://github.com/dotnet/runtime) or large internal
  monorepos).
- Run against the Roslyn repo via
  [`AnalyzerRunner`](https://github.com/dotnet/roslyn-analyzers/tree/main/src/AnalyzerRunner).
- Verify diagnostics are actionable and the suppression rate is reasonable
  before shipping.

## 22. Debugging Analyzers

Analyzers run within the compiler process (`csc.exe` or `VBCSCompiler.exe`)
during builds, or inside the IDE process (`devenv.exe` /
`ServiceHub.RoslynCodeAnalysisService.exe`).

### How to debug:

1. **Visual Studio (VSIX / IDE Debugging)**: If you have a `.vsix` project in
   your solution, set it as the startup project. Press F5. An experimental
   instance of Visual Studio will launch with your analyzer installed.
   - Modern Visual Studio runs out-of-process analysis. Try attaching to
     `ServiceHub.RoslynCodeAnalysisService.exe` for analyzer execution or
     `devenv.exe` for code fixes.
   - Ensure the loaded binaries match the local build being debugged.
2. **VS Code / Rider**: You cannot easily spin up an "experimental instance"
   like Visual Studio's VSIX debugger. The most reliable way to debug across all
   IDEs is via **Unit Tests** (see below). Alternatively, you can use
   `Debugger.Launch()` during a command-line build.
3. **Debugger.Launch()**: For tricky build-time scenarios where `dotnet build`
   fails but IDE succeeds, temporarily add
   `System.Diagnostics.Debugger.Launch();` to your `Initialize` method to force
   a debugger prompt when MSBuild invokes the compiler. *Note: Remove this
   before committing!*
4. **Unit Tests (Recommended)**: The easiest and most cross-platform way to
   debug analyzer logic is by writing a unit test for the specific code pattern
   and repeatedly debugging the test itself. This approach avoids most
   host-process attachment issues in Visual Studio, VS Code, and Rider.

## 23. Pre-publish checklist

Before publishing an analyzer package, verify the following:

1. The analyzer assembly targets `netstandard2.0` unless a narrower audience is
   intentional and documented.
2. The chosen `Microsoft.CodeAnalysis` package version is the lowest minor that
   exposes the required APIs.
3. No RS10xx / RS20xx authoring diagnostics remain unresolved.
4. `AnalyzerReleases.Unshipped.md` and `AnalyzerReleases.Shipped.md` accurately
   reflect the package state.
5. The package layout contains only the intended analyzer assets and no
   duplicate unversioned/versioned copies.
6. Any `build/` or `buildTransitive/` targets were validated in a real
   consuming project.
7. The analyzer was validated in both CLI build and Visual Studio live analysis
   for at least one supported host/toolset combination.
8. Positive, negative, boundary, configuration-dependent, and
   suppressor-specific tests pass.
9. Any multi-Roslyn bands were each built and tested independently.
10. Documentation states the minimum supported Roslyn/SDK/Visual Studio
    requirements clearly.

## 24. Key takeaways

- Prefer the smallest viable API surface: `IOperation` or symbols first, syntax
  only when source shape matters.
- Treat diagnostic metadata and consumer configuration as part of the public
  contract of the rule.
- Choose precise diagnostic locations and stable code-fix equivalence keys.
- Keep configuration precedence explicit, documented, and tested.
- Test environment-sensitive behavior such as generated code, nullable context,
  target framework, and analyzer options.
- Prefer `IOperation` for semantic rules over syntax-level checks.
- Cache per-compilation state inside `CompilationStartAction`, not in static
  fields.
- Filter aggressively in callbacks to minimize allocations and wasted work.
- Analyzer exceptions are caught by the host (not truly silent — VS info-bar
  and `/reportanalyzer` surface them) but diagnostics are lost — never let them
  escape a callback.
- Pin to the lowest Roslyn version that supports your required APIs to maximize
  host compatibility.
- Avoid false positives: a rule that fires incorrectly erodes trust and gets
  suppressed.
- Test every rule with positive, negative, and boundary cases.
- When shipping Roslyn-versioned builds, keep `AssemblyName`, diagnostic IDs,
  and assembly versioning identical across all bands so consumer configurations
  remain valid regardless of which band is loaded.
- Do not place the same analyzer assembly in both a `roslyn{version}/cs/`
  folder and the unversioned `dotnet/cs/` folder unless you intentionally want
  both loaded simultaneously (additive loading). Use the lowest
  `roslyn{version}` folder as the fallback instead.
- Ship a `build/{PackageId}.targets` file to manually inject the fallback
  analyzer on older SDKs that do not set `$(SupportsRoslynComponentVersioning)`.
- Mirror the Roslyn-version split in both code-fix and test projects so that
  every band is built, fixed, and tested as a unit.

### Common authoring mistakes to avoid

- using syntax analysis when `IOperation` or symbol analysis would be simpler
  and more robust
- comparing symbols by reference equality instead of `SymbolEqualityComparer`
- reading `AdditionalFiles` or resolving expensive symbols inside per-node
  callbacks
- letting malformed configuration values throw exceptions instead of defaulting
  safely
- reporting diagnostics on broad declarations when a smaller, more actionable
  location exists
- using Fix All with overlapping or shared-file edits that require a custom
  merge strategy
- pinning Roslyn package versions higher than necessary for the intended host
  audience

## 25. Dataflow analysis framework

For complex semantic, security, or correctness rules, consider Roslyn's public
`ControlFlowGraph` APIs in `Microsoft.CodeAnalysis.FlowAnalysis` and, when the
dependency is acceptable, the higher-level dataflow utilities described in the
[Roslyn analyzer dataflow guidance](https://github.com/dotnet/roslyn-analyzers/blob/main/docs/Writing%20dataflow%20analysis%20based%20analyzers.md).
Types such as `PointsToAnalysis`, `ValueContentAnalysis`, and
`DataFlowOperationVisitor` come from the analyzer-utilities layer (for example
`Microsoft.CodeAnalysis.AnalyzerUtilities`), not from the base compiler package,
so packaging and host-compatibility implications must be handled like any other
analyzer dependency.

Key guidance:
- Start with existing well-known analyses when possible (for example
  `PointsToAnalysis`, `ValueContentAnalysis`, `CopyAnalysis`,
  `TaintedDataAnalysis`, `DisposeAnalysis`, or `PropertySetAnalysis`).
- Enable interprocedural analysis only when precision gains justify the
  performance cost.
- Bound interprocedural depth and expose configuration options so consumers can
  tune analysis depth.
- Use callsite predicates (`InterproceduralAnalysisPredicate`) to limit
  unnecessary interprocedural exploration.

## 26. Porting Legacy Rules (FxCop/Binary)

When modernizing or porting older rule sets to Roslyn (see [FxCop porting
guidance](https://github.com/dotnet/roslyn-analyzers/tree/main/docs/FxCopPort)):
- Factor rules into cohesive packages (e.g., API-surface specific vs
  theme-based packages).
- Prioritize high-value, low-noise rules first.
- Use historical telemetry and suppression signals as an input, but do not
  blindly port noisy rules without rethinking their heuristics.
- Explicitly document intentional rule behavior changes compared to the legacy
  implementation.

## 27. Consumer-side suppression

Consumers of an analyzer can suppress individual diagnostics through three
mechanisms. Each has different scope and persistence characteristics.

### `#pragma warning` (inline, per-file)

Suppresses a diagnostic for a specific span of source code:

```csharp
#pragma warning disable YOUR_RULE_ID // reason for suppression
var x = ProblematicCode();
#pragma warning restore YOUR_RULE_ID
```

- Affects only the enclosing scope between `disable` and `restore`.
- Visible in source history — the reason comment is important for
  maintainability.
- Applies to both IDE live analysis and `dotnet build`.

### `[SuppressMessage]` (attribute, per-member)

Suppresses a diagnostic on a specific declaration:

```csharp
[System.Diagnostics.CodeAnalysis.SuppressMessage(
    "Category",
    "YOUR_RULE_ID",
    Justification = "Reason why this specific case is safe.")]
public void MyMethod() { }
```

- Scoped to the decorated member.
- Requires `using System.Diagnostics.CodeAnalysis;` unless the attribute is fully
  qualified as shown above.
- Recorded in source — reviewers can see and challenge the justification.
- Applies to both IDE and `dotnet build`.

### `.editorconfig` severity override (project-wide or path-scoped)

Overrides the severity of a rule for all files matching the section pattern:

```ini
[*.cs]
dotnet_diagnostic.YOUR_RULE_ID.severity = none   # suppress everywhere

[**/*.Generated.cs]
dotnet_diagnostic.YOUR_RULE_ID.severity = none   # suppress only in generated files
```

- Centralised — one place to manage project-wide policy.
- Path-scoped sections allow different treatment for test code, generated code,
  etc.
- Applies consistently to both IDE and `dotnet build` when the `.editorconfig`
  is on the compiler's include path.
- Prefer this over `<NoWarn>` when the suppression is intentional policy rather
  than a temporary workaround.

### When to document suppression strategy in the rule

Every non-trivial analyzer rule should include a "When to suppress" section in
its documentation (see [Section 9](#9-diagnostic-design)) that explains:

- specific patterns where the diagnostic is a known false positive
- the preferred suppression mechanism for those patterns
- whether the rule provides a `DiagnosticSuppressor` to handle false positives
  automatically

This reduces noise from indiscriminate suppressions and helps consumers
understand the intent behind the rule.
