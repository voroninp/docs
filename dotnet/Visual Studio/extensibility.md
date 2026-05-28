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
| New extension with one unsupported legacy shell operation | Start with `VisualStudio.Extensibility`; isolate the legacy call in an in-process bridge if needed. |
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

Official references:

- <https://learn.microsoft.com/visualstudio/extensibility/visualstudio.extensibility/get-started/create-your-first-extension>
- <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist#make-it-feel-native-to-vs>

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

Marketplace publisher roles include Creator, Reader, Contributor, and Owner. See: <https://learn.microsoft.com/visualstudio/extensibility/walkthrough-publishing-a-visual-studio-extension#add-additional-users-to-manage-your-publisher-account>.

## 20. Versioning and compatibility

### Visual Studio version ranges

Do not use broad open-ended ranges casually. Microsoft's publishing checklist recommends supporting only the previous and current Visual Studio version where possible and says not to specify open-ended ranges such as `[16.0,)`.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/vsix/publish/checklist#use-proper-version-ranges>.

Practical policy:

- support the current Visual Studio 2026 channel and the previous supported major channel if your APIs allow it;
- use a preview/internal channel for Insiders-specific features;
- keep separate VSIXs if one version must use different APIs or runtime assumptions;
- test every claimed Visual Studio version;
- update version ranges only after validation.

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
6. Keep unsupported features behind an in-process bridge.
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

### Users report UI delay notifications

Check:

- command handlers doing work on UI thread;
- synchronous file/network calls;
- package autoload timing;
- MEF components doing work during composition;
- activity log UI delay IDs;
- ETW trace as described in Microsoft docs.

Official reference: <https://learn.microsoft.com/visualstudio/extensibility/how-to-diagnose-ui-delays-caused-by-extensions>.

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
