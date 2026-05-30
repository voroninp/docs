# Visual Studio Extension Development Guide

This document is a practical guide for building, testing, publishing, and maintaining Visual Studio extensions in the Visual Studio 2026 era. It focuses on the modern `VisualStudio.Extensibility` SDK, but it also covers the legacy Visual Studio SDK (`VSSDK`), the Community Visual Studio Toolkit, hybrid extensions, Marketplace publishing, licensing, trials, donations, diagnostics, and migration strategy.

Primary official sources used for this guide:

- Microsoft Learn, *About VisualStudio.Extensibility (Preview)*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/visualstudio-extensibility>
- Microsoft Learn, *Choose the right Visual Studio extensibility model for you*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/extensibility-models>
- Microsoft Learn, *Introduction to VisualStudio.Extensibility for VSSDK users*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/oop-extensibility-model-overview>
- Microsoft Learn, *Create your first Visual Studio extension*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/create-your-first-extension>
- Microsoft Learn, *Components of a VisualStudio.Extensibility extension*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/extension-anatomy>
- Microsoft Learn, *Rule-based activation constraints*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/activation-constraints>
- Microsoft Learn, *Why Remote UI*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/remote-ui>
- Microsoft Learn, *Debug a Visual Studio extension*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/debug-extensions>
- Microsoft Learn, *The Experimental Instance*: <https://learn.microsoft.com/visualstudio/extensibility/the-experimental-instance>
- Microsoft Learn, *Best practices checklist to publish a Visual Studio extension*: <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist>
- Microsoft Learn, *Walkthrough: Publish a Visual Studio extension*: <https://learn.microsoft.com/visualstudio/extensibility/walkthrough-publishing-a-visual-studio-extension>
- Microsoft Learn, *Walkthrough: Publishing a Visual Studio extension via command line*: <https://learn.microsoft.com/visualstudio/extensibility/walkthrough-publishing-a-visual-studio-extension-via-command-line>
- Microsoft Learn, *VSIX extension schema 2.0 reference*: <https://learn.microsoft.com/visualstudio/extensibility/vsix-extension-schema-2-0-reference>
- Microsoft Learn, *Prepare extensions for Windows Installer deployment*: <https://learn.microsoft.com/visualstudio/extensibility/preparing-extensions-for-windows-installer-deployment>
- Microsoft Learn, *Make extensions compatible with Visual Studio 2019/2017 and Visual Studio 2015*: <https://learn.microsoft.com/visualstudio/extensibility/how-to-roundtrip-vsixs>
- Microsoft Learn, *Supporting Multiple Versions of Visual Studio*: <https://learn.microsoft.com/visualstudio/extensibility/supporting-multiple-versions-of-visual-studio>
- Microsoft Learn, *Update a Visual Studio extension*: <https://learn.microsoft.com/visualstudio/extensibility/how-to-update-a-visual-studio-extension>
- Microsoft Learn, *Extension compatibility model for Visual Studio*: <https://learn.microsoft.com/visualstudio/extensibility/migration/extension-compatibility>
- Microsoft Learn, *Use AsyncPackage to load VSPackages in the background*: <https://learn.microsoft.com/visualstudio/extensibility/how-to-use-asyncpackage-to-load-vspackages-in-the-background>
- Microsoft Learn, *Manage multiple threads in managed code*: <https://learn.microsoft.com/visualstudio/extensibility/managing-multiple-threads-in-managed-code>
- Microsoft Learn, *Registering VSPackages*: <https://learn.microsoft.com/visualstudio/extensibility/internals/registering-vspackages>
- Microsoft Learn, *CreatePkgDef utility*: <https://learn.microsoft.com/visualstudio/extensibility/internals/createpkgdef-utility>
- Microsoft Learn, *Specifying VSPackage File Location to the VS Shell*: <https://learn.microsoft.com/visualstudio/extensibility/internals/specifying-vspackage-file-location-to-the-vs-shell>
- Microsoft Learn, *VisualStudio.Extensibility Diagnostics Explorer*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/diagnostics/visualstudio-extensibility-diagnostics-extension>
- Microsoft Learn, *Logging extension diagnostics*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/logging>
- Microsoft Learn, *Find, install, and manage extensions for Visual Studio*: <https://learn.microsoft.com/visualstudio/ide/finding-and-using-visual-studio-extensions>
- Microsoft DevBlogs, *Visual Studio extensions and version ranges demystified*: <https://devblogs.microsoft.com/visualstudio/visual-studio-extensions-and-version-ranges-demystified/>
- VSExtensibility repository announcements, breaking changes, known issues, and samples: <https://github.com/microsoft/VSExtensibility>

## Table of contents

- [Quick start checklist](#quick-start-checklist)
- [1. Mental model](#1-mental-model)
- [2. Choosing an extensibility model](#2-choosing-an-extensibility-model)
- [3. Architecture and history](#3-architecture-and-history)
- [4. VisualStudio.Extensibility project setup](#4-visualstudioextensibility-project-setup)
- [5. Extension anatomy](#5-extension-anatomy)
- [6. Commands](#6-commands)
- [7. Activation constraints](#7-activation-constraints)
- [8. Remote UI, dialogs, prompts, and tool windows](#8-remote-ui-dialogs-prompts-and-tool-windows)
- [9. Editor, document, project, and language features](#9-editor-document-project-and-language-features)
- [10. VSSDK and Community Toolkit](#10-vssdk-and-community-toolkit)
- [11. Visual Studio extension MSBuild properties and `.pkgdef` registration](#11-visual-studio-extension-msbuild-properties-and-pkgdef-registration)
- [12. Threading, async, performance, and reliability](#12-threading-async-performance-and-reliability)
- [13. Configuration, settings, and state](#13-configuration-settings-and-state)
- [14. Testing strategy](#14-testing-strategy)
- [15. Debugging and diagnostics](#15-debugging-and-diagnostics)
- [16. Packaging and VSIX metadata](#16-packaging-and-vsix-metadata)
- [17. Publishing and deployment](#17-publishing-and-deployment)
- [18. Licensing, trials, paid versions, sponsorship, and donations](#18-licensing-trials-paid-versions-sponsorship-and-donations)
- [19. Security, privacy, and governance](#19-security-privacy-and-governance)
- [20. Versioning and compatibility](#20-versioning-and-compatibility)
- [21. Migration and hybrid architecture](#21-migration-and-hybrid-architecture)
- [22. Future direction and acknowledged pain points](#22-future-direction-and-acknowledged-pain-points)
- [23. Typical misuses and antipatterns](#23-typical-misuses-and-antipatterns)
- [24. Troubleshooting](#24-troubleshooting)
- [25. Pre-publish checklist](#25-pre-publish-checklist)
- [26. Key takeaways](#26-key-takeaways)

## Quick start checklist

When creating a new Visual Studio extension in 2026, verify these decisions first:

- Prefer `VisualStudio.Extensibility` for new extensions when the API surface covers the scenario. It runs out-of-process, targets modern .NET, uses asynchronous APIs, isolates extension failures from `devenv.exe`, and enables hot-loading without a Visual Studio restart.
- Use VSSDK or the Community Visual Studio Toolkit only when you need an API that is not yet available in `VisualStudio.Extensibility`, or when you must support older Visual Studio versions that cannot load the new model.
- For a hybrid extension, isolate shared business logic in normal class libraries and keep Visual Studio integration code thin.
- Install the **Visual Studio extension development** workload before creating or building extension projects.
- Treat `VisualStudio.Extensibility` as the strategic direction, but remember that Microsoft Learn still labels it as preview and points to the VSExtensibility GitHub repository for latest announcements, known issues, and breaking changes.
- Keep extension startup lazy. Surface commands and UI only when context rules say they are relevant.
- Use rule-based activation constraints instead of eager package loading.
- For out-of-process UI, use Remote UI and MVVM-style binding; do not expect arbitrary in-process WPF controls, code-behind, or custom controls to work.
- For VSSDK packages, derive from `AsyncPackage`, set `AllowsBackgroundLoading = true`, and use async service retrieval wherever possible.
- Add `Microsoft.VisualStudio.SDK.Analyzers` to VSSDK projects to catch common threading and extensibility mistakes.
- Always test in the Visual Studio Experimental Instance before installing into the main development instance.
- In Visual Studio 2026 and later, use **Extensions** > **Extension Development** to start or reset the Experimental Instance when available.
- Validate both debug-time and installed VSIX behavior. F5 deployment is not a replacement for testing the actual packaged extension.
- Publish with a clear icon, short name, accurate description, screenshots, license, privacy notice, support link, and conservative Visual Studio version ranges.
- Avoid open-ended Visual Studio version ranges unless you have a deliberate servicing process for future versions.
- If the extension contacts any remote service, collects telemetry, checks a license, or opens sponsor/payment pages, disclose that behavior in the Marketplace description and privacy policy.

## 1. Mental model

A Visual Studio extension is code plus metadata that Visual Studio discovers, loads, and invokes when the user performs an IDE action or when a relevant IDE context becomes active. Extensions can add commands, menus, tool windows, editor features, language services, debugger visualizers, project integration, templates, options, settings, and other IDE behavior.

Modern Visual Studio extensibility has two major worlds:

1. **In-process extensibility** — the traditional VSSDK model. Extension code runs inside `devenv.exe` and can use the deep Visual Studio object model, COM-based services, `DTE`, MEF editor components, project-system APIs, and shell services. It is powerful but risky because extension bugs can freeze or crash the IDE.
2. **Out-of-process extensibility** — the `VisualStudio.Extensibility` model. Extension code runs in a dedicated ServiceHub process and communicates with Visual Studio through RPC-compatible brokered services. The IDE can discover extension contributions from metadata without loading the extension assembly until an activation condition is met.

The high-level trade-off is straightforward:

| Need | Prefer |
|---|---|
| New command, editor interaction, prompt, dialog, tool window, debugger visualizer, settings, project queries, or language-server provider supported by the new SDK | `VisualStudio.Extensibility` out-of-process |
| Deep legacy shell APIs, unsupported VSSDK service, older Visual Studio support, existing large VSIX with many VSSDK dependencies | VSSDK or Community Toolkit |
| New extension needing one unsupported legacy API | `VisualStudio.Extensibility` in-process or a hybrid VSIX |
| Maximum IDE reliability and ability to target modern .NET | `VisualStudio.Extensibility` out-of-process |
| Maximum API breadth today | VSSDK |

Official reference: Microsoft Learn, *Choose the right Visual Studio extensibility model for you*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/extensibility-models>.

## 2. Choosing an extensibility model

Visual Studio currently has three practical authoring models.

### VisualStudio.Extensibility

`VisualStudio.Extensibility` is the new model. It is designed around:

- out-of-process hosting;
- modern .NET;
- async APIs end-to-end;
- dependency injection;
- metadata-driven contributions;
- hot-loading where possible;
- better reliability when an extension hangs or crashes.

Microsoft Learn explicitly states that the new model was created to address problems in the old model: extension-caused crashes and hangs, inconsistent and out-of-date APIs/docs, overwhelming architecture, restart-required installs, and no .NET Core support. See *Introduction to VisualStudio.Extensibility for VSSDK users*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/oop-extensibility-model-overview>.

Use it when:

- you are starting a new extension;
- you only need to support Visual Studio versions that include the new model;
- the required feature area exists in the new SDK;
- reliability and responsiveness are product requirements;
- your extension needs modern .NET libraries;
- you want users to install without restarting Visual Studio when possible.

Do not assume all VSSDK features are available yet. Microsoft Learn says the new model is in active development and not yet as broad as VSSDK. When a needed feature is missing, use in-process `VisualStudio.Extensibility` or VSSDK as a bridge.

### VSSDK

VSSDK is the traditional model and the foundation for most long-lived Visual Studio extensions. It is also what Visual Studio itself historically used for many internal extension points.

Strengths:

- deepest API coverage;
- mature samples and ecosystem;
- supports older Visual Studio versions;
- can reach shell, editor, project system, debugger, COM, DTE, and MEF APIs.

Costs:

- extension runs in `devenv.exe`;
- targets .NET Framework because Visual Studio's in-process shell is based on .NET Framework;
- extension bugs can freeze or crash Visual Studio;
- many APIs require main-thread affinity;
- command definitions often use `.vsct` files;
- APIs reflect decades of Visual Studio evolution and include COM, DTE, MEF, shell services, and project-system concepts.

Use VSSDK when the new SDK cannot express your scenario, or when support for older Visual Studio versions is mandatory.

### Community Visual Studio Toolkit

The Community Toolkit wraps VSSDK to provide a friendlier authoring experience. It can be a good choice for smaller VSSDK-based extensions, especially when onboarding developers who do not need to learn every shell detail immediately.

However, the Toolkit is still built on VSSDK. It does not remove the fundamental limitations of in-process extension hosting. Microsoft Learn calls out a specific risk: a simpler wrapper can give a false sense of simplicity and hide threading requirements that still matter underneath. See the comparison section in *Choose the right Visual Studio extensibility model for you*: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/extensibility-models#comparing-the-different-visual-studio-extensibility-models>.

### Decision matrix

| Scenario | Recommendation |
|---|---|
| New extension for VS 2026 with supported command/editor/tool-window APIs | Use `VisualStudio.Extensibility` out-of-process. |
| New extension with one unsupported legacy shell operation | Start with `VisualStudio.Extensibility`; try a brokered service first, then isolate the legacy call in an in-process bridge only if needed. |
| Existing VSIX that targets VS 2019 and VS 2022/2026 | Keep VSSDK for down-level support; split common business logic into shared libraries; consider a separate new-model VSIX for newer VS versions. |
| Existing extension that only supports modern Visual Studio and uses APIs now covered by the new SDK | Rewrite or gradually migrate to `VisualStudio.Extensibility`. |
| Extension is primarily a Roslyn analyzer/source generator | Package as NuGet analyzer assets, not as a VSIX, unless you also need IDE UI. |
| Extension depends on arbitrary WPF code-behind inside a tool window | VSSDK is simpler today; for new SDK, redesign around Remote UI and MVVM constraints. |

## 3. Architecture and history

### Why old Visual Studio extensibility looks the way it does

Visual Studio has decades of compatibility history. Its extension surface grew through several eras:

- COM services and package registration;
- automation through `EnvDTE`;
- shell services and `IVs*` interfaces;
- MEF-based editor and language-service components;
- VSIX packaging and Marketplace distribution;
- `AsyncPackage` and background loading to reduce startup and solution-load cost;
- out-of-process services and ServiceHub-based architecture.

This evolution explains why a VSSDK extension often mixes concepts that feel unrelated: `.vsct` command tables, GUIDs, COM service querying, `DTE`, MEF exports, `AsyncPackage`, `JoinableTaskFactory`, and `.vsixmanifest` metadata. These APIs exist because Visual Studio has had to preserve compatibility while modernizing the IDE.

Microsoft Learn explicitly acknowledges that VSSDK APIs were aggregated over the years as Visual Studio transformed and evolved, and that an extension might need to work with COM-based APIs, DTE, and MEF in one codebase. See: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/extensibility-models#vssdk>.

### Why `AsyncPackage` exists

Loading and initializing packages can perform disk I/O and service discovery. When that happens on the UI thread, Visual Studio can become unresponsive. Visual Studio 2015 introduced `AsyncPackage` to let packages load on a background thread. Official guidance says to derive from `AsyncPackage`, set `PackageRegistration(UseManagedResourcesOnly = true, AllowsBackgroundLoading = true)`, mark async-queryable services with `IsAsyncQueryable = true`, and use `PackageAutoLoadFlags.BackgroundLoad` for UI-context autoloading. See: <https://learn.microsoft.com/visualstudio/extensibility/how-to-use-asyncpackage-to-load-vspackages-in-the-background>.

`AsyncPackage` is a mitigation for the old architecture. It improves responsiveness but does not isolate extension code from the Visual Studio process.

### Why VisualStudio.Extensibility moved out-of-process

The new model moves extension code out of `devenv.exe` into a ServiceHub host process. Microsoft Learn describes the technology as RPC contracts provided as brokered services from Visual Studio. The extension host communicates with Visual Studio via RPC, consuming IDE services and sometimes providing services back to Visual Studio.

This architecture has several consequences:

- an extension crash should not crash Visual Studio;
- extension CPU work is less likely to block the UI thread;
- extension code can target modern .NET;
- APIs must be asynchronous because many operations cross process boundaries;
- arbitrary object references cannot be shared directly;
- UI must be represented by serializable data, XAML, and commands rather than arbitrary in-process object graphs;
- Visual Studio can read contribution metadata before loading the extension.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/oop-extensibility-model-overview#technology>.

### Metadata-first activation

In `VisualStudio.Extensibility`, extension parts are marked with `[VisualStudioContribution]`. Configuration properties are evaluated during build and saved as extension metadata. Visual Studio can then know that a command, tool window, listener, or visualizer exists without loading the extension assembly immediately.

That design matters because extension loading is no longer the mechanism used to discover every possible contribution. The IDE can track activation points, then load the extension only when the user invokes the feature or when an activation constraint becomes true.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/create-your-first-extension>.

## 4. VisualStudio.Extensibility project setup

### Prerequisites

Install Visual Studio 2026 with the **Visual Studio extension development** workload. Microsoft Learn currently states that the `VisualStudio.Extensibility` preview works with Visual Studio 2022 version 17.9 Preview 1 or higher, but the same documentation includes Visual Studio 2026-specific Experimental Instance menu guidance. Treat the public docs and the VSExtensibility GitHub repository as the source of truth for the current supported build.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/visualstudio-extensibility#install-visualstudioextensibility>
- <https://github.com/microsoft/VSExtensibility>

### NuGet packages and SDK references

The new model is delivered as a set of NuGet packages that provide the APIs, build tooling, code generation, and analyzers. Reference the meta-package and let it pull in the rest rather than referencing the individual packages directly.

| Package | Role | Notes |
|---|---|---|
| `Microsoft.VisualStudio.Extensibility.Sdk` | Primary meta-package | Reference this from every new-model extension. It carries dependencies on the prerequisite packages below, so you normally do not reference them explicitly. |
| `Microsoft.VisualStudio.Extensibility.Build` | Build tooling and project-capability code generators | Required for the build and for F5 debugging in the Visual Studio IDE. |
| `Microsoft.VisualStudio.Extensibility` | SDK APIs and utility libraries | The out-of-process API surface; pulled in transitively by the SDK meta-package. |
| `Microsoft.VisualStudio.Extensibility.JsonGenerators.Sdk` | Metadata code generators | Generates the contribution metadata emitted at build time. Without it a compiled extension may not work because the metadata files are missing. |
| `Microsoft.VisualStudio.Sdk` | VSSDK meta-package | Add only for in-process/hybrid scenarios that must consume VSSDK or MEF services (see Section 10 and Section 21). |

Reference both new-model packages with `PrivateAssets="all"` so they do not flow to consumers of the project:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.VisualStudio.Extensibility.Sdk" Version="17.*" PrivateAssets="all" />
  <PackageReference Include="Microsoft.VisualStudio.Extensibility.Build" Version="17.*" PrivateAssets="all" />
</ItemGroup>
```

Microsoft may ship optional feature-area packages (for example debugger or source-control APIs) that are not pulled in by the meta-package; add those only when you use the corresponding feature.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/inside-the-sdk#nuget-packages>
- <https://www.nuget.org/packages/Microsoft.VisualStudio.Extensibility.Sdk/>

### Recommended repository layout

For a medium-to-large extension, prefer a split like this:

```plaintext
src/
  MyExtension/
    MyExtension.csproj                # VisualStudio.Extensibility VSIX
  MyExtension.LegacyBridge/
    MyExtension.LegacyBridge.csproj   # optional VSSDK/in-proc bridge
  MyExtension.Core/
    MyExtension.Core.csproj           # extension-independent business logic
  MyExtension.Tests/
    MyExtension.Tests.csproj          # unit tests for core and models
  MyExtension.IntegrationTests/
    MyExtension.IntegrationTests.csproj
docs/
  marketplace-overview.md
  privacy.md
  license.md
```

Reasons:

- core behavior can be unit tested without launching Visual Studio;
- Visual Studio APIs stay at the edge;
- licensing, networking, telemetry, and update logic can be reused across extension models;
- migration from VSSDK to the new model becomes incremental.

### Minimal extension class

The template creates a class derived from `Extension`. This class is the entry point for the extension and a place to register dependency-injection services:

```csharp
[VisualStudioContribution]
internal sealed class ExtensionEntrypoint : Extension
{
    protected override void InitializeServices(IServiceCollection serviceCollection)
    {
        base.InitializeServices(serviceCollection);

        serviceCollection.AddSingleton<MySharedService>();
    }
}
```

Project-specific rules from this repository still apply to extension code: prefer `internal sealed` classes, least exposure, `var`, explicit non-target typed `new`, and cancellation tokens with `default` where appropriate.

### Extension metadata

The `ExtensionConfiguration.Metadata` value is used to generate VSIX metadata. Keep these values stable:

- extension ID / VSIX ID;
- publisher name;
- display name;
- version;
- description.

Changing the VSIX ID breaks update identity. Marketplace publishing docs state that the VSIX ID is the unique identifier Visual Studio uses for the extension and is required for auto-update behavior. See: <https://learn.microsoft.com/visualstudio/extensibility/walkthrough-publishing-a-visual-studio-extension#publish-the-extension-to-visual-studio-marketplace>.

### Localization implementation notes

Localization quality is an operational concern, not just translation coverage.
For extension UI strings (commands, prompts, settings, progress messages,
diagnostics shown to users):

- centralize user-visible strings in resource files and reference keys from
  metadata (`%...%`) where supported;
- use stable, descriptive resource keys (`Command.OpenDocumentation.DisplayName`)
  so keys survive refactoring;
- include both short labels and longer descriptions where the API supports both
  (for example setting `DisplayName` + `Description`);
- avoid concatenating sentence fragments in code, which harms translation
  quality and grammar in many languages;
- localize error/help links and docs entry points when region-specific content
  exists, otherwise keep one canonical URL;
- add a localization smoke test pass (switch VS UI language, verify command
  labels, settings descriptions, and help links).

## 5. Extension anatomy

A `VisualStudio.Extensibility` extension has four important concepts.

### Extension instance

The `Extension` instance is the starting point for extension execution. It provides configuration, localized resources, and dependency-injection services shared by extension parts.

### VisualStudioExtensibility object

`VisualStudioExtensibility` is the object used to reach extensibility feature areas exposed by Visual Studio. Feature-specific extension methods provide access to shell, editor, output window, project, and other services.

### Extension parts

Extension parts are classes marked with `[VisualStudioContribution]`, such as:

- command handlers;
- tool windows;
- text view opened/closed listeners;
- text view change listeners;
- margin providers;
- debugger visualizers.

Visual Studio discovers them from generated metadata, and the SDK creates instances with dependency injection.

### Client context

`IClientContext` represents a snapshot of relevant IDE state at the time a command or callback is invoked. Because out-of-process communication is asynchronous, this state can be stale by the time your code awaits and resumes. Use it as an input snapshot, not as a live mutable view of the IDE.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/extension-anatomy>.

## 6. Commands

Commands are the most common extension entry point.

In the new model:

- create a class deriving from `Command`;
- mark it with `[VisualStudioContribution]`;
- define `CommandConfiguration` in code instead of a `.vsct` file;
- implement `ExecuteCommandAsync`;
- use `IClientContext` and injected services to do work.

Example shape:

```csharp
[VisualStudioContribution]
internal sealed class OpenDocumentationCommand : Command
{
    public override CommandConfiguration CommandConfiguration => new("%OpenDocumentation.DisplayName%")
    {
        Icon = new(ImageMoniker.KnownValues.Extension, IconSettings.IconAndText),
        Placements = [CommandPlacement.KnownPlacements.ExtensionsMenu],
        EnabledWhen = ActivationConstraint.SolutionState(SolutionState.Exists),
    };

    public override async Task ExecuteCommandAsync(
        IClientContext context,
        CancellationToken cancellationToken)
    {
        await context.ShowPromptAsync(
            "Open documentation from the extension.",
            PromptOptions.OK,
            cancellationToken);
    }
}
```

Guidelines:

- keep command handlers thin;
- move business logic to injected services;
- honor `CancellationToken`;
- do not perform long synchronous file/network operations inside command callbacks;
- prefer activation constraints over checking context late;
- localize display names through string resources;
- avoid adding top-level menus; Microsoft publish guidance says not to add a new menu next to File/Edit/etc.

### Command placement and discoverability checklist

When adding commands, treat placement as a UX decision, not just a technical one.
A practical placement rubric:

- put commands in the **closest user workflow context** first (editor context
  menu for editor actions, project/solution context menu for project actions);
- add an **Extensions menu** fallback only if discoverability would otherwise be
  poor;
- reserve toolbars for high-frequency actions that users repeatedly invoke in a
  session;
- avoid duplicating the same command in many places unless each placement serves
  a different entry flow;
- use command text that starts with a clear verb and scope (for example,
  `Generate API Client` vs `Run`);
- if a command is only valid in specific contexts, hide/disable it via
  activation constraints (new model) or visibility constraints/UI contexts
  (VSSDK) instead of showing a command that fails at execution time.

Quick acceptance checks before shipping:

1. Can a new user find the command where they naturally expect it?
2. Is the command absent from irrelevant contexts?
3. Does disabled state explain *why* it is disabled when possible?
4. Is there exactly one primary place to invoke it?

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/create-your-first-extension>
- <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist#make-it-feel-native-to-vs>

### VSSDK command definition with `.vsct` files

In the new model, command metadata (display name, placement, icon, visibility,
enabled state) lives in code as `CommandConfiguration`. In VSSDK and hybrid
extensions, that same metadata is authored declaratively in an XML **command
table** file, a `.vsct` file. A `.vsct` file describes the layout and appearance
of command items for a VSPackage: buttons, combo boxes, menus, toolbars, and
groups of command items. It is the VSSDK counterpart to the new model's
metadata-driven contributions, and you still need it whenever you ship an
in-process VSPackage that adds UI.

Three building blocks underpin every command table:

- **Commands** are the procedures the user can invoke (exposed as menu items,
  buttons, list boxes, or other controls).
- **Groups** are containers that hold commands and menus.
- **Menus** are the containers shown in the UI (menus, submenus, toolbars, or
  tool windows).

The structural rules that the build enforces:

- a command must live in a group; a group must live in a menu; a submenu must be
  placed in a group, not directly in a menu;
- only menus are actually displayed; groups and commands are not displayed on
  their own;
- every `Menu`, `Group`, and `Button` is identified by a `GUID:ID` pair, and
  each pair must be unique;
- an item can appear in more than one location by reusing it through a
  `CommandPlacement` rather than by giving it multiple parents.

The root element is `CommandTable`, which contains a `Commands` element (whose
`Package` attribute names the owning package) and a `Symbols` element. The most
common child elements are:

| Element | Purpose |
|---|---|
| `Menus` | Defines menus, submenus, context menus, and toolbars |
| `Groups` | Defines command groups (containers) |
| `Buttons` | Defines command buttons / menu items and binds them to handlers |
| `Bitmaps` | Declares icon images used by buttons |
| `CommandPlacements` | Places an existing command/group/menu in additional locations |
| `VisibilityConstraints` | Shows a command/menu only in a specified UI context |
| `KeyBindings` | Assigns keyboard shortcuts to commands |
| `UsedCommands` | Declares that this package implements a command defined elsewhere |
| `Extern` / `Include` | References Visual Studio header files (for example `stdidcmd.h`, `vsshlids.h`) so you can parent your UI under IDE-defined menus and groups |
| `Symbols` | Maps friendly names to the GUIDs and IDs used throughout the file |

The `Symbols` section typically declares three `GuidSymbol` elements: one for the
package GUID, one for the command set (all menus, groups, and commands), and one
for the image/bitmap store. Each `GuidSymbol` contains `IDSymbol` elements that
name the individual menus, groups, commands, and icons; no two `IDSymbol` values
under the same `GuidSymbol` may collide. Visual Studio templates generate the
package and command-set GUIDs automatically.

Minimal shape:

```xml
<CommandTable xmlns="http://schemas.microsoft.com/VisualStudio/2005-10-18/CommandTable"
              xmlns:xs="http://www.w3.org/2001/XMLSchema">
  <Extern href="stdidcmd.h" />
  <Extern href="vsshlids.h" />

  <Commands Package="guidMyPackage">
    <Groups>
      <Group guid="guidMyCommandSet" id="MyMenuGroup" priority="0x0600">
        <Parent guid="guidSHLMainMenu" id="IDM_VS_MENU_TOOLS" />
      </Group>
    </Groups>

    <Buttons>
      <Button guid="guidMyCommandSet" id="MyCommandId" priority="0x0100" type="Button">
        <Parent guid="guidMyCommandSet" id="MyMenuGroup" />
        <Icon guid="guidImages" id="bmpPic1" />
        <Strings>
          <ButtonText>My Command</ButtonText>
        </Strings>
      </Button>
    </Buttons>
  </Commands>

  <Symbols>
    <GuidSymbol name="guidMyPackage" value="{...}" />
    <GuidSymbol name="guidMyCommandSet" value="{...}">
      <IDSymbol name="MyMenuGroup" value="0x1020" />
      <IDSymbol name="MyCommandId" value="0x0100" />
    </GuidSymbol>
    <GuidSymbol name="guidImages" value="{...}">
      <IDSymbol name="bmpPic1" value="1" />
    </GuidSymbol>
  </Symbols>
</CommandTable>
```

Guidelines for `.vsct` authoring:

- keep the package, command-set, and image-store GUIDs stable; the `GUID:ID`
  pair is the identity Visual Studio and your `OleMenuCommandService`/
  `IOleCommandTarget` code uses to route a command;
- parent your groups under IDE-defined menus/groups via `Extern` references
  rather than inventing new top-level menus, consistent with the publish
  checklist guidance to feel native to Visual Studio;
- prefer `VisibilityConstraints` (UI context rules) over loading the package
  just to hide a command, the VSSDK equivalent of the new model's activation
  constraints;
- match each `Button` ID to the command ID your package wires up in code so the
  handler is invoked;
- the `.vsct` is compiled by the VSSDK build tools into the package's
  command-table resource; the resulting registration is referenced by the
  package's generated `.pkgdef` (see Section 11).

Pure out-of-process `VisualStudio.Extensibility` extensions do not use `.vsct`
files; their command metadata is generated from `CommandConfiguration`. Use a
`.vsct` only for the in-process VSPackage portion of a VSSDK or hybrid extension.

Official references:

- Design XML command table (`.vsct`) files: <https://learn.microsoft.com/visualstudio/extensibility/internals/designing-xml-command-table-dot-vsct-files>
- How VSPackages add user interface elements: <https://learn.microsoft.com/visualstudio/extensibility/internals/how-vspackages-add-user-interface-elements>
- Author `.vsct` files: <https://learn.microsoft.com/visualstudio/extensibility/internals/authoring-dot-vsct-files>
- VSCT XML schema reference: <https://learn.microsoft.com/visualstudio/extensibility/vsct-xml-schema-reference>
- IDE-defined commands, menus, and groups: <https://learn.microsoft.com/visualstudio/extensibility/internals/ide-defined-commands-menus-and-groups>

## 7. Activation constraints

Activation constraints define when an extension loads, when a command is visible, and when a command is enabled. They are essential for performance and for a native user experience.

Examples of supported constraint concepts include:

- client context keys, such as active selection file name;
- active project capability;
- project-added item pattern;
- solution has project capability;
- solution state;
- editor content type;
- legacy UI context GUIDs where needed.

Example:

```csharp
EnabledWhen =
    ActivationConstraint.SolutionState(SolutionState.Exists) &
    ActivationConstraint.ClientContext(
        ClientContextKey.Shell.ActiveEditorFileName,
        @"\.(md|markdown)$");
```

Practical guidance:

- make commands visible only when they apply;
- make commands enabled only when execution can succeed;
- avoid loading the extension just to decide whether UI should be hidden;
- prefer project capabilities over hard-coded project type GUIDs where possible;
- use the Diagnostics Explorer client context page to discover available state for activation rules.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/activation-constraints>
- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/diagnostics/visualstudio-extensibility-diagnostics-extension>

## 8. Remote UI, dialogs, prompts, and tool windows

### Why Remote UI exists

Out-of-process extensions cannot directly place arbitrary WPF objects into the Visual Studio process. Microsoft created Remote UI so an out-of-process extension can define WPF-based UI that Visual Studio renders in-process while state and commands are proxied across the process boundary.

The official Remote UI documentation explains the architectural constraint: most UI frameworks are in-process, while the new model deliberately runs extensions outside the IDE process. See: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/remote-ui>.

### Remote UI rules

Remote UI differs from ordinary WPF:

- operations are asynchronous;
- data context types must be serializable by Remote UI;
- data context members used by binding require `DataContract` and `DataMember`;
- custom controls from the extension assembly are not supported;
- no code-behind or event handlers;
- communicate from UI to extension through async commands and binding;
- the XAML is instantiated in the Visual Studio process, not in the extension host process.

#### Serializable types in a Remote UI data context

Only data the Remote UI infrastructure can replicate into the proxy living in the Visual Studio process can be bound. The serializable set is:

- primitive data (most .NET numeric types, enums, `bool`, `string`, `DateTime`);
- extension-defined types marked with `[DataContract]` whose `[DataMember]` properties are themselves serializable;
- objects implementing `IAsyncCommand` (surfaced as `ICommand` in the Visual Studio process);
- `XamlFragment`, `SolidColorBrush`, and `Color` values;
- `Nullable<>` of a serializable type;
- collections of serializable types, including observable collections (`ObservableList<T>`).

### Remote UI worked example

A *remote user control* is defined across four files: a command that opens the tool window, the `ToolWindow`, the `RemoteUserControl`, and its XAML. The control automatically loads the embedded XAML resource that shares its type name.

Data context (the MVVM *ViewModel*). Extend `NotifyPropertyChangedObject` to get `SetProperty`/`INotifyPropertyChanged` for observable members, and expose user actions as `AsyncCommand` rather than event handlers:

```csharp
using System.Runtime.Serialization;
using Microsoft.VisualStudio.Extensibility.UI;

[DataContract]
internal sealed class MyToolWindowData : NotifyPropertyChangedObject
{
    public MyToolWindowData()
    {
        HelloCommand = new((parameter, cancellationToken) =>
        {
            Text = $"Hello {Name}!";
            return Task.CompletedTask;
        });
    }

    private string _name = string.Empty;

    [DataMember]
    public string Name
    {
        get => _name;
        set => SetProperty(ref _name, value);
    }

    private string _text = string.Empty;

    [DataMember]
    public string Text
    {
        get => _text;
        set => SetProperty(ref _text, value);
    }

    [DataMember]
    public AsyncCommand HelloCommand { get; }
}
```

Remote user control. The data context is passed to the base constructor (the `DataContext` property is read-only; the root object cannot be swapped, though its members can change). Use `ControlLoadedAsync` for initialization that depends on the UI being attached, and override `Dispose` for cleanup:

```csharp
using Microsoft.VisualStudio.Extensibility.UI;

internal sealed class MyToolWindowContent : RemoteUserControl
{
    public MyToolWindowContent()
        : base(dataContext: new MyToolWindowData())
    {
    }

    public override async Task ControlLoadedAsync(CancellationToken cancellationToken)
    {
        await base.ControlLoadedAsync(cancellationToken);
        // Initialize data that can change independently of UI events here.
    }
}
```

XAML. A remote user control is a single `DataTemplate` using the Remote UI namespace. It is normal WPF XAML with no code-behind; the `IAsyncCommand` is bound as a regular `ICommand`:

```xml
<DataTemplate xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
              xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
              xmlns:vs="http://schemas.microsoft.com/visualstudio/extensibility/2022/xaml">
    <StackPanel Orientation="Vertical">
        <TextBox Text="{Binding Name, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}" />
        <Button Content="Say hello" Command="{Binding HelloCommand}" />
        <TextBlock Text="{Binding Text}" />
    </StackPanel>
</DataTemplate>
```

Mark the XAML as an embedded resource (not a WPF `Page`) so Remote UI can load it:

```xml
<ItemGroup>
  <EmbeddedResource Include="MyToolWindowContent.xaml" />
  <Page Remove="MyToolWindowContent.xaml" />
</ItemGroup>
```

Tool window. Return the control from `GetContentAsync`; a separate command opens it via `ShellExtensibility.ShowToolWindowAsync`:

```csharp
[VisualStudioContribution]
internal sealed class MyToolWindow : ToolWindow
{
    private readonly MyToolWindowContent _content = new MyToolWindowContent();

    public MyToolWindow(VisualStudioExtensibility extensibility)
        : base(extensibility)
    {
        Title = "My Tool Window";
    }

    public override ToolWindowConfiguration ToolWindowConfiguration => new()
    {
        Placement = ToolWindowPlacement.DocumentWell,
    };

    public override Task<IRemoteUserControl> GetContentAsync(CancellationToken cancellationToken)
    {
        return Task.FromResult<IRemoteUserControl>(_content);
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing)
        {
            _content.Dispose();
        }

        base.Dispose(disposing);
    }
}
```

The end-to-end flow: the control binds to a *proxy* of the data context inside the Visual Studio process; typing updates `Name` on the proxy, which propagates to `MyToolWindowData` in the extension host; clicking the button executes `HelloCommand` asynchronously in the host, which updates the observable `Text`, which propagates back to the proxy and refreshes the bound `TextBlock`. Because every hop is asynchronous and crosses a process boundary, prefer capturing values via command parameters at click time over assuming a shared object graph updates instantly.

For ownership rules with dialogs (`ShowDialogAsync` transfers ownership of the control to Visual Studio, so the extension must not dispose it), context menus, and images, see the advanced Remote UI references below.

Official references:

- Why Remote UI (tutorial): <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/remote-ui>
- Other Remote UI concepts (context menus, images): <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/other-remote-ui>
- Tool windows: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/tool-window/tool-window>
- Dialogs: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/dialog/dialog>

### Tool windows

Tool windows are dockable Visual Studio windows. In the new model, implement a `ToolWindow`, return a `RemoteUserControl`, and expose a command that calls `ShellExtensibility.ShowToolWindowAsync`.

Guidelines:

- dispose the remote user control when the tool window is disposed;
- use Visual Studio theme resources so the UI matches dark/light/high-contrast themes;
- avoid `TwoWay` binding where races matter;
- use command parameters to capture values synchronously at click time;
- treat UI state changes as asynchronous messages, not direct property writes into a shared object graph.

### Dialogs and prompts

Use prompts for simple user decisions and dialogs/tool windows for richer UI. Avoid modal UI unless the workflow genuinely requires blocking user progress.

### Accessibility and native feel

Visual Studio extensions should feel like part of Visual Studio:

- use VS colors and styles;
- support keyboard navigation;
- keep focus behavior predictable;
- avoid popups during startup;
- do not steal focus without user action;
- respect high-contrast themes;
- localize user-facing strings;
- avoid creating new top-level menus.

Official references:

- Remote UI: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/remote-ui>
- Tool windows: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/tool-window/tool-window>
- Publish checklist: <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist>

## 9. Editor, document, project, and language features

`VisualStudio.Extensibility` provides feature areas for common IDE operations:

| Feature area | Typical use |
|---|---|
| Commands | User-triggered actions in menus/toolbars/context menus. |
| Editor | Text view listeners, text changes, editor margins, file/content-type scoped features. |
| Documents | Work with open documents and document identity. |
| Output window | Write build-like or tool-like output without inventing custom UI. |
| Tool windows | Persistent dockable UI. |
| Prompts/dialogs | User interaction. |
| Debugger visualizers | Custom visualization for debug-time values. |
| Project Query | Query solution/project state. |
| Settings | Store extension settings. |
| Language Server Provider | Add language intelligence through LSP. |

Official overview: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/visualstudio-extensibility#overviews>.

Guidelines:

- prefer the narrowest API that matches the feature;
- do not use editor listeners as a substitute for Roslyn analyzers when the goal is code diagnostics;
- use Roslyn analyzers/source generators for compile-time C# semantics;
- use a VS extension when you need IDE UI, commands, tool windows, debugger integration, project interaction, or editor features not represented by analyzer diagnostics;
- use LSP for language support rather than building a custom language service from scratch where possible.

## 10. VSSDK and Community Toolkit

### When VSSDK remains appropriate

Use VSSDK when:

- the required Visual Studio service is not exposed by the new SDK;
- the extension must support older Visual Studio versions;
- the extension already has a large in-process codebase;
- you need MEF editor exports not yet modeled in `VisualStudio.Extensibility`;
- you need APIs that require in-process COM or shell integration.

### VSSDK package rules

For VSSDK packages:

- derive from `AsyncPackage`;
- use `AllowsBackgroundLoading = true`;
- use `ProvideAutoLoad(..., PackageAutoLoadFlags.BackgroundLoad)` only when autoloading is necessary;
- avoid `GetService` in `InitializeAsync` unless you have switched to the UI thread and understand the deadlock risk;
- prefer `GetServiceAsync` and `IAsyncServiceProvider`;
- use `JoinableTaskFactory.SwitchToMainThreadAsync` for UI-thread-only APIs;
- never block the UI thread waiting for async work;
- add `Microsoft.VisualStudio.SDK.Analyzers`;
- keep VSPackage registration attributes accurate because they are used to generate the package's `.pkgdef` file.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/how-to-use-asyncpackage-to-load-vspackages-in-the-background>
- <https://learn.microsoft.com/visualstudio/extensibility/managing-multiple-threads-in-managed-code>
- <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist#adhere-to-threading-rules>
- <https://learn.microsoft.com/visualstudio/extensibility/internals/registering-vspackages>

### In-process VisualStudio.Extensibility and bridging VSSDK/MEF services

When the new SDK does not yet cover a scenario, you do not have to drop back to a full VSSDK package. `VisualStudio.Extensibility` extensions can run *in-process*, which keeps the modern SDK programming model while giving access to VSSDK and MEF services to cover the feature gap. The trade-off is that an in-process extension shares the Visual Studio process, so it loses process isolation and must target .NET Framework, and it must obey the same UI-thread rules as VSSDK code. Before choosing in-process hosting, check whether the functionality you need is reachable as a [brokered service](#brokered-services), which keeps the extension out-of-process and isolated.

To host the new model in-process, set `RequiresInProcessHosting` to `true` on the extension and (for an existing VSSDK project) enable VSSDK compatibility:

```csharp
[VisualStudioContribution]
internal sealed class ExtensionEntrypoint : Extension
{
    public override ExtensionConfiguration ExtensionConfiguration => new()
    {
        RequiresInProcessHosting = true,
    };
}
```

```xml
<PropertyGroup>
  <!-- Enables VSSDK features for compatibility in a new-model project. -->
  <VssdkCompatibleExtension>true</VssdkCompatibleExtension>
</PropertyGroup>
```

When adding new-model parts (commands, tool windows, editor listeners) to an *existing* VSSDK extension, also declare the combined extension type in `source.extension.vsixmanifest`:

```xml
<Installation ExtensionType="VSSDK+VisualStudio.Extensibility">
```

Consume Visual Studio SDK services through .NET dependency injection rather than `GetServiceAsync`/MEF imports directly. The `AsyncServiceProviderInjection<TService, TInterface>` and `MefInjection<TService>` classes (namespace `Microsoft.VisualStudio.Extensibility.VSSdkCompatibility`) let you inject those services into the constructor of a DI-created part:

```csharp
[VisualStudioContribution]
internal sealed class MyCommand : Command
{
    private readonly MefInjection<IBufferTagAggregatorFactoryService> _tagAggregatorFactory;
    private readonly AsyncServiceProviderInjection<DTE, DTE2> _dte;

    public MyCommand(
        MefInjection<IBufferTagAggregatorFactoryService> tagAggregatorFactory,
        AsyncServiceProviderInjection<DTE, DTE2> dte)
    {
        _tagAggregatorFactory = tagAggregatorFactory;
        _dte = dte;
    }
}
```

For in-process parts that must touch the UI thread, the framework also injects `JoinableTaskFactory` and `JoinableTaskContext`; use them to switch to the main thread without deadlocking, exactly as VSSDK code does. Prefer services offered directly by the `VisualStudio.Extensibility` SDK first, and reach for these bridges only for the missing API.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/in-proc-extensions>
- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/dependency-injection#additional-services-for-in-process-extensions>

### Brokered services

Brokered services are the RPC mechanism that lets out-of-process extensions call into Visual Studio (and lets components call each other) across process boundaries. They are the preferred way to consume Visual Studio functionality that is not surfaced directly by the `VisualStudio.Extensibility` SDK convenience APIs, because they work from the isolated extension host without requiring in-process hosting.

A brokered service is acquired from an `IServiceBroker` as a strongly typed proxy. In a `VisualStudio.Extensibility` extension, inject the broker directly (the SDK registers it in the extension's dependency-injection graph) and request well-known services from `VisualStudioServices`:

```csharp
using Microsoft.ServiceHub.Framework;
using Microsoft.VisualStudio.RpcContracts.FileSystem;

internal sealed class MyComponent
{
    private readonly IServiceBroker _serviceBroker;

    public MyComponent(IServiceBroker serviceBroker)
    {
        _serviceBroker = serviceBroker;
    }

    public async Task UseServiceAsync(CancellationToken cancellationToken)
    {
        IFileSystem? fileSystem = await _serviceBroker.GetProxyAsync<IFileSystem>(
            VisualStudioServices.VS2022.FileSystem,
            cancellationToken);

        try
        {
            if (fileSystem is not null)
            {
                // Call methods on the proxy.
            }
        }
        finally
        {
            // Always dispose the proxy when finished with it.
            (fileSystem as IDisposable)?.Dispose();
        }
    }
}
```

Rules and caveats:

- a service is identified by a `ServiceRpcDescriptor` (the well-known ones live on `VisualStudioServices`, grouped by Visual Studio version such as `VS2022`); choose the lowest version that exposes the API you need so the extension runs on more Visual Studio builds;
- `GetProxyAsync<T>` returns `null` when the service is unavailable (wrong version, not activated, or not installed) — always null-check rather than assuming success;
- the returned proxy is disposable; dispose it (`(proxy as IDisposable)?.Dispose()`) when you are done, because it owns an RPC channel. Do not cache a proxy for the lifetime of the extension and reuse it indefinitely — acquire it close to use, or re-acquire after disposal;
- proxies are not guaranteed thread-safe; do not call a single proxy concurrently from multiple threads;
- arguments and return values must be serializable by the RPC layer; pass data contracts, not live object graphs;
- to author your own brokered service, define an interface contract, a `ServiceRpcDescriptor`, register it with `ProvideBrokeredServiceAttribute` or `ExportBrokeredServiceAttribute`, proffer it through `IBrokeredServiceContainer` or MEF, and let consumers acquire it through their own `IServiceBroker`.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/use-and-provide-brokered-services>
- <https://learn.microsoft.com/visualstudio/extensibility/internals/brokered-service-essentials>
- <https://learn.microsoft.com/visualstudio/extensibility/how-to-consume-brokered-service>
- <https://learn.microsoft.com/visualstudio/extensibility/how-to-provide-brokered-service>

### Community Toolkit rules

The Community Toolkit is useful, but do not treat it as a separate runtime model. It still runs in-process and inherits VSSDK's threading and reliability constraints.

Use it to reduce VSSDK boilerplate, not to avoid understanding:

- UI thread affinity;
- `JoinableTaskFactory`;
- package load timing;
- command visibility;
- service retrieval;
- VSIX metadata.

## 11. Visual Studio extension MSBuild properties and `.pkgdef` registration

This section applies to VSSDK and hybrid extensions that still contain an in-process VSPackage. Most pure out-of-process `VisualStudio.Extensibility` extensions do not need a hand-authored `.pkgdef` file because their contributions are discovered from the new SDK's generated metadata.

Microsoft does not publish one single human-written table for every property imported by `Microsoft.VSSDK.BuildTools`. The practical catalog below lists the Visual Studio extension-specific MSBuild properties extension authors commonly set or intentionally override. Properties ending in `DependsOn` are extension points for custom build targets; properties beginning with `_` are usually implementation details and should not be set directly.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/vsix-extension-schema-2-0-reference#placeholder-syntax-for-extension-manifests>
- <https://learn.microsoft.com/visualstudio/extensibility/vsix/get-started/extension-anatomy#compilation>
- <https://learn.microsoft.com/visualstudio/extensibility/preparing-extensions-for-windows-installer-deployment>

### VSIX and VSSDK project-shape properties

| Property | Applies to | What it controls | When to set it |
|---|---|---|---|
| `VsixType` | Legacy VSSDK / VSIX projects | Selects the VSIX build mode used by newer VSSDK tooling. Microsoft Learn's compatibility guidance recommends `v3` for newer VSIX projects that must round-trip across modern Visual Studio versions. | Set to `v3` in older or migrated project files when following VSSDK compatibility guidance. New templates usually set the right value. |
| `VisualStudioVersion` | Legacy VSSDK projects | Chooses the Visual Studio toolset/version context used by imported VSSDK targets. | Set conditionally only for compatibility scenarios, for example when keeping older projects buildable across Visual Studio versions. |
| `MinimumVisualStudioVersion` | Legacy VSSDK projects | Communicates the minimum Visual Studio version expected by the project system/build tooling. | Use with `VisualStudioVersion` in migrated legacy projects. Microsoft Learn's compatibility guidance sets it to `$(VisualStudioVersion)`. |
| `TargetFramework` with `vs16.x`, `vs17.x`, or later VS target monikers | SDK-style VSIX projects | Selects the Visual Studio version targeted by SDK-style VSIX build logic. Microsoft Learn shows `vs17.0` for Visual Studio 2022 and `vs16.10` for Visual Studio 2019; future SDKs can add newer monikers. | Use in SDK-style VSIX projects so manifest placeholders such as `GetInstallationTargetVersion` and `GetPrerequisiteTargetVersion` produce the right Visual Studio version ranges. |

Example compatibility properties from legacy VSSDK guidance:

```xml
<PropertyGroup>
  <VisualStudioVersion Condition="'$(VisualStudioVersion)' == ''">17.0</VisualStudioVersion>
  <MinimumVisualStudioVersion>$(VisualStudioVersion)</MinimumVisualStudioVersion>
  <VsixType>v3</VsixType>
</PropertyGroup>
```

Example SDK-style VSIX target framework:

```xml
<PropertyGroup>
  <TargetFramework>vs17.0</TargetFramework>
</PropertyGroup>
```

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/how-to-roundtrip-vsixs#modify-the-project-file-myprojectcsproj>
- <https://learn.microsoft.com/visualstudio/extensibility/vsix-extension-schema-2-0-reference#placeholder-syntax-for-extension-manifests>

### VSIX package creation properties

| Property | Applies to | What it controls | When to set it |
|---|---|---|---|
| `CreateVsixContainer` | VSIX/VSSDK projects | Creates the final `.vsix` package. VSSDK targets default it to `true` in normal extension projects. | Set to `false` only for unusual builds that intentionally produce loose extension files without creating a VSIX. Keep `true` for Marketplace or normal local install testing. |
| `TargetVsixContainerName` | VSIX/VSSDK projects | File name of the generated VSIX. VSSDK targets default it from `$(TargetName).vsix`. | Override for CI artifact naming, branding, or multi-targeted builds that need distinct package names. |
| `TargetVsixContainer` | VSIX/VSSDK projects | Full output path of the generated VSIX. VSSDK targets normally place it under the output directory. | Override only when a build pipeline requires a different artifact location. |
| `ZipPackageCompressionLevel` | VSIX/VSSDK projects | Compression level used when creating the VSIX ZIP container. VSSDK targets commonly use faster/no compression for Debug and normal compression for Release. | Keep defaults for local development. Set explicitly in deterministic CI packaging if artifact size or build speed matters. |
| `CopyZipOutputToOutputDirectory` | VSIX/VSSDK projects | Controls whether the generated zip/package output is copied to the output directory. | Leave enabled for normal project builds. Disable only when a custom target stages artifacts elsewhere. |
| `CopyVsixManifestToOutput` | VSIX/VSSDK projects | Copies the processed VSIX manifest to the output when packaging, copying, or deployment needs it. | Leave default unless a custom packaging pipeline creates or stages the manifest itself. |

### VSIX deployment and debugging properties

| Property | Applies to | What it controls | When to set it |
|---|---|---|---|
| `DeployExtension` | VSIX/VSSDK projects | Deploys the extension to the Visual Studio Experimental Instance during build/debug. The VSIX project properties UI exposes this behavior. | Keep enabled for F5 development. Set to `false` in CI or release packaging builds that should not modify a local Experimental Instance. |
| `VSSDKTargetPlatformRegRootSuffix` | VSSDK projects | Root suffix used by VSSDK deployment targets, commonly the Experimental Instance suffix. | Override when testing in a non-default experimental hive. Keep the default for ordinary F5 debugging. |
| `StartAction` | Visual Studio project debug properties | Determines how F5 starts debugging. | For VSSDK projects, use `Program` when launching Visual Studio directly. |
| `StartProgram` | Visual Studio project debug properties | Program launched by F5. | Point to `$(DevEnvDir)devenv.exe` for in-process VSSDK debugging. |
| `StartArguments` | Visual Studio project debug properties | Arguments passed to `StartProgram`. | Use `/rootsuffix Exp` or another root suffix so debugging starts an Experimental Instance instead of the main instance. |

Example debug properties:

```xml
<PropertyGroup>
  <DeployExtension>true</DeployExtension>
  <StartAction>Program</StartAction>
  <StartProgram>$(DevEnvDir)devenv.exe</StartProgram>
  <StartArguments>/rootsuffix Exp</StartArguments>
</PropertyGroup>
```

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/how-to-roundtrip-vsixs#modify-the-project-file-myprojectcsproj>.

### VSIX content-inclusion properties

| Property | Applies to | What it controls | When to set it |
|---|---|---|---|
| `IncludeAssemblyInVSIXContainer` | VSIX/VSSDK projects | Includes the main built assembly in the VSIX. | Keep `true` for most extensions. Set `false` only if the package is purely metadata/content or the assembly is supplied by another asset. |
| `IncludeAssemblyFromOutputFolderInVSIXContainer` | VSIX/VSSDK projects | Includes the assembly from the output folder instead of the normal built-project output item path. | Use only for custom output-layout builds where the assembly must be taken from final output. |
| `AssemblyVSIXSubPath` | VSIX/VSSDK projects | Subfolder inside the VSIX where assemblies and related files are placed. | Set when the extension needs a custom folder layout. Keep unset for standard template layout. |
| `IncludeAddModulesInVSIXContainer` | VSIX/VSSDK projects | Includes managed module outputs from `AddModules`. | Keep default unless you deliberately use add-module outputs. |
| `IncludeSGenDllInVSIXContainer` | VSIX/VSSDK projects | Includes serialization assemblies generated by `SGen`. | Keep default unless excluding generated serialization assemblies is intentional and tested. |
| `IncludeDebugSymbolsInVSIXContainer` | VSIX/VSSDK projects | Includes `.pdb` files in the packaged VSIX. VSSDK targets typically default this to `false`. | Enable for private/internal diagnostic builds. Keep disabled for public Marketplace builds unless you intentionally ship symbols. |
| `IncludeDebugSymbolsInLocalVSIXDeployment` | VSIX/VSSDK projects | Includes `.pdb` files in local Experimental Instance deployment. VSSDK targets typically default this to `true`. | Keep enabled for local debugging; disable only for reproducing release-like installed behavior. |
| `IncludeDocFilesInVSIXContainer` | VSIX/VSSDK projects | Includes XML documentation files. VSSDK targets typically default this to `false`. | Enable only if runtime components or consumers need XML docs. |
| `IncludeSatelliteAssembliesInVSIXContainer` | VSIX/VSSDK projects | Includes localized satellite assemblies. | Keep enabled for localized extensions. Disable only for invariant-culture/private builds. |
| `IncludeCOMReferencesInVSIXContainer` | VSIX/VSSDK projects | Includes outputs from COM references when copy-local data exists. | Keep default if the extension depends on COM interop assemblies. Disable only when those dependencies are provided by Visual Studio or another installer. |
| `IncludeCopyLocalReferencesInVSIXContainer` | VSIX/VSSDK projects | Includes copy-local references in the VSIX. | Keep enabled for normal dependency packaging. Disable only when dependencies are already provided by Visual Studio or would create binding/version conflicts. |
| `IncludePkgdefInVSIXContainer` | VSSDK/hybrid projects | Includes generated `.pkgdef` files in the VSIX when `GeneratePkgDefFile` is enabled. | Keep enabled for VSSDK packages. Disable only when using a fully custom registration/deployment mechanism. |

Microsoft Learn's VSIX schema documentation describes project output groups such as `BuiltProjectOutputGroup`, `DebugSymbolsProjectOutputGroup`, `DocumentationFilesProjectOutputGroup`, `PkgDefProjectOutputGroup`, and `SatelliteDllsProjectOutputGroup`. The properties above decide which of those build outputs normally become VSIX source items.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/vsix-extension-schema-2-0-reference#placeholder-syntax-for-extension-manifests>.

### Loose-file and MSI-staging properties

| Property | Applies to | What it controls | When to set it |
|---|---|---|---|
| `CopyVsixExtensionFiles` | VSIX/VSSDK projects | Copies the files that would be in the VSIX to a directory as loose extension content. | Use when preparing extension contents for an MSI/setup project or custom installer. Do not use as a substitute for testing the actual VSIX. |
| `CopyVsixExtensionLocation` | VSIX/VSSDK projects | Destination directory for `CopyVsixExtensionFiles`. | Set to the directory consumed by the setup project or packaging pipeline. |

Microsoft Learn's MSI deployment guidance says to set the VSIX manifest `InstalledByMsi` element to `true` and use the VSIX project property page to copy VSIX content to a setup-project pickup location. The MSBuild properties above are the project-file-level representation of that staging workflow.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/preparing-extensions-for-windows-installer-deployment>.

### Extension points for custom build logic

The VSSDK targets expose `DependsOn` properties so advanced projects can insert custom targets without replacing Microsoft targets. Use these only when normal item metadata or manifest assets are not enough.

| Property | Use |
|---|---|
| `GetVsixSourceItemsDependsOn` | Add targets that produce extra files to package as VSIX source items. |
| `IncludeVSIXItemsDependsOn` | Add targets that classify or transform items before packaging. |
| `GeneratePkgDefDependsOn` | Add work before or around `.pkgdef` generation. |
| `CopyPkgDefDependsOn` | Add work to stage or transform generated `.pkgdef` files. |
| `CopyVsixManifestFileDependsOn` | Add work around processed manifest staging. |
| `ValidateVsixManifestDependsOn` | Add validation before packaging. |
| `ValidateVsixPartsDependsOn` | Add validation for package parts. |
| `CreateVsixContainerDependsOn` | Add steps before the final VSIX container is created. |
| `CopyVsixExtensionFilesDependsOn` | Add steps before loose extension files are copied. |
| `DeployVsixExtensionFilesDependsOn` | Add steps before local Experimental Instance deployment. |

Prefer adding items to existing output groups or declaring assets in the VSIX manifest before overriding these hooks. Custom target hooks increase build complexity and can break when VSSDK build tools change.

### What a `.pkgdef` file is

A `.pkgdef` file is Visual Studio's private-registry registration file for extensions. Microsoft Learn describes it as the file that contains the registration information that would otherwise be added to the system registry. Visual Studio uses `.pkgdef` data to describe and locate VSPackages, services, tool windows, editor factories, language configuration entries, project-system registrations, and other shell-level extension points.

The important historical point is that `.pkgdef` replaced direct machine-registry deployment for Visual Studio extensions. This made VSIX deployment practical because an extension can carry registration information in the package instead of requiring an installer with registry-write privileges.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/internals/registering-vspackages>
- <https://learn.microsoft.com/visualstudio/extensibility/internals/createpkgdef-utility>

### How `.pkgdef` files are generated

For managed VSPackages, registration usually starts as attributes in code. Examples include package registration, tool-window registration, auto-load contexts, command resources, provided services, editor factories, and project-related registrations. The VSSDK build then runs the package-definition generation step, conceptually equivalent to `CreatePkgDef`, against the built assembly and emits a `.pkgdef` file.

Most Visual Studio SDK project templates generate `.pkgdef` files automatically as part of the build. You should normally prefer generated `.pkgdef` files over manually maintained registry fragments because the source-of-truth stays close to the package class and the compiler can catch more mistakes.

### `RegisterWithCodebase`

`RegisterWithCodebase` controls how the generated registration tells Visual Studio where to find the VSPackage assembly. It corresponds to the `CreatePkgDef /codebase` behavior: the generated `.pkgdef` uses a `CodeBase` entry with a file path to the package DLL instead of relying on assembly identity alone.

Use `RegisterWithCodebase` when the package assembly is deployed in the VSIX extension folder or another private installation directory and Visual Studio needs an explicit path to load it. This is the normal shape for many VSIX-deployed VSSDK packages because the package DLL is not installed into Visual Studio's `PublicAssemblies`, `PrivateAssemblies`, or the global assembly cache.

Typical cases:

- the project is a VSIX-deployed VSSDK package and the package DLL lives under the extension's installation folder;
- the generated `.pkgdef` must contain a path-based `CodeBase` entry so Visual Studio can locate the package assembly;
- a migrated project is generating `.pkgdef` output but Visual Studio fails to load the package because the registration does not point to the deployed DLL location.

Example project properties:

```xml
<PropertyGroup>
  <GeneratePkgDefFile>true</GeneratePkgDefFile>
  <RegisterWithCodebase>true</RegisterWithCodebase>
</PropertyGroup>
```

Do not use `RegisterWithCodebase` as a general fix for every package-load problem. If the package assembly is intentionally installed in Visual Studio's `PrivateAssemblies` or `PublicAssemblies` folder, Microsoft Learn notes that a `CodeBase` registry entry is not required. If the package is loaded by strong assembly identity or by a different deployment mechanism, use the registration mode expected by that deployment model.

Related but different: `[ProvideCodeBase]` can add CLR `codeBase` entries for dependent assemblies so the runtime can find those dependencies. `RegisterWithCodebase` is about locating the VSPackage assembly itself; `[ProvideCodeBase]` is about dependent assembly binding.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/internals/createpkgdef-utility>
- <https://learn.microsoft.com/visualstudio/extensibility/internals/specifying-vspackage-file-location-to-the-vs-shell>
- <https://learn.microsoft.com/dotnet/api/microsoft.visualstudio.shell.providecodebaseattribute>

### When to set `GeneratePkgDefFile` to `true`

Set the MSBuild property `GeneratePkgDefFile` to `true` when the project is a VSSDK package project that contributes Visual Studio registration through package attributes and you need the build to emit a `.pkgdef` file for VSIX packaging or experimental-instance deployment.

Typical cases:

- the project contains a class derived from `Package` or `AsyncPackage`;
- the project has VSSDK registration attributes such as package registration, provided services, tool windows, editor factories, or auto-load UI contexts;
- the project contributes in-process shell behavior in a hybrid extension;
- a migrated or hand-authored SDK-style project no longer uses the old VSSDK project template but still needs VSPackage registration;
- the build output is missing the generated `.pkgdef`, and the installed VSIX cannot load or locate the package even though the assembly is present.

Example project property:

```xml
<PropertyGroup>
  <GeneratePkgDefFile>true</GeneratePkgDefFile>
</PropertyGroup>
```

If the extension is a pure out-of-process `VisualStudio.Extensibility` extension and does not contain a VSSDK package, do not turn this on just because the project builds a VSIX. The new model uses contribution metadata rather than VSPackage registry data for commands, tool windows, prompts, and other supported new-SDK features.

### When to keep `GeneratePkgDefFile` unset or `false`

Do not generate a `.pkgdef` file for projects that are only support libraries, test projects, analyzers, source generators, or extension-independent business-logic assemblies. These projects do not register VSPackages and should not appear as Visual Studio package assets.

Also avoid generating an empty or stale `.pkgdef` as a workaround for a packaging issue. If Visual Studio cannot load a package, fix the underlying registration attributes, VSIX assets, installation target, or version range instead of shipping unrelated registration data.

### Hand-authored `.pkgdef` files

Some scenarios still require a hand-authored `.pkgdef` file. For example, Microsoft Learn's language-configuration documentation shows a `.pkgdef` file used to point Visual Studio at a language configuration file and grammar. In those cases:

- set the `.pkgdef` file's build action to `Content`;
- include it in the VSIX;
- ensure it is copied to output when needed;
- add it as a VSIX asset, commonly with `Type="Microsoft.VisualStudio.VsPackage"`.

Use hand-authored `.pkgdef` entries sparingly. They are powerful, but they are also easy to make stale because they duplicate registration state outside code.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/language-configuration#create-a-pkgdef-file>.

## 12. Threading, async, performance, and reliability

### General rules

- Assume every operation that crosses into Visual Studio can be asynchronous.
- Honor cancellation tokens.
- Keep extension startup and command activation cheap.
- Do not perform network I/O on the UI path.
- Do not synchronously read large files from command handlers.
- Cache only immutable data that is safe for the extension lifetime.
- Keep mutable shared state behind services with clear lifetime and threading rules.
- Treat `IClientContext` as a snapshot.
- Do not assume extension parts are constructed in a particular order.

### Out-of-process model

Out-of-process extensions avoid many VSSDK main-thread problems, but they introduce distributed-systems problems at a small scale:

- data can be stale after an `await`;
- UI binding updates are asynchronous;
- RPC failures can happen;
- not every in-process object can be serialized;
- excessive chatty calls across the process boundary can become slow.

Design APIs and services to batch operations when possible.

### Reentrancy protection for long-running operations

For operations that mutate solution or project state across multiple async steps
(rename flows, file moves, refactoring orchestration, bulk project updates), use an
explicit reentrancy guard pattern:

1. Enter a dedicated operation scope before the first mutation.
2. Activate a UI context or equivalent state flag used by command visibility/enabled
   logic.
3. Block known-disruptive commands during the critical window (for example build,
   close solution, conflicting rename commands, or project add/remove operations).
4. Surface a short user-facing message when an operation is blocked so users know
   the behavior is temporary.
5. Always clear the guard in a `finally` block so commands re-enable after success,
   cancellation, or failure.

Implementation notes:

- Keep the blocked-command set narrow and scenario-specific.
- Fail open if the guard infrastructure is unavailable, but log the condition.
- Prefer command interception/`QueryStatus` guards over ad-hoc boolean checks spread
  across handlers.
- Treat this as reliability protection, not as a licensing or policy gate.

### UI feedback for long-running operations

When an operation can run longer than a brief UI interaction, make progress and
state explicit so users do not retry commands, assume Visual Studio is hung, or
interrupt the workflow at unsafe points.

Recommended feedback pattern:

1. **Start signal**
   - Immediately show that work started (status bar text, tool-window banner, or
     output pane message).
   - Include operation scope in plain language (for example, "Renaming project and
     related files...").
2. **Progress updates**
   - Prefer coarse-grained milestone updates over high-frequency UI churn.
   - For indeterminate work, show phase-based progress (scan, validate, apply,
     finalize) instead of fake percentages.
   - For determinate work, report completed/total units when available.
3. **Blocked-action feedback**
   - If commands are temporarily disabled by a reentrancy guard, provide a concise
     explanation and expected next step ("Operation in progress; try again when it
     completes.").
   - Avoid repeated modal dialogs; use non-modal surfaces where possible.
4. **Cancellation feedback**
   - If cancellation is supported, expose it from the same UI surface where progress
     is shown.
   - Acknowledge cancellation quickly and report rollback/partial-completion state
     clearly.
5. **Completion feedback**
   - End with a clear success/failure/canceled summary.
   - On failure, include actionable next steps and where to find diagnostics
     (`ActivityLog.xml`, `%TEMP%\VSLogs`, extension output pane).

#### API mapping: `VisualStudio.Extensibility` (out-of-process)

- **Task Status Center progress (preferred for long background work)**
  - Start: `ShellExtensibility.StartProgressReportingAsync(...)`
  - Update: `ProgressReporter.Report(new ProgressStatus(percent, description))`
  - End: dispose `ProgressReporter`.
  - Use this for operations that can outlive a single command callback and should
    be visible in Visual Studio's task/progress UX.
- **Output details stream**
  - Create channel once: `VisualStudioExtensibility.Views().Output.CreateOutputChannelAsync(...)`
  - Write updates: `OutputChannel.WriteLineAsync(...)` or `OutputChannel.Writer.WriteLineAsync(...)`.
  - Use this for milestone/detail logs users might review after completion.
- **User prompts for explicit decisions**
  - `ShellExtensibility.ShowPromptAsync(...)`.
  - Use sparingly for blocking confirmations (cancel/continue/retry), not as a
    primary progress channel.

Notes:

- `VisualStudio.Extensibility` Output window APIs are currently documented as
  preview and may change; keep usage isolated behind a small adapter.
- Prefer one progress reporter per logical operation to avoid noisy concurrent
  progress signals.

#### API mapping: VSSDK / in-process extensions

- **Status bar (lightweight, non-blocking)**
  - `IVsStatusbar.SetText(...)` for status text.
  - `IVsStatusbar.Progress(...)` for determinate progress (known total).
  - `IVsStatusbar.Animation(...)` for indeterminate progress.
- **Threaded Wait Dialog (long-running, optionally cancelable)**
  - Obtain `IVsThreadedWaitDialogFactory` (`SVsThreadedWaitDialogFactory`).
  - Use `ThreadedWaitDialogHelper.StartWaitDialog(...)` or
    `IVsThreadedWaitDialog2/3/4` start/update APIs.
  - Best when work may take several seconds and needs explicit cancel/progress UI.
- **Task Status Center (persistent background task tracking)**
  - Use `IVsTaskStatusCenterService` for operations that should appear in the
    standard Visual Studio background task experience.

Selection guidance:

- Use **status bar/progress reporter** for normal long-running operations.
- Add **Output window** for diagnostic detail and post-run review.
- Escalate to **Threaded Wait Dialog** only when cancel/progress needs a dedicated
  dialog surface.
- Use **Task Status Center** for operations that can continue in background and
  should remain discoverable.

UX and reliability rules:

- Do not block startup or editor hot paths with progress UI.
- Do not spam notifications for sub-second operations.
- Keep messages idempotent and correlation-friendly (operation ID/target name).
- Keep wording consistent across status text, prompts, and logs.

This pattern aligns with Microsoft guidance to avoid UI-thread blocking and to keep
activation/context rules explicit and testable:

- Manage multiple threads in managed code: <https://learn.microsoft.com/visualstudio/extensibility/managing-multiple-threads-in-managed-code>
- Use AsyncPackage to load VSPackages in the background: <https://learn.microsoft.com/visualstudio/extensibility/how-to-use-asyncpackage-to-load-vspackages-in-the-background>
- Rule-based activation constraints: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/activation-constraints>
- Start background progress reporting (`ShellExtensibility.StartProgressReportingAsync`): <https://learn.microsoft.com/dotnet/api/microsoft.visualstudio.extensibility.shell.shellextensibility.startprogressreportingasync>
- Write to the Visual Studio output window (`CreateOutputChannelAsync`, `OutputChannel`): <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/output-window/output-window>
- `ProgressStatus` and `ProgressReporter.Report`: <https://learn.microsoft.com/dotnet/api/microsoft.visualstudio.rpccontracts.progressreporting.progressstatus>
- VSSDK status bar (`IVsStatusbar.SetText`, `Progress`, `Animation`): <https://learn.microsoft.com/dotnet/api/microsoft.visualstudio.shell.interop.ivsstatusbar>
- VSSDK threaded wait dialog (`IVsThreadedWaitDialogFactory`): <https://learn.microsoft.com/dotnet/api/microsoft.visualstudio.shell.interop.ivsthreadedwaitdialogfactory>
- VSSDK task status center (`IVsTaskStatusCenterService`): <https://learn.microsoft.com/dotnet/api/microsoft.visualstudio.taskstatuscenter.ivstaskstatuscenterservice>
- Visual Studio UX guidance for notifications/progress: <https://learn.microsoft.com/visualstudio/extensibility/ux-guidelines/notifications-and-progress-for-visual-studio>

### In-process model

In-process code must be extremely careful with the UI thread. Microsoft Learn warns that Visual Studio threading is hard and that incorrect use has been a constant source of bugs even for internal Visual Studio engineers. The official guidance points to `JoinableTaskFactory` for avoiding deadlocks. See: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/extensibility-models#ui-thread-versus-background-thread>.

### UI delay diagnostics

Visual Studio can notify users when extension modules are on the UI thread stack during a UI delay or crash. The notification does not prove the extension is the root cause, but users may disable the extension to avoid future issues. The official extension management docs describe these notifications and the user choices. See: <https://learn.microsoft.com/visualstudio/ide/finding-and-using-visual-studio-extensions#manage-extensions>.

For UI delay investigation, use activity logs and ETW traces as described in: <https://learn.microsoft.com/visualstudio/extensibility/how-to-diagnose-ui-delays-caused-by-extensions>.

## 13. Configuration, settings, and state

Visual Studio extensions commonly need several kinds of configuration:

| Need | Suggested storage |
|---|---|
| User preferences | Visual Studio settings APIs, extension settings, options pages. |
| Workspace-specific behavior | Files in the repository, `.editorconfig` for analyzer-like behavior, or project properties when appropriate. |
| Licensing token | Secure local storage where possible; never plain text secrets. |
| Cached remote data | Local cache with expiration, versioning, and offline behavior. |
| Feature flags | Remote config only if privacy and offline behavior are documented. |

Guidelines:

- separate user settings from machine cache;
- avoid storing secrets in project files;
- make settings export/import friendly where possible;
- default to offline-safe behavior;
- never block IDE startup while reading remote config;
- document all remote endpoints.

### Configuration precedence model

When an extension can read configuration from multiple channels, document and
implement a deterministic precedence order. A practical default is:

1. command/session override (if your extension supports it);
2. workspace/repository configuration (if applicable);
3. user settings/options page values;
4. remote feature flags/config;
5. built-in defaults.

Rules:

- keep precedence stable across releases;
- if two channels define the same value, log which source won in diagnostic
  output;
- avoid hidden precedence (for example, environment variables silently overriding
  user settings) unless explicitly documented;
- test conflict scenarios (same key set in multiple sources).

### Settings migration and versioning

Treat settings schema as a versioned contract.

- keep stable setting keys when possible;
- when a setting is renamed, support old-key fallback for at least one release;
- when a setting is removed, map it to a safe replacement or ignore it
  explicitly;
- perform migration once at startup/initialization, then persist migrated values;
- include migration notes in release notes for enterprise users who script
  settings deployment.

For VSSDK options pages, changing property names or types can break persisted
values. For VisualStudio.Extensibility settings, changing setting identifiers can
orphan existing user values. Prefer additive evolution over destructive
renaming.

### User-facing configuration surfaces

A production extension should expose configuration in a place users naturally
expect, and the surface depends on the extensibility model.

#### VSSDK / in-process options pages

For VSSDK packages, the standard user-facing configuration entry point is
**Tools > Options** via a `DialogPage` registered with
`[ProvideOptionPage]`.

- Use `DialogPage` for straightforward property-grid settings.
- Use `UIElementDialogPage` or an overridden `Window` only when a custom UI is
  necessary.
- Keep one logical options page per scenario (general, advanced, diagnostics),
  not one giant page with mixed concerns.
- Register each page explicitly with `[ProvideOptionPage]` so it appears under
  a predictable category/subcategory.

This keeps settings discoverable, scriptable (where automation is used), and
consistent with other Visual Studio features.

Example (`DialogPage` + `[ProvideOptionPage]`):

```csharp
using System.ComponentModel;
using Microsoft.VisualStudio.Shell;

internal sealed class DiagnosticsOptionsPage : DialogPage
{
    [Category("Logging")]
    [DisplayName("Enable diagnostic logging")]
    [Description("Enables extension diagnostic logging.")]
    public bool EnableDiagnosticLogging { get; set; }

    [Category("Logging")]
    [DisplayName("Log level")]
    [Description("Error, Warning, Information, or Verbose.")]
    public string LogLevel { get; set; } = "Warning";
}

[PackageRegistration(UseManagedResourcesOnly = true, AllowsBackgroundLoading = true)]
[ProvideOptionPage(
    typeof(DiagnosticsOptionsPage),
    "My Extension",
    "Diagnostics",
    0,
    0,
    true)]
internal sealed class MyPackage : AsyncPackage
{
    internal DiagnosticsOptionsPage Options =>
        (DiagnosticsOptionsPage)GetDialogPage(typeof(DiagnosticsOptionsPage));
}
```

#### VisualStudio.Extensibility settings API

For out-of-process extensions, define settings with
`[VisualStudioContribution]` categories and setting definitions, then read/monitor
values via the settings APIs/observers.

- Keep display names and descriptions localized because users see them in UI.
- Add descriptions and validation metadata so users understand what each setting
  does before changing it.
- Group settings into clear categories; avoid dumping unrelated switches into a
  single category.
- Use observers/subscriptions for live updates so changes apply without restart
  when feasible.

The key principle is the same as VSSDK: put user-facing configuration in a
stable, discoverable UI instead of hidden files or undocumented command-line
flags.

Example (`[VisualStudioContribution]` settings):

```csharp
#pragma warning disable VSEXTPREVIEW_SETTINGS

using Microsoft.VisualStudio.Extensibility;

internal sealed class ExtensionEntrypoint : Extension
{
    [VisualStudioContribution]
    internal static SettingCategory DiagnosticsCategory { get; } =
        new("diagnostics", "Diagnostics");

    [VisualStudioContribution]
    internal static Setting.Boolean EnableDiagnosticLogging { get; } =
        new(
            "enableDiagnosticLogging",
            "Enable diagnostic logging",
            DiagnosticsCategory,
            defaultValue: false)
        {
            Description = "Enables extension diagnostic logging.",
        };

    [VisualStudioContribution]
    internal static Setting.Enum LogLevel { get; } =
        new(
            "logLevel",
            "Log level",
            DiagnosticsCategory,
            defaultValue: "Warning",
            values: ["Error", "Warning", "Information", "Verbose"])
        {
            Description = "Controls diagnostic log verbosity.",
        };
}
```

Reading a value in a command handler:

```csharp
var result = await Extensibility.Settings().ReadEffectiveValueAsync(
    ExtensionEntrypoint.EnableDiagnosticLogging,
    cancellationToken);

var enabled = result.ValueOrDefault(defaultValue: false);
```

### Configurable diagnostic logging

Diagnostic logging should be configurable, because always-on verbose logging can
hurt performance, increase noise, and capture more operational data than a user
expects.

Recommended pattern:

1. Expose a user setting such as `DiagnosticLoggingEnabled` and a log verbosity
   level (`Error`, `Warning`, `Information`, `Verbose`).
2. Default to conservative logging (`Error`/`Warning`), and require explicit
   opt-in for verbose traces.
3. Apply setting changes dynamically where possible (no IDE restart required).
4. Clearly document where logs are written (`%TEMP%\\VSLogs` for
   VisualStudio.Extensibility traces, `ActivityLog.xml` for host/package issues)
   and what they may contain.
5. Add a command/button to open the log location or copy diagnostic info, so
   support requests are easier to fulfill.

Privacy and governance implications:

- treat logging configuration as part of your privacy contract;
- avoid logging source code, secrets, or full file contents by default;
- if verbose mode can include sensitive values, say so directly in the option
  description;
- include log-level and logging-enabled state in support bundles so reports are
  interpretable.

### Self-documenting and help discoverability

Extensions should be self-documented in-product, not only in Marketplace text.
At minimum, users should be able to discover what the extension does, how to
configure it, and where to get support without leaving Visual Studio blindly.

Practical help surfaces:

- a **Help** command (for example under Extensions menu or your extension menu)
  that opens online docs/release notes/support;
- concise descriptions for commands, settings, and options-page fields;
- a short "Getting started" section in a tool window or first-run prompt;
- a diagnostics/help command that collects version, configuration summary, and
  log locations for support tickets;
- Marketplace overview/license/privacy/support links kept in sync with in-product
  help text.

Treat documentation as a feature with versioning discipline: when behavior,
settings, or troubleshooting steps change, update both Marketplace content and
in-product help entry points in the same release.

Official references:

- VisualStudio.Extensibility settings: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/settings/settings>
- Create options pages (`DialogPage`, `ProvideOptionPageAttribute`): <https://learn.microsoft.com/visualstudio/extensibility/internals/creating-options-pages>
- Options and options pages (VSSDK): <https://learn.microsoft.com/visualstudio/extensibility/internals/options-and-options-pages>
- Logging extension diagnostics: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/logging>

## 14. Testing strategy

Testing a Visual Studio extension is harder than testing a normal library because part of the product is the IDE host itself. Use a layered strategy.

### Unit tests

Put most logic in extension-independent libraries and unit test it normally. This includes:

- parsers;
- license-state calculations;
- option validation;
- model transformations;
- generated command text;
- service clients with mocked HTTP handlers;
- serialization logic for Remote UI view models.

### Extension part tests

Where possible, instantiate extension services and command handlers with fake dependencies. Verify:

- cancellation is observed;
- bad input does not throw unexpected exceptions;
- command services call the expected IDE abstraction;
- licensing gates produce clear messages;
- telemetry/logging is not emitted when disabled.

### Experimental Instance tests

Microsoft recommends running extensions in the Experimental Instance during development. F5 launches a separate instance with separate settings and extensions. In Visual Studio 2026 and later, the **Extensions** > **Extension Development** menu can start or reset it.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/the-experimental-instance>
- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/debug-extensions>

Manual test matrix:

- first install;
- update from previous version;
- uninstall;
- command appears only in intended context;
- command disabled state is correct;
- tool window opens, closes, docks, and survives theme changes;
- extension works with no solution loaded;
- extension works during solution load;
- extension works with a large solution;
- extension works after Visual Studio restart;
- extension works offline;
- extension handles expired or missing license;
- extension handles network failure;
- extension works under light, dark, and high-contrast themes.

### Packaged VSIX tests

F5 deployment is not enough. Also test the actual `.vsix` from Release output:

1. Build in Release.
2. Install the `.vsix` into an Experimental Instance or clean VM.
3. Verify Marketplace metadata in Manage Extensions.
4. Verify file layout and dependencies.
5. Uninstall and confirm cleanup.

### Integration and smoke automation

For serious extensions, add a smoke-test script or pipeline that:

- builds the VSIX;
- validates VSIX contents;
- launches an Experimental Instance where practical;
- checks that the extension loads;
- captures activity logs;
- archives logs and VSIX artifacts.

Full UI automation is expensive and brittle, but a small smoke suite catches packaging and activation regressions.

## 15. Debugging and diagnostics

### Debugging out-of-process extensions

When you press F5 for a `VisualStudio.Extensibility` extension, Visual Studio launches the Experimental Instance and the `Microsoft.ServiceHub.Host.Extensibility` process. That ServiceHub process brokers communication between the IDE and the extension host.

Important debugging behavior:

- the extension assembly may not load immediately;
- Visual Studio tracks activation metadata before loading the assembly;
- trigger the command or feature before expecting breakpoints to bind;
- out-of-process call stacks show your extension stack, while Visual Studio stack frames live in another process;
- `IClientContext` values are snapshots.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/debug-extensions>.

### Diagnostics Explorer

The VisualStudio.Extensibility Diagnostics Explorer is an extension for extension authors. It can inspect extensibility points, configuration objects, metadata, events, and client contexts. Microsoft Learn says it is compatible with Visual Studio 2022 17.12 and higher.

Use it to:

- inspect command metadata;
- inspect activation constraints;
- view live extensibility events;
- discover client context keys;
- filter diagnostics by extension ID;
- debug why a contribution is not showing.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/diagnostics/visualstudio-extensibility-diagnostics-extension>.

### Extension logging

`VisualStudio.Extensibility` extension parts can inject a `TraceSource` created by the framework. Microsoft recommends using it for diagnostics because it can integrate with future Visual Studio diagnostics tooling. Current logs are written under `%TEMP%\VSLogs` in `.svclog` XML format and can be opened with Microsoft Service Trace Viewer.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/logging>.

### Activity log

For in-process and host-level issues, start Visual Studio with `/log` and inspect `ActivityLog.xml` under the Visual Studio instance's application data folder. This is especially useful for package load failures, MEF composition failures, VSIX installation issues, and UI delay IDs.

### Support bundle template

To reduce back-and-forth in support tickets, publish a fixed "support bundle"
checklist and ask users to attach it unchanged.

Minimum bundle:

- extension version;
- Visual Studio version/edition/channel;
- `ActivityLog.xml` (captured with `devenv /log` after repro);
- `%TEMP%\\VSLogs\\*.svclog` (for out-of-process extensions);
- clear repro steps with expected vs actual behavior;
- whether the issue reproduces in a reset Experimental Instance;
- if relevant, screenshots or short recording.

Optional but high-value:

- installed extension list (to detect conflicts);
- exported extension settings/options values (with secrets removed);
- solution characteristics (size, project count, target frameworks).

### Diagnosing issues reported after publishing

Development-time debugging assumes you can press F5 and attach a debugger. Once
the extension is on the Marketplace (or in a private gallery), failures are
reported by users on machines, Visual Studio editions, and version bands you do
not control. Diagnose them from evidence the user can collect, not from a local
repro you may not be able to reproduce.

Ask the reporting user for:

- the exact extension version installed (Manage Extensions shows it);
- the Visual Studio version, edition, and channel (Help > About);
- whether the problem is at startup, on a specific command, or during a
  long-running operation;
- whether it reproduces in a reset Experimental Instance or only in their main
  instance (a strong signal of a conflicting extension or corrupted MEF cache);
- the relevant log artifacts described below.

Production log artifacts to request:

- **`ActivityLog.xml`** — have the user relaunch with `devenv /log` and
  reproduce. This captures package load failures, MEF composition failures, and
  VSIX install issues on their machine.
- **`%TEMP%\VSLogs\*.svclog`** — the `TraceSource` output your out-of-process
  extension wrote. This is why you should log meaningful diagnostics through the
  injected `TraceSource` rather than `Console`/`Debug`: it is the one channel you
  can ask a remote user to retrieve.
- **UI-delay / hang report** — when Visual Studio attributes a UI delay or crash
  to your extension, it surfaces a notification and records an ID. Ask for the
  ID and, for deeper analysis, an ETW trace as described in the official UI-delay
  diagnostics guide.

Map common post-publication symptoms to causes:

| Symptom reported by a user | Likely cause | First check |
|---|---|---|
| Extension does not appear after install | Visual Studio version/edition outside the manifest's supported range, or missing required workload | `.vsixmanifest` install targets vs. their VS version/edition |
| Installs but command never shows | Activation constraint is false in their context, or (VSSDK) package failed to load | Diagnostics Explorer client context; `ActivityLog.xml` for load failure |
| VSSDK package fails to load only when installed | Generated `.pkgdef` missing or cannot locate the deployed DLL | `RegisterWithCodebase`, `GeneratePkgDefFile`, `.pkgdef` present in VSIX (Section 11) |
| Works for you, hangs for the user | Synchronous UI-thread work on a slower machine or larger solution | UI-delay ID; move work off the UI thread (Section 12) |
| Update never reaches users | VSIX ID changed, version not increased, or listing not public | Stable VSIX ID and incremented version (Section 20) |
| Tool window blank for some users | Theme/serialization issue surfacing only in their environment | `%TEMP%\VSLogs`; Remote UI serialization rules (Section 8) |

Practices that make production reports diagnosable:

- Log actionable diagnostics through the framework `TraceSource` (out-of-process)
  or the activity log (in-process), including an operation name and a correlation
  ID, so a user's `.svclog` is self-explanatory.
- Surface a clear, copyable error message (and where to find logs) instead of
  failing silently.
- Keep the VSIX ID, command names, and settings keys stable so support scripts
  and user reports remain valid across versions (Section 20).
- Monitor the Marketplace Q&A, ratings, and any crash/telemetry feed after each
  release; the publish workflow treats this as part of shipping (Section 25).
- Reproduce in a clean VM or reset Experimental Instance with the user's reported
  VS version before changing code; most post-publish reports are environment,
  version-range, or conflicting-extension issues rather than logic bugs.

Official references:

- Diagnose UI delays caused by extensions: <https://learn.microsoft.com/visualstudio/extensibility/how-to-diagnose-ui-delays-caused-by-extensions>
- Logging extension diagnostics: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/inside-the-sdk/logging>
- Find, install, and manage extensions (UI-delay/crash notifications): <https://learn.microsoft.com/visualstudio/ide/finding-and-using-visual-studio-extensions#manage-extensions>

## 16. Packaging and VSIX metadata

A VSIX is the standard package for Visual Studio IDE extensions. Microsoft Learn describes a VSIX package as a `.vsix` file containing one or more Visual Studio extensions and metadata used to classify and install them. The VSIX format follows the Open Packaging Conventions and can be opened by ZIP tools.

Metadata to treat as public contract:

- VSIX ID;
- display name;
- publisher;
- version;
- installation targets;
- supported Visual Studio versions;
- supported editions;
- icon and preview image;
- license;
- privacy information;
- tags and categories;
- release notes / overview.

For VSSDK and hybrid extensions, `.pkgdef` files are also part of the package contract. The generated `.pkgdef` tells Visual Studio how to find and register the in-process VSPackage. If the VSIX contains a package assembly but omits the needed `.pkgdef`, Visual Studio can install the VSIX while still failing to discover the package at runtime.

Guidelines:

- keep the VSIX ID stable forever;
- use semantic versioning or another predictable scheme;
- include a 90x90 high-quality PNG icon as recommended by Microsoft publish guidance;
- include a license because it appears in Marketplace, VSIX installer, and Manage Extensions;
- include a privacy notice if the extension sends data to remote endpoints;
- do not include unnecessary assemblies;
- include generated or hand-authored `.pkgdef` files only for VSSDK registration scenarios that require them;
- for VSIX-deployed VSSDK packages, verify the generated registration can locate the package assembly, often by using `RegisterWithCodebase`;
- test installation from the packaged VSIX.

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/vsix/get-started/extension-anatomy>
- <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist>
- <https://learn.microsoft.com/visualstudio/extensibility/internals/createpkgdef-utility>

## 17. Publishing and deployment

### Marketplace publishing

The Visual Studio Marketplace is the primary public distribution channel for Visual Studio IDE extensions. Once published, users can find extensions through Visual Studio's extension management UI.

Marketplace listing fields include:

- internal name;
- display name;
- version;
- VSIX ID;
- logo;
- short description;
- overview;
- supported Visual Studio versions;
- supported Visual Studio editions;
- type;
- categories;
- tags;
- pricing category;
- source code repository;
- Q&A setting.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/walkthrough-publishing-a-visual-studio-extension>.

### Command-line publishing

Use `VsixPublisher.exe` for automated publishing. Microsoft Learn documents the tool at:

```plaintext
${VSInstallDir}\VSSDK\VisualStudioIntegration\Tools\Bin\VsixPublisher.exe
```

Supported commands include `publish`, `deletePublisher`, `deleteExtension`, `login`, and `logout`. The `publish` command takes a payload, publish manifest, optional ignored warnings, and a personal access token.

Example publish manifest shape:

```json
{
  "$schema": "http://json.schemastore.org/vsix-publish",
  "categories": ["coding"],
  "identity": {
    "internalName": "MyExtension"
  },
  "overview": "overview.md",
  "priceCategory": "free",
  "publisher": "MyPublisher",
  "private": false,
  "qna": true,
  "repo": "https://github.com/MyPublisher/MyExtension"
}
```

The documented `priceCategory` values are `free`, `trial`, and `paid`. See: <https://learn.microsoft.com/visualstudio/extensibility/walkthrough-publishing-a-visual-studio-extension-via-command-line#publishmanifest-file>.

### Multi-version publishing strategy

Multi-version publishing means maintaining compatible update flows for users on
more than one Visual Studio generation without breaking auto-update identity.

Core rules:

- keep the same VSIX ID for updates that are part of the same upgrade line;
- publish a strictly higher extension version for every update;
- do not reuse version numbers;
- test each published package against every Visual Studio version it claims.

Marketplace and update behavior to design around:

- Visual Studio installs an update when the incoming VSIX has the same extension
  `ID` and a higher `Version`;
- if ID differs, Visual Studio treats it as a separate extension;
- Marketplace update metadata marks several fields as non-changeable on update,
  including supported Visual Studio versions/editions, so treat those as stable
  per listing.

Practical publishing patterns:

1. **Single listing, single package line**
   - Use one extension ID and one version stream when one VSIX can support all
     intended Visual Studio versions.
   - Best fit when the same binaries and manifest constraints work across the
     whole target matrix.
2. **Separate listings for separate compatibility lines**
   - Use separate extension IDs/listings when compatibility bands diverge (for
     example legacy VSSDK package for older Visual Studio and a modern package
     for newer Visual Studio).
   - This avoids immutable-listing metadata collisions and keeps support
     expectations explicit.
3. **Ringed rollout per compatibility line**
   - Publish private first, validate install/update on each target Visual Studio
     band, then make public.

Versioning policy for parallel lines:

- keep independent version streams per listing (for example `2.x` legacy,
  `3.x` modern) to simplify support and rollback;
- document end-of-support dates for older Visual Studio lines;
- when retiring a line, publish a final release note with migration guidance.

Operational checklist for multi-version publishing:

1. Lock extension identity values (VSIX ID, internal name, publisher).
2. Decide whether one listing is sufficient or separate listings are required.
3. Align manifest installation targets with the intended compatibility line.
4. Build and package each line independently in CI.
5. Run install/update/uninstall smoke tests per supported Visual Studio version.
6. Publish with monotonic versions and archive the exact VSIX artifacts.
7. Monitor post-release reports by Visual Studio version band.

Official references:

- Update behavior by VSIX ID and version: <https://learn.microsoft.com/visualstudio/extensibility/how-to-update-a-visual-studio-extension>
- Marketplace update fields and publishing flow: <https://learn.microsoft.com/visualstudio/extensibility/walkthrough-publishing-a-visual-studio-extension#update-a-published-extension-in-visual-studio-marketplace>
- Multi-version support background: <https://learn.microsoft.com/visualstudio/extensibility/supporting-multiple-versions-of-visual-studio>

### CI/CD automation for publishing

Treat publishing as a gated release pipeline, not a single script step. A robust
pipeline usually has these stages:

1. **Build and package**
   - restore, build, and run tests;
   - produce the Release VSIX artifact;
   - validate manifest/version consistency and required package assets.
2. **Quality and compliance gates**
   - run packaged VSIX smoke checks (preferably in a clean environment);
   - run dependency/license/security checks;
   - fail before publish if any gate is red.
3. **Channel routing**
   - decide target channel from branch/tag/release ring policy (public
     Marketplace, private Marketplace, or enterprise/internal channel).
4. **Publish**
   - execute `VsixPublisher.exe publish` non-interactively;
   - use a publish manifest generated/validated by the pipeline;
   - pass authentication via secure CI secrets, never from repo files.
5. **Post-publish verification**
   - verify listing/version visibility;
   - run an install/update smoke check from the published artifact;
   - emit release notes/notifications and archive immutable artifacts.

Recommended CI/CD controls:

- **Immutable artifacts**: publish exactly the VSIX produced by the validated
  build stage; avoid rebuilding in the publish stage.
- **Secret hygiene**: store PATs/tokens in the CI secret store and scope them to
  minimum required permissions.
- **Version discipline**: enforce monotonic version increments; never reuse a
  version for hotfixes.
- **Branch/tag policy**: allow publishing only from trusted release refs.
- **Rollback readiness**: keep previous known-good artifact + unlist/hotfix
  workflow ready (see rollback subsection below).
- **Traceability**: persist commit SHA, pipeline run ID, artifact hash, publish
  timestamp, and target channel in release metadata.

A practical release-ring model:

- **Ring 0**: internal/private publish for validation;
- **Ring 1**: limited audience (pilot users/teams);
- **Ring 2**: broad public release.

This reduces blast radius when extension behavior differs across Visual Studio
versions, workloads, or enterprise environments.

### Distribution channels and monetization models

Visual Studio Marketplace is the default public channel, but it is not the only
viable distribution path. Choose channel strategy based on audience,
compliance constraints, update governance, and monetization mechanics.

#### Channel options

| Channel | Typical audience | Discovery UX | Update UX | Governance profile |
|---|---|---|---|---|
| Visual Studio Marketplace (public) | Broad developer community | High (search inside VS + web listing) | Managed through Marketplace versions | Microsoft malware scanning + publisher verification + public metadata expectations |
| Public direct VSIX distribution (website/GitHub Releases) | OSS users, niche tools, early adopters | Medium (you drive traffic externally) | Manual or custom update guidance | You own signing, integrity, and support workflow |
| Private Marketplace listing | Controlled pilot groups | Low to medium (requires access) | Marketplace-backed for authorized users | Good for staged rollout before public release |
| Closed enterprise registry/gallery/internal artifact portal | Enterprise-managed fleets | Low external discovery, high internal discoverability | IT-managed rollout cadence | Highest control for compliance, allow-lists, and approval workflows |
| Bundled with enterprise installer / managed software deployment | Locked-down environments | Not self-discovery; distributed by IT catalog | Controlled by enterprise deployment system | Suitable when machines cannot install arbitrary VSIX from internet |

Practical channel guidance:

- use Marketplace for broad adoption and easier discoverability;
- use private/internal channels when legal, data residency, or procurement rules
  restrict public distribution;
- keep the same product documentation quality regardless of channel
  (overview, support, privacy, changelog), because enterprise users still need
  auditability;
- if you support both public and private channels, define whether versions ship
  simultaneously or in staged rings.

#### Monetization by channel

Monetization design should align with the channel's trust and procurement model.

| Monetization approach | Works best in | Notes |
|---|---|---|
| Marketplace `priceCategory=free` + optional donations/sponsorship | Public Marketplace, OSS | Lowest friction; monetize via support tiers/sponsorship rather than paywall pressure inside IDE |
| Marketplace `trial` / `paid` listing + extension-side entitlement checks | Public Marketplace | Marketplace listing category is not a complete entitlement system; keep your own license validation architecture (Section 18) |
| Free VSIX + paid cloud/service backend | Public or private | Common for AI/SaaS-backed extensions; extension is installable but premium features require service license |
| Enterprise site license / contract distribution | Closed enterprise channels | Purchase and entitlement handled outside Marketplace; extension should support offline/proxy/license caching workflows |
| Dual-channel (community free + enterprise paid) | Mixed audiences | Keep feature boundaries and support terms explicit to avoid confusion |

Monetization/channel anti-patterns:

- one pricing model in Marketplace text and a different one in-product;
- trial expiration that blocks uninstall/export/help surfaces;
- network-dependent entitlement checks on every command execution;
- no enterprise path for offline/proxy-constrained customers.

When channel and monetization are mixed (for example Marketplace public +
enterprise private builds), keep three artifacts versioned together per release:

1. technical package (VSIX),
2. commercial terms (license/trial text),
3. operational docs (support/privacy/update notes).

### Update rollback and hotfix strategy

Even with pre-publish testing, bad releases can happen. Define rollback before
you need it.

Recommended release controls:

1. Keep the last known-good VSIX artifact and publish manifest in your release
   storage.
2. If a release is broken, unlist/private the bad version quickly and publish a
   hotfix with incremented version rather than reusing the same version.
3. Maintain a short rollback playbook (owner, steps, communication template,
   verification checks).
4. Post a clear status note in Marketplace overview/Q&A when a rollback or
   hotfix is in progress.
5. After recovery, add a regression test for the escaped defect.

Do not change the VSIX ID during rollback/hotfix; update continuity depends on
stable identity.

### Private deployment

Options:

- share the `.vsix` file directly;
- host on an internal server;
- use a private gallery;
- publish privately in Marketplace during validation;
- distribute through enterprise software deployment tools.

For enterprise environments, validate:

- offline installation;
- proxy behavior;
- certificate and trust requirements;
- Visual Studio edition restrictions;
- uninstall behavior;
- update policy;
- telemetry and privacy approval.

### Closed enterprise registries and internal galleries

In regulated or locked-down organizations, Marketplace may be disallowed even
for free extensions. In that case, distribution usually moves to an internal
software catalog, private extension gallery, or enterprise artifact registry.

Enterprise rollout model:

1. Security/compliance review of VSIX contents and dependencies.
2. Internal signing/trust verification according to company policy.
3. Publication to internal registry/catalog.
4. Staged rollout rings (pilot team -> broader engineering org -> full fleet).
5. Controlled rollback path to previous approved version.

Operational requirements to document for enterprise consumers:

- exact supported Visual Studio editions/versions;
- required workloads/components;
- whether internet access is required after install;
- proxy and certificate requirements for licensing/telemetry/update checks;
- support contact and SLA expectations;
- explicit update cadence (for example monthly train vs ad-hoc hotfix).

If your extension has both public and enterprise builds, clearly label build
lineage (`Public`, `Enterprise`, `Gov`, etc.) and avoid ambiguous version labels
that make support triage difficult.

### Marketplace protections

The Visual Studio extension management documentation states that Marketplace extensions are malware-scanned before public usage and that verified publishers can prove domain ownership. Users can still disable extensions associated with crashes or UI unresponsiveness.

Official reference: <https://learn.microsoft.com/visualstudio/ide/finding-and-using-visual-studio-extensions#marketplace-protections>.

## 18. Licensing, trials, paid versions, sponsorship, and donations

This section is product and legal design guidance, not legal advice.

### License file and user expectations

Microsoft's publishing checklist says a license should always be specified, and that the license is shown on Marketplace, in the VSIX installer, and in the Visual Studio extension management UI. This is important even for free extensions because enterprise users need to know redistribution, modification, warranty, and support terms.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist#add-license>.

Common approaches:

| Extension type | Typical license approach |
|---|---|
| Open-source extension | MIT, Apache-2.0, BSD-2/3-Clause, GPL-family if copyleft is intentional. |
| Free closed-source extension | Custom EULA granting use but not redistribution/modification. |
| Commercial extension | Custom EULA covering seats, devices, organizations, updates, support, telemetry, and refund policy. |
| Internal enterprise extension | Internal software policy plus source repository license headers. |

Practical rules:

- include `LICENSE.md` in the repository and package;
- link the license from the Marketplace overview;
- keep third-party notices for bundled dependencies;
- do not copy a license template without understanding obligations;
- document whether the extension is supported, best-effort, or unsupported.

### Trial versions

The command-line Marketplace publish manifest supports `priceCategory` values `free`, `trial`, and `paid`, but the public Visual Studio extension docs do not describe a general built-in entitlement API for enforcing licenses inside a Visual Studio IDE extension. Therefore, if you offer a trial or paid extension, plan the entitlement flow as part of your own product architecture.

Reasonable trial models:

1. **Time-limited local trial**
   - Start trial on first use.
   - Store a signed local token with start/end date.
   - Do not rely only on editable local timestamps.
   - Provide a clear grace period and purchase path.
2. **Account-based trial**
   - User signs in to your service.
   - Service returns signed entitlement claims.
   - Extension caches claims for offline use.
   - Best for paid SaaS-backed functionality.
3. **Feature-limited free edition**
   - Basic functionality is free.
   - Advanced functionality requires a license.
   - Often friendlier than hard expiration.
4. **Evaluation VSIX**
   - Publish a separate trial extension ID or build channel.
   - Avoid this unless you have a strong reason, because users may lose settings or update continuity between IDs.

Trial UX rules:

- never block Visual Studio startup with licensing UI;
- do not show modal nags during typing/build/debug loops;
- show license state in a command, tool window, or options page;
- allow offline use for a documented period;
- fail closed only for premium features, not for extension uninstall or data export;
- make expiration messages actionable;
- provide enterprise offline activation if selling to locked-down companies.

### Paid versions

Paid extensions need more than a Marketplace price category:

- a purchase flow;
- license issuance;
- license validation;
- refund/cancellation policy;
- support channel;
- privacy policy;
- offline and proxy behavior;
- account deletion/data deletion story;
- clear handling of major-version upgrades.

Architecture recommendations:

- keep license validation in a small service abstraction;
- return signed entitlement documents from the server;
- cache only the minimum required data;
- never store raw secrets in plain text;
- validate server certificates normally; do not bypass TLS errors;
- use exponential backoff for license checks;
- make license checks cancellation-friendly;
- do not phone home on every command execution;
- keep the extension useful enough to let users uninstall, export settings, or disable telemetry after expiration.

### Open-source + commercial (dual licensing) policy

A common model in developer tools is: source code is public, while enterprise use
or enterprise-only capabilities are covered by commercial terms.

Typical variants:

| Model | Community terms | Commercial terms |
|---|---|---|
| Open source core + paid support/SLA | OSI license for code use/modification | Paid support, response SLA, compliance packaging, procurement-friendly terms |
| Copyleft + commercial exception | Copyleft obligations for default use | Commercial license waives/replaces copyleft obligations for enterprise procurement |
| Free community build + paid enterprise build/service | Free package for individuals/small teams | Paid entitlement for enterprise features, policy controls, or managed backend |

Policy rules to keep this defensible:

- keep license terms explicit in repository, Marketplace overview, and in-product
  help;
- state clearly what is free, what is paid, and what triggers commercial terms;
- avoid ambiguous wording such as "free for personal use" without a precise
  definition of commercial/organizational use;
- keep technical behavior aligned with legal text (no hidden paywalls that are
  not described in license docs);
- publish a license FAQ for procurement/legal teams.

### Commercial licensing when the author is an individual (physical person)

An individual author can legally sell commercial licenses, but enterprise
procurement usually requires more operational structure than community sales.

Practical requirements enterprises often expect:

1. A legal seller identity (your legal personal identity or a registered entity).
2. Ability to issue invoices/receipts with tax information required by the
   buyer's country.
3. Contract documents (license terms, support terms, privacy notice,
   data-processing terms when applicable).
4. A payment channel acceptable to enterprises (wire, card, marketplace/vendor
   platform, or approved reseller).
5. Clear support and security contact points.

If you sell as an individual, validate early:

- whether your jurisdiction requires business registration or tax registration
  once revenue crosses thresholds;
- whether you must collect/remit VAT/GST/sales tax for digital goods/services;
- whether target enterprise customers require vendor onboarding that is hard for
  individuals (insurance, formal company data, compliance questionnaires).

A common progression path:

- phase 1: individual sales to small teams / early adopters;
- phase 2: create a legal entity when enterprise demand requires stronger
  procurement compatibility.

This document is not legal or tax advice; use qualified legal/tax counsel for
jurisdiction-specific obligations.

### Sponsorship and donations

The Visual Studio Marketplace listing supports an overview document and a source code repository link. For open-source or donation-supported extensions, use those surfaces to disclose sponsorship options rather than building a payment flow into the IDE.

Recommended approach:

- add sponsor links in `overview.md` and the repository README;
- use GitHub Sponsors or another external donation platform;
- keep donation calls unobtrusive;
- do not show recurring donation popups in Visual Studio;
- do not degrade existing free functionality for users who do not donate;
- disclose whether donations influence roadmap or support priority;
- keep sponsor network calls out of startup and editor hot paths.

Marketplace listing docs mention the `overview` field and `repo` field in the publish manifest. See: <https://learn.microsoft.com/visualstudio/extensibility/walkthrough-publishing-a-visual-studio-extension-via-command-line#publishmanifest-file>.

### Privacy and telemetry

Microsoft's publishing checklist says to add a privacy notice if the extension collects data or communicates with a remote endpoint. This applies equally to:

- telemetry;
- crash reporting;
- license validation;
- update checks;
- AI/cloud features;
- sponsor/donation link tracking;
- feature flags;
- support diagnostics upload.

Telemetry rules:

- disclose what is collected and why;
- avoid source code, file paths, secrets, repository URLs, and customer data;
- provide opt-out where practical;
- batch and send asynchronously;
- never block IDE startup or editor operations;
- avoid telemetry before privacy settings are known.

Using telemetry to detect enterprise usage:

- acceptable only when explicitly disclosed in privacy/licensing documents;
- collect the minimum signal needed for business/operational purpose;
- do not use covert fingerprinting techniques to infer organization identity;
- do not gate core uninstall/help/export flows based on inferred telemetry;
- provide enterprise-friendly controls (opt-out, proxy/offline behavior,
  documented data fields);
- prefer explicit commercial entitlement checks over inference from telemetry.

A safer pattern is: telemetry for product quality + explicit license state for
commercial enforcement.

### License and telemetry disclosure templates

Use the templates below as a starting point and adapt legal/tax/privacy wording
to your jurisdiction and product model.

#### Template A — README / repository `LICENSE-AND-PRIVACY.md`

```markdown
# License and Telemetry Notice

## Licensing
- Source code availability: This project is published with source code access.
- Community usage: Community/non-commercial usage is governed by the terms in `LICENSE.md`.
- Commercial/enterprise usage: Commercial or organizational usage may require a separate commercial license.
- Support terms: Paid support/SLA terms are described in `SUPPORT.md`.

For commercial licensing, contact: support@your-domain.example

## Telemetry
This extension may collect limited operational telemetry for quality and support.

Collected data (example):
- extension version
- Visual Studio version/edition/channel
- feature usage counters
- error/failure events with correlation IDs

Not collected by default:
- source code contents
- repository URLs
- secrets/tokens
- full file contents

## Controls
- Telemetry can be configured in extension settings.
- Verbose diagnostics require explicit opt-in.
- Enterprise/offline users can disable remote telemetry.

## Privacy
See `PRIVACY.md` for data categories, retention, opt-out, and contact details.
```

#### Template B — Marketplace overview snippet

```markdown
## License
This extension is source-available/open-source for community usage under the repository license.
Commercial or enterprise usage may require separate commercial terms.
Contact: support@your-domain.example

## Telemetry and Privacy
The extension collects minimal operational telemetry (version, environment, usage/error counters)
to improve quality and support.
It does not collect source code contents or secrets by default.
Telemetry behavior and opt-out controls are documented in the privacy notice:
https://your-domain.example/privacy
```

#### Template C — In-product options page / settings description

```text
Telemetry level
Controls diagnostics sent by the extension.
- Off: no remote telemetry
- Basic: anonymous usage and error counters
- Diagnostic: extended troubleshooting telemetry (opt-in)

No source code contents or secrets are transmitted by default.
See Privacy Notice for details.
```

#### Template D — First-run disclosure prompt (concise)

```text
This extension can send minimal operational telemetry to improve reliability.
You can choose Off, Basic, or Diagnostic now and change it later in settings.
No source code contents are sent by default.
```

Template usage checklist:

- keep terminology consistent across README, Marketplace, in-product settings,
  and legal pages;
- if commercial terms apply to enterprise usage, define "enterprise" in a
  precise, non-ambiguous way;
- keep contact channels current (sales/support/privacy);
- version your disclosure text and update it in the same release where behavior
  changes.

### Commercial license FAQ template

Use this FAQ template in `COMMERCIAL-LICENSE-FAQ.md` (or your docs site) and
link it from Marketplace overview + in-product help.

```markdown
# Commercial License FAQ

## 1) What counts as enterprise/commercial use?
Commercial use means use by a legal entity (company, government, non-profit,
or educational institution) for organizational work, client work, or
revenue-generating activities.
If your policy has exceptions (for example, OSS maintainers, students, or small
teams), list them explicitly.

## 2) Do contractors need separate licenses?
State your policy clearly:
- seat-based model: each human user needs a seat, including contractors;
- organization model: contractors are covered when working under a licensed
  organization.
Also specify whether shared accounts are prohibited.

## 3) Is the license per-user, per-device, per-project, or per-organization?
Document exactly one primary metric and any secondary constraints.
Examples:
- per-user (named seat) with up to N devices;
- per-organization up to N active developers;
- per-project/repository with unlimited users in that scope.

## 4) How do purchase and invoicing work?
Describe:
- available payment methods;
- whether purchase orders are accepted;
- invoice format and tax fields;
- refund/cancellation terms;
- contact email for procurement.

## 5) Can an individual author sell commercial licenses?
Yes. State seller legal identity and invoicing/tax handling details.
If enterprise onboarding requires a legal entity, mention whether an authorized
reseller/partner channel exists.

## 6) Is offline or proxy-constrained activation supported?
Document:
- offline activation availability;
- license cache duration;
- renewal flow when internet is unavailable;
- proxy/certificate requirements for online checks.

## 7) What telemetry is used for licensing?
Clarify separation between telemetry and entitlement:
- telemetry for product quality/support;
- explicit license state for commercial enforcement.
List data categories and provide privacy notice link.

## 8) What happens when a license expires?
State behavior precisely:
- which premium features are disabled;
- what remains available (for example uninstall/help/export/config access);
- grace period and reactivation flow.

## 9) Are updates included?
Describe update rights by plan:
- all updates while subscription is active;
- only patch updates;
- major-version upgrades sold separately.

## 10) What support is included?
Document support channels and targets:
- response time windows by plan (best effort / business day SLA / priority SLA);
- security issue reporting path;
- required support bundle artifacts.
```

FAQ governance checklist:

- keep FAQ answers contract-consistent with license/EULA text;
- update FAQ in the same release where pricing/license behavior changes;
- include a "last updated" timestamp;
- ensure sales/support contacts are monitored and tested periodically.

### Telemetry schema governance

Treat telemetry as a versioned contract so analytics and support queries remain
stable across releases.

- define a naming convention (`Area.EventName`) and keep names stable;
- version event payloads explicitly when fields change;
- classify fields (operational, product usage, potentially sensitive) and block
  sensitive classes by default;
- document retention windows and deletion policy per event class;
- sample high-volume events rather than dropping them unpredictably;
- include extension version, VS version/channel, and correlation ID in key
  diagnostic events to support production triage.

Keep telemetry specs in source control with code review, the same way you review
public API changes.

### Telemetry ingestion abuse protection

If telemetry is sent to infrastructure you control, assume hostile or malformed
traffic will occur. Design ingestion so spam is expensive for the attacker and
cheap to discard for you.

#### Threats to plan for

- bot-driven high-rate event spam;
- forged telemetry from non-extension clients;
- replay of previously captured requests;
- oversized or deeply nested payloads intended to exhaust CPU/memory;
- high-cardinality payload values that poison analytics and storage.

#### Layered controls

1. **Authentication and transport**
   - require HTTPS/TLS;
   - require an extension-issued token per installation/user/org;
   - support token revocation/rotation.
2. **Rate limiting at multiple dimensions**
   - per IP, per token/install ID, per organization/account, plus a global cap;
   - enforce both request-rate and byte-rate limits;
   - return `429` and require client backoff.
3. **Strict schema validation**
   - allowlist event names;
   - enforce field types, max lengths, and max nesting depth;
   - reject unknown critical fields and malformed payloads early.
4. **Replay resistance**
   - include timestamp + nonce in signed batches;
   - reject stale timestamps and duplicate nonces in a replay window.
5. **Ingestion isolation**
   - front endpoint -> queue/buffer -> async processor -> storage;
   - never write raw incoming payloads directly into primary analytics tables.

#### Resource protection defaults

Apply explicit limits before parsing business payloads:

- max request body size;
- max events per batch;
- max string length per field;
- max JSON depth;
- request timeout and concurrent request cap.

Fail closed for malformed traffic and fail open for extension UX (drop telemetry,
do not block core extension functionality).

#### Detection and response

Detect and alert on:

- sudden spikes per token/IP;
- high invalid-schema ratio;
- repeated replay/signature failures;
- sudden high-cardinality dimensions (for example random IDs in event names).

Automatic mitigations:

- temporary token suspension;
- IP cooldown/block rules;
- dynamic sampling increase;
- emergency "drop non-critical telemetry" mode.

#### Baseline server-side policy (example values)

These are starting points, not universal constants:

- 120 requests/min per token;
- 1 MB/min telemetry bytes per token;
- 100 events per batch;
- 64 KB max request body;
- 10 levels max JSON depth;
- 5-minute replay window for signed batches.

Tune by observing real usage and keep thresholds in configuration, not hard-coded.

#### Implementation notes for solo/SMB operators

If telemetry currently goes directly to a home/self-hosted machine, add a managed
front door first (API gateway/WAF/rate limiter) and keep your ingestion service
behind it. This single step reduces abuse risk significantly.

#### Incident playbook (minimum)

1. Confirm abuse signal and affected dimensions (token/IP/event family).
2. Activate emergency rate-limit/sampling profile.
3. Revoke compromised tokens or block abusive clients.
4. Preserve forensic summaries (not raw sensitive payloads).
5. Publish status note if telemetry degradation affects support workflows.
6. Add regression rules/tests so the same abuse pattern is auto-mitigated later.

#### Recommended stack: mostly .NET + low ops + enterprise-friendly

For teams building primarily on .NET who want minimal operational burden and
enterprise-ready controls, a pragmatic baseline is:

1. **Client instrumentation**
   - use OpenTelemetry-compatible instrumentation in extension-adjacent services
     (or minimal custom telemetry events from the extension itself);
   - keep event schema explicit and versioned (see governance section above).
2. **Managed front door**
   - place an API gateway/WAF/rate limiter in front of ingestion;
   - enforce auth, request size limits, and per-token/IP quotas before events
     reach your backend.
3. **Managed telemetry backend**
   - use Azure Monitor Application Insights as storage/query/alert backend;
   - note: Application Insights is active and supported as part of Azure Monitor
     (not discontinued); modern usage increasingly favors OpenTelemetry-based
     ingestion patterns.
4. **Enterprise operations layer**
   - documented privacy notice + data retention policy;
   - proxy/offline behavior documentation;
   - support bundle workflow tied to correlation IDs.

Reference architecture (logical flow):

`Extension -> HTTPS ingestion endpoint -> WAF/API gateway -> queue/buffer -> processor -> Azure Monitor / Application Insights`

Why this stack is a strong default:

- low ops compared with self-hosted observability clusters;
- good .NET ecosystem fit and tooling familiarity;
- enterprise-friendly controls (RBAC, retention configuration, auditing,
  alerting, and integration with broader Azure governance).

When to deviate:

- strict sovereign/on-prem requirements may require internal SIEM/observability
  platforms;
- multi-cloud neutrality requirements may push toward OpenTelemetry Collector +
  vendor-agnostic backends.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist#add-privacy-notice>.

## 19. Security, privacy, and governance

Visual Studio extensions execute on developer machines that often contain source code, credentials, signing keys, production configuration, and customer data. Treat extensions as high-trust software.

Security checklist:

- minimize permissions and capabilities;
- avoid reading files unrelated to the user's explicit action;
- do not scan entire repositories unless necessary and documented;
- never upload source code without explicit user action;
- sanitize paths and command arguments;
- do not execute repository scripts silently;
- validate all remote responses;
- sign releases where appropriate;
- keep dependencies updated;
- include third-party notices;
- support enterprise proxy and offline policies;
- avoid storing tokens in plain text;
- use the OS credential store where feasible.

Governance checklist:

- define owner(s) for the Marketplace publisher;
- use least privilege publisher roles;
- protect PATs used by `VsixPublisher.exe`;
- require code review for release branches;
- archive release VSIX artifacts;
- keep a changelog;
- have a security contact;
- document support expectations.

Dependency and binding reliability checklist (especially for VSSDK/hybrid):

- pin and review transitive dependencies included in VSIX;
- avoid shipping duplicate assembly versions under different paths in the VSIX;
- validate satellite/resource assembly presence for localized UI;
- test package load in a clean Experimental Instance to catch assembly binding
  differences masked by dev machines;
- when a load failure occurs, correlate `ActivityLog.xml` entries with package
  dependency versions in the VSIX content.

Marketplace publisher roles include Creator, Reader, Contributor, and Owner. See: <https://learn.microsoft.com/visualstudio/extensibility/walkthrough-publishing-a-visual-studio-extension#add-additional-users-to-manage-your-publisher-account>.

## 20. Versioning and compatibility

### Visual Studio version ranges

Do not use broad open-ended ranges casually. Microsoft's publishing checklist recommends supporting only the previous and current Visual Studio version where possible and says not to specify open-ended ranges such as `[16.0,)` unless you have a deliberate compatibility and servicing policy.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist#use-proper-version-ranges>.

Practical policy:

- support the current Visual Studio 2026 channel and the previous supported major channel if your APIs allow it;
- use a preview/internal channel for Insiders-specific features;
- keep separate VSIXs if one version must use different APIs or runtime assumptions;
- test every claimed Visual Studio version;
- update version ranges only after validation.

### Supporting multiple Visual Studio versions

Supporting multiple Visual Studio versions is a compatibility contract, not just
a manifest setting. Decide first whether one VSIX can safely serve all target
versions, or whether separate VSIXs are clearer and safer.

Use one VSIX when:

- the same extension binaries can run on every target version;
- all APIs used by the extension exist in the minimum supported Visual Studio
  version;
- prerequisites and workloads are the same across target versions;
- UI, command placement, activation constraints, and packaging behavior have
  been tested on every claimed version.

Use separate VSIXs when:

- the extension needs different target frameworks, SDK packages, or VSSDK
  references per Visual Studio generation;
- newer Visual Studio APIs are required for newer users, but older users still
  need a maintained build;
- `VisualStudio.Extensibility` is used for modern Visual Studio versions while a
  VSSDK build is kept for older Visual Studio versions;
- the extension has edition/workload-specific dependencies that would make one
  manifest misleading.

Practical version-range examples:

```xml
<!-- Visual Studio 2022 and Visual Studio 2026 under the VS 2026 compatibility model. -->
<InstallationTarget
  Id="Microsoft.VisualStudio.Community"
  Version="[17.0,18.0)" />
```

```xml
<!-- New Visual Studio 2026 templates can use a 17.0 lower bound with no upper bound. -->
<InstallationTarget
  Id="Microsoft.VisualStudio.Community"
  Version="[17.0,)" />
```

```xml
<!-- Legacy VSSDK-style range for a VSIX that intentionally supports VS 2015-2019. -->
<InstallationTarget
  Id="Microsoft.VisualStudio.Community"
  Version="[14.0,17.0)" />
```

For Visual Studio 2026, Microsoft documents an API-version-based compatibility
model for VSIX extensions. Visual Studio 2026 supports API version `17.x`, uses
the lower bound of the installation target version range to evaluate
compatibility, and ignores the upper bound. This means a VSIX that works in
Visual Studio 2022 and targets supported stable APIs can usually load in Visual
Studio 2026 without republishing. Existing extensions with a minimum version of
`16.0` or lower should be updated to declare a `17.0` minimum if they are meant
to participate in the modern compatibility model.

For older Visual Studio support, the manifest range is only one part of the
work. The project must also avoid references unavailable in the minimum Visual
Studio version, use compatible VSSDK build tools, declare prerequisites with
ranges broad enough for all intended versions, and test installation and runtime
behavior in each target IDE. Microsoft Learn's round-tripping guidance for
Visual Studio 2015 through 2019 specifically calls out updating install targets,
prerequisites, `MinimumVisualStudioVersion`, `VsixType`, debug properties, and
conditional imports for build tools.

Checklist for a multi-version VSIX:

1. Pick the minimum supported Visual Studio version based on the oldest API you
   need, not on marketing reach.
2. Reference the VSSDK package/build tools that match the lowest supported
   Visual Studio version, then conditionally handle newer APIs only after
   checking availability.
3. Keep installation targets and prerequisites aligned; do not claim a Visual
   Studio version if a required component did not exist there.
4. Avoid preview APIs for production Marketplace builds unless the target channel
   is explicitly preview/internal.
5. Test install, update, command visibility, command execution, package load,
   tool windows, settings migration, and uninstall in every supported Visual
   Studio major version.
6. For MSI-distributed extensions, handle version detection and installation
   yourself; the VSIX compatibility model does not manage MSI compatibility.

Official references:

- Visual Studio 2026 extension compatibility model: <https://learn.microsoft.com/visualstudio/extensibility/migration/extension-compatibility>
- Round-tripping VSIX projects across Visual Studio 2015/2017/2019: <https://learn.microsoft.com/visualstudio/extensibility/how-to-roundtrip-vsixs>
- Publishing checklist version-range guidance: <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist#use-proper-version-ranges>

### Extension versioning

Use a predictable version scheme:

- patch: bug fixes and compatibility updates;
- minor: new features that do not break workflows;
- major: breaking changes, paid upgrade boundaries, or changed minimum Visual Studio version.

Document:

- minimum Visual Studio version;
- supported editions;
- dependencies and workloads;
- known limitations;
- migration steps for breaking changes.

### Stable IDs

Keep stable:

- VSIX ID;
- command IDs / command names where applicable;
- settings keys;
- license product IDs;
- telemetry event names;
- exported service contracts.

Changing identity values breaks updates, settings migration, user docs, enterprise allow-lists, and support scripts.

### Settings compatibility across versions

When extension settings evolve, compatibility issues can be more disruptive than
API breaks because users notice them as behavior drift.

- prefer additive changes (new settings with defaults) over renaming/removing
  existing keys;
- if a rename is unavoidable, support old-key read + new-key write for at least
  one release cycle;
- include a schema/version marker for complex serialized settings blobs;
- document migration behavior in release notes and support docs;
- add tests that upgrade from N-1 settings payloads to the current version.

## 21. Migration and hybrid architecture

### Migration goals

The end state for many modern extensions should be:

- out-of-process `VisualStudio.Extensibility` for commands, UI, editor, project queries, and supported IDE features;
- shared business logic in normal .NET libraries;
- minimal in-process bridge only for APIs not yet available in the new SDK;
- a clear deletion plan for legacy VSSDK code.

### Migration strategy

1. Inventory all extension features and the Visual Studio APIs they use.
2. Classify each feature as supported by the new SDK, unsupported, or unclear.
3. Move non-Visual-Studio logic to shared libraries.
4. Add unit tests around shared logic before changing host integration.
5. Rebuild one command or tool window in `VisualStudio.Extensibility`.
6. Keep unsupported features behind a bridge. First try a [brokered service](#brokered-services), which keeps the extension out-of-process. If the API is only reachable in-process, the lightest bridge is an *in-process* `VisualStudio.Extensibility` extension (`RequiresInProcessHosting = true`, `VssdkCompatibleExtension = true`) that injects the missing VSSDK/MEF service through `AsyncServiceProviderInjection`/`MefInjection` rather than a full separate VSSDK package. See the in-process hosting subsection in Section 10.
7. Release incrementally and monitor crashes, UI delays, and user feedback.
8. Remove VSSDK dependencies when the new SDK covers the remaining feature.

Microsoft's model-selection guidance recommends separate VSIX projects and shared common libraries when supporting both older and newer Visual Studio versions. See: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/extensibility-models#comparison-chart>.

### Hybrid extension risks

- two hosting models can make debugging harder;
- dependencies may need different target frameworks;
- in-process code can still crash or hang Visual Studio;
- UI behavior may differ between old WPF and Remote UI;
- duplicated command IDs or metadata can confuse users;
- license and settings code must be shared carefully.

Keep the bridge narrow and documented.

## 22. Future direction and acknowledged pain points

### Direction of travel

The direction is clear: Visual Studio extensibility is moving toward out-of-process, async, .NET-based extensions with metadata-driven activation, Remote UI, and brokered services. Microsoft Learn says the goal is that eventually the `VisualStudio.Extensibility` SDK should be able to write any extension you could write with the Visual Studio SDK, while acknowledging that the current SDK does not yet have full VSSDK breadth.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/visualstudio-extensibility>.

### Officially acknowledged pain points in the old model

Microsoft documentation explicitly identifies several old-model problems:

- extension-caused crashes and hangs of Visual Studio and other extensions;
- inconsistent and out-of-date docs and APIs;
- requirement for specialized expertise;
- overwhelming architecture;
- restart required to install extensions;
- no .NET Core / modern .NET support;
- complex threading and main-thread dependencies;
- Community Toolkit simplicity can hide important VSSDK concepts;
- VSSDK APIs accumulated over years and include COM, DTE, and MEF patterns in the same extension.

These are not just community complaints; they are stated in Microsoft Learn's new-model introduction and extensibility model comparison:

- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/oop-extensibility-model-overview>
- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/extensibility-models>

### Current limitations of the new model

The new model's limitations are also acknowledged:

- it is still documented as preview;
- not every VSSDK scenario is available;
- some APIs are experimental;
- out-of-process UI requires Remote UI constraints;
- in-process fallback loses the main benefits of modern .NET and process isolation;
- older Visual Studio versions cannot use the new model.

Follow:

- announcements: <https://github.com/microsoft/VSExtensibility/blob/main/docs/announcements.md>
- breaking changes: <https://github.com/microsoft/VSExtensibility/blob/main/docs/breaking_changes.md>
- experimental APIs: <https://github.com/microsoft/VSExtensibility/blob/main/docs/experimental_apis.md>
- known issues: <https://github.com/microsoft/VSExtensibility/blob/main/docs/known_issues.md>

### Practical forecast for extension authors

Expect the ecosystem to evolve like this:

- more VSSDK feature areas exposed through brokered async contracts;
- more stable APIs and fewer experimental attributes over time;
- stronger diagnostics around activation metadata and Remote UI;
- continued pressure to reduce in-process extension code;
- better hot-loading and no-restart install experiences;
- more extension isolation and reliability signals surfaced to users;
- ongoing need for VSSDK bridges for deep legacy scenarios.

Design extensions so they can move with that evolution: keep core logic host independent, keep Visual Studio integration thin, avoid unnecessary VSSDK dependencies, and monitor breaking-change announcements.

## 23. Typical misuses and antipatterns

The pitfalls below are the failures that most often make a Visual Studio
extension unreliable, slow to load, or rejected from Marketplace. They
consolidate warnings spread throughout this guide and are grounded in Microsoft
Learn's model-selection, threading, Remote UI, and publishing checklist
documentation.

### Model selection and architecture

- **Reaching for VSSDK by default.** For a new extension, start with
  `VisualStudio.Extensibility` and only fall back to VSSDK/Community Toolkit for
  an unsupported API or older Visual Studio support. Defaulting to in-process
  hosting forfeits process isolation and modern .NET.
- **Treating the Community Toolkit as a separate runtime model.** It still runs
  in-process on VSSDK and inherits every UI-thread and reliability constraint;
  the simpler surface can hide threading requirements that still apply.
- **Putting business logic inside extension parts.** Keep parsers, license
  logic, networking, and models in host-independent libraries so they can be
  unit tested and reused across extension models. Command handlers and tool
  windows should stay thin.

### Threading and reliability

- **Doing synchronous file or network I/O on the UI path.** This is the leading
  cause of UI-delay notifications, which can prompt users to disable the
  extension. Keep startup and command activation cheap.
- **Blocking the UI thread on async work** (`.Result`, `.Wait()`,
  `GetAwaiter().GetResult()`). Use `JoinableTaskFactory` and
  `SwitchToMainThreadAsync` for UI-thread-only APIs; never block waiting for a
  task.
- **Calling `GetService` instead of `GetServiceAsync` in `InitializeAsync`.**
  Synchronous service retrieval during package init risks deadlocks; prefer the
  async service provider and switch to the main thread explicitly when required.
- **Ignoring cancellation tokens.** Long operations and command handlers must
  honor the supplied `CancellationToken`.
- **Running multi-step solution/project mutations without a reentrancy guard.**
  If rename/move/update workflows can overlap with build, close-solution,
  project-structure changes, or a second invocation of the same command, race
  conditions and partial state are likely. Use an operation scope plus temporary
  command blocking and guaranteed cleanup in `finally`.
- **Treating `IClientContext` as a live, mutable view of the IDE.** It is a
  snapshot taken at invocation time and can be stale after an `await`; capture
  the values you need synchronously.

### Activation and packaging

- **Eager package autoload to decide whether UI should appear.** Use
  rule-based activation constraints (`VisibleWhen` / `EnabledWhen`) so Visual
  Studio can hide or disable contributions without loading the extension
  assembly.
- **Hard-coding project type GUIDs instead of project capabilities** where a
  capability check is available; capabilities are more robust across project
  systems.
- **Changing the VSIX ID, command names, or settings keys between releases.**
  These are stable contract values; changing them breaks auto-update, settings
  migration, enterprise allow-lists, and support scripts.
- **Generating a `.pkgdef` for projects that register nothing** (support
  libraries, analyzers, test projects), or shipping a stale/empty `.pkgdef` as a
  workaround. Only VSSDK/hybrid package projects with registration attributes
  should emit one.
- **Forgetting `RegisterWithCodebase` for a VSIX-deployed VSSDK package.** When
  the package DLL lives in the extension folder rather than
  `PrivateAssemblies`/`PublicAssemblies`/GAC, the generated registration needs a
  path-based `CodeBase` entry or Visual Studio cannot locate the package.

### Remote UI

- **Expecting arbitrary in-process WPF to work out-of-process.** Remote UI does
  not support code-behind, event handlers, or your own custom controls; design
  around MVVM with XAML, data binding, and async commands.
- **Binding to types that are not Remote UI serializable.** Data context types
  must be `[DataContract]` with `[DataMember]` members, and only Remote
  UI-serializable types (primitives, `IAsyncCommand`, `XamlFragment`,
  serializable collections, etc.) can be bound.
- **Relying on `TwoWay` binding where races matter.** Capture values
  synchronously at click time via command parameters rather than assuming a
  shared object graph updates instantly across the process boundary.
- **Ignoring themes.** Use Visual Studio theme resources so the UI works under
  light, dark, and high-contrast themes.

### Publishing, licensing, and privacy

- **Blocking IDE startup with licensing UI or showing modal nags during typing,
  build, or debug loops.** Surface license state in a command, tool window, or
  options page, and fail closed only for premium features.
- **Using open-ended Visual Studio version ranges** such as `[16.0,)`. Support
  the current and previous version and validate every claimed version.
- **Adding a new top-level menu next to File/Edit.** Publish guidance says to
  feel native to Visual Studio; place commands in existing, contextually
  appropriate menus.
- **Shipping without a license, privacy notice (when any remote communication or
  telemetry exists), high-quality 90x90 PNG icon, or accurate description.**
  These are required by the publishing checklist and appear in Marketplace and
  the installer.
- **Sending source code, file paths, secrets, or repository URLs in telemetry**,
  or phoning home on every command. Disclose, minimize, batch asynchronously,
  and offer opt-out.

### Testing

- **Treating F5 deployment as sufficient.** Always install and test the actual
  packaged `.vsix` from Release output and validate install, update, disable,
  enable, and uninstall in a reset Experimental Instance.

## 24. Troubleshooting

### Extension does not appear

Check:

- Was the VSIX installed into the correct Visual Studio instance?
- Does the `.vsixmanifest` target `Microsoft.VisualStudio.Community` or the correct SKU, not `Microsoft.VisualStudio.IntegratedShell`?
- Is the supported Visual Studio version range correct?
- Does the command have activation constraints that are currently false?
- Does the extension require a workload/component not installed?
- Is the extension private/unpublished in Marketplace?
- Does the Experimental Instance need reset?

### Command appears but is disabled

Check:

- `EnabledWhen` constraints;
- active editor filename/content type;
- solution state;
- project capability;
- client context values in Diagnostics Explorer.

### Breakpoints do not bind

For out-of-process extensions:

- trigger the extension feature first;
- ensure the extension assembly has loaded;
- confirm F5 attached to the ServiceHub extension host;
- rebuild and reset the Experimental Instance if stale binaries are suspected.

### Tool window UI does not render

Check:

- XAML file is embedded as a resource;
- embedded resource logical name matches the `RemoteUserControl` full name;
- XAML does not reference extension-only custom controls;
- data context types are marked with `DataContract` and `DataMember`;
- bound types are serializable by Remote UI;
- exceptions are visible in logs under `%TEMP%\VSLogs`.

### VSSDK package load fails

Check:

- ActivityLog.xml;
- missing dependencies;
- MEF composition errors;
- package registration attributes;
- whether `GeneratePkgDefFile` is enabled for the VSSDK package project;
- whether `RegisterWithCodebase` is needed so the generated `.pkgdef` points to the deployed package DLL;
- whether the generated or hand-authored `.pkgdef` is included in the VSIX;
- VSIX installation target;
- Visual Studio version range;
- target framework;
- synchronous service calls in `InitializeAsync`.

Dependency/binding-specific checks:

- verify the failing assembly version is present exactly once in the VSIX content;
- check `ActivityLog.xml` for `FileNotFoundException`, `FileLoadException`,
  or binding/MEF composition errors that name the missing assembly;
- confirm localized resource/satellite assemblies exist when failures happen only
  under specific UI languages;
- compare the failing machine's installed extension set with a clean
  Experimental Instance to detect extension-to-extension dependency conflicts.

### Users report UI delay notifications

Check:

- command handlers doing work on UI thread;
- synchronous file/network calls;
- package autoload timing;
- MEF components doing work during composition;
- activity log UI delay IDs;
- ETW trace as described in Microsoft docs.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/how-to-diagnose-ui-delays-caused-by-extensions>.

### Rename/move workflow intermittently fails or leaves partial state

Check:

- whether the workflow has a single-operation guard (scope flag/UI context) to
  prevent reentrancy;
- whether disruptive commands are temporarily blocked during the critical window;
- whether cleanup/deactivation runs in a `finally` block on all exits;
- whether blocked-command feedback is clear so users do not retry aggressively;
- whether cancellation/timeout paths release the guard;
- whether logs include operation start/finish/cancel/fail events for correlation.

If failures correlate with overlapping rename/build/project-structure actions,
prioritize reentrancy protection before tuning performance.

### Marketplace update does not reach users

Check:

- VSIX ID did not change;
- version increased;
- Marketplace listing is public;
- supported versions include the user's Visual Studio version;
- the extension was uploaded and then made public;
- users restarted Visual Studio when required for install/update.

## 25. Pre-publish checklist

Before publishing or updating a Visual Studio extension:

1. Choose the correct extensibility model and document why.
2. Verify the extension builds in Release.
3. Install and test the packaged VSIX, not only F5 deployment.
4. Test in a reset Experimental Instance.
5. Test install, update, disable, enable, and uninstall.
6. Verify command visibility and enabled state in relevant contexts.
7. Verify no command appears in irrelevant contexts.
8. Verify startup and solution load are not noticeably impacted.
9. For VSSDK, run `Microsoft.VisualStudio.SDK.Analyzers` and fix threading warnings.
10. For Remote UI, test theme changes and high contrast.
11. Verify cancellation and network failure behavior.
12. Verify offline behavior.
13. Verify license/trial expiration behavior if applicable.
14. Add a high-quality 90x90 PNG icon.
15. Add a clear short description.
16. Add a detailed Marketplace overview with screenshots or GIFs.
17. Add a license.
18. Add a privacy notice if any remote communication or telemetry exists.
19. Add a support contact or issue tracker.
20. Use conservative Visual Studio version ranges.
21. Verify supported editions.
22. Verify third-party dependency licenses and notices.
23. Review extension-specific MSBuild properties such as `CreateVsixContainer`, `DeployExtension`, dependency inclusion flags, and symbol inclusion flags for the intended build configuration.
24. For VSSDK or hybrid extensions, verify `GeneratePkgDefFile` is enabled when package registration attributes must generate a `.pkgdef` file.
25. For VSIX-deployed VSSDK packages, verify `RegisterWithCodebase` is enabled when Visual Studio needs a `CodeBase` entry to locate the package DLL.
26. Verify any generated or hand-authored `.pkgdef` file is included in the VSIX only when needed.
27. Protect publisher PATs and Marketplace owner permissions.
28. Archive the exact VSIX that was published.
29. Monitor Marketplace Q&A, ratings, crash reports, and support channels after release.
30. Validate settings migration from at least one previous released version.
31. Validate configuration precedence behavior when the same option is defined in multiple channels.
32. Verify telemetry event schema/version updates are documented and backward compatible for dashboards/support queries.
33. Rehearse rollback/hotfix steps (owner, unlist action, communication template, and publish of corrected version).
34. Confirm distribution channel strategy for this release (public Marketplace, private Marketplace, direct VSIX, or enterprise registry) and ensure metadata/docs align with that channel.
35. Confirm monetization terms shown in Marketplace/repository/in-product UI are consistent (free/trial/paid language, entitlement behavior, support terms).
36. For enterprise/private channels, verify internal deployment artifacts and approval metadata are published together with the VSIX (release notes, compatibility matrix, support contact).

## 26. Key takeaways

- `VisualStudio.Extensibility` is the strategic model for new Visual Studio extensions.
- VSSDK remains the compatibility and maximum-breadth model.
- The old model's core problem is not just API complexity; it is in-process reliability, UI-thread coupling, and historical layering.
- The new model's core trade-off is that process isolation requires async RPC, serializable state, Remote UI, and a still-growing API surface.
- VSSDK and hybrid extensions still rely on `.pkgdef` registration; set `GeneratePkgDefFile` to `true` when package attributes must produce a `.pkgdef` for deployment, and use `RegisterWithCodebase` when the generated registration must point to the deployed package DLL.
- Use activation constraints to avoid unnecessary loading and irrelevant UI.
- Keep Visual Studio integration thin and business logic host-independent.
- Test in the Experimental Instance and also test the packaged VSIX.
- Treat licensing, telemetry, privacy, and support as product features, not as afterthoughts.
- Prefer unobtrusive sponsorship links in Marketplace/repository docs over donation prompts inside the IDE.
- Follow Microsoft Learn and the VSExtensibility GitHub repository for breaking changes, known issues, and API maturity.
