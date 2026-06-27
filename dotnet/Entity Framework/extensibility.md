# Entity Framework Core 10 Extension Development Guide

Entity Framework Core (EF Core) is built on a **service-oriented architecture**, where nearly every component—from SQL generation to model
validation—is a pluggable service managed by an internal dependency injection (DI) container. This guide details how to design, dynamically extend,
and test robust, AOT-friendly extensions for EF Core 10.

---

## Contents

1. [Extension Strategies](#1-extension-strategies)
2. [Interceptors in EF Core 10](#2-interceptors-in-ef-core-10)
3. [Dynamically Extending the Model](#3-dynamically-extending-the-model)
4. [Internal vs. External Dependency Injection](#4-internal-vs-external-dependency-injection)
5. [Logging and Diagnostics](#5-logging-and-diagnostics)
6. [Query Translation (LINQ to SQL Customization)](#6-query-translation-linq-to-sql-customization)
7. [Custom Type Mapping](#7-custom-type-mapping)
8. [Migrations and SQL Generation](#8-migrations-and-sql-generation)
9. [Design-Time vs. Run-Time](#9-design-time-vs-run-time)
10. [Interoperability and Extension Safety](#10-interoperability-and-extension-safety)
11. [Designing AOT-Friendly Extensions](#11-designing-aot-friendly-extensions)
12. [Gotchas and Antipatterns](#12-gotchas-and-antipatterns)
13. [Testing Extensions](#13-testing-extensions)
14. [Sources and Further Reading](#sources-and-further-reading)

---

## 1. Extension Strategies

EF Core extensions generally fall into three categories based on the depth of framework integration required:

1.  **Interceptors**: Observe, modify, or suppress specific operations in the execution pipeline without replacing services. Implement the relevant
    interceptor interface (e.g., `ISaveChangesInterceptor`), then register the instance via `AddInterceptors()` on `DbContextOptionsBuilder`, or add
    it to the internal service collection inside `IDbContextOptionsExtension.ApplyServices` for library packaging.
2.  **Model Conventions**: Automate model configuration based on CLR types, attributes, or naming patterns. Implement one or more convention
    interfaces (e.g., `IEntityTypeAddedConvention`), expose them through `IConventionSetPlugin`, and register the plugin as an internal service so EF
    Core discovers it automatically from the internal service provider.
3.  **Internal Service Replacement**: Replace core framework logic (e.g., `IQuerySqlGenerator`) via
    `optionsBuilder.ReplaceService<IQuerySqlGeneratorFactory, MyQuerySqlGeneratorFactory>()` for targeted changes in application code, or by
    registering the replacement inside `IDbContextOptionsExtension.ApplyServices` when packaging as a library.

### Overriding SaveChanges vs. Using Interceptors
While `ISaveChangesInterceptor` is the standard for reusable extensions, sometimes a simple override of `SaveChanges` or `SaveChangesAsync` on the
`DbContext` is better:
*   **Override SaveChanges**: Best when logic is **specific to one context type** and does not need to be packaged as a library. It is simpler and
    allows direct access to the context's private state and local application services. Override both methods in the `DbContext` subclass to cover
    both synchronous and asynchronous save paths:

    ```csharp
    private readonly TimeProvider _timeProvider;

    public AppDbContext(
        DbContextOptions<AppDbContext> options,
        TimeProvider timeProvider)
        : base(options)
        => 
        _timeProvider = timeProvider;

    public override int SaveChanges()
    {
        SetAuditFields();
        return base.SaveChanges();
    }

    public override async Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        SetAuditFields();
        return await base.SaveChangesAsync(cancellationToken);
    }

    private void SetAuditFields()
    {
        var now = _timeProvider.GetUtcNow();

        foreach (var entry in ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = now;
            }
            entry.Entity.UpdatedAt = now;
        }
    }
    ```

*   **Use an Interceptor**: Best for **reusable libraries** (e.g., a NuGet package for auditing). Interceptors move cross-cutting concerns out of
    application logic, ensuring consistency across multiple context types. Implement the relevant interface, register the instance via
    `AddInterceptors()`, and package the extension via `IDbContextOptionsExtension` as described in Section 4.

---

## 2. Interceptors in EF Core 10

Interceptors allow you to influence the execution pipeline. Below are the interceptor interfaces available in EF Core 10:

| Interceptor | Operations Intercepted | Singleton? |
| :--- | :--- | :--- |
| **IDbCommandInterceptor** | SQL command creation, execution (Reader/NonQuery/Scalar), and failures. | No |
| **IDbConnectionInterceptor** | Connection lifecycle: opening, closing, and creating. | No |
| **IDbTransactionInterceptor** | Transaction lifecycle: starting, committing, and rolling back. | No |
| **ISaveChangesInterceptor** | Hooks into `SaveChanges`, failures, and concurrency resolution. | No |
| **IMaterializationInterceptor** | Entity instance creation and initialization from query results. | **Yes** |
| **IQueryExpressionInterceptor**| Modifies the LINQ expression tree before compilation. | **Yes** |
| **IIdentityResolutionInterceptor**| Resolves conflicts when tracking multiple instances with the same key. | **Yes** |

### Code Example: Automatic Timestamps (`ISaveChangesInterceptor`)

Override both `SavingChanges` and `SavingChangesAsync` — EF Core invokes only the matching variant depending on whether `SaveChanges` or
`SaveChangesAsync` was called. An implementation that only overrides the synchronous method silently skips timestamp updates on async save paths.

```csharp
public sealed class TimestampInterceptor(TimeProvider timeProvider)
    : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData, InterceptionResult<int> result)
    {
        SetTimestamps(eventData.Context);
        return result;
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        SetTimestamps(eventData.Context);
        return new ValueTask<InterceptionResult<int>>(result);
    }

    private void SetTimestamps(DbContext? context)
    {
        if (context is null)
        {
            return;
        }

        var now = timeProvider.GetUtcNow();

        foreach (var entry in context.ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = now;
            }
            entry.Entity.UpdatedAt = now;
        }
    }
}
```

Register the interceptor once as a singleton when it is stateless. This avoids repeated allocation and makes the lifetime explicit. The
`ManyServiceProvidersCreatedWarning` concern in [Section 12](#12-gotchas-and-antipatterns) applies specifically to EF singleton interceptors, not to
ordinary per-context interceptors such as this `SaveChangesInterceptor`:

```csharp
// Resolve from the application DI container so the same instance is reused.
services.AddSingleton<TimeProvider>(TimeProvider.System);
services.AddSingleton<TimestampInterceptor>();
services.AddDbContext<AppDbContext>((sp, options) =>
    options.UseSqlServer(connectionString)
           .AddInterceptors(sp.GetRequiredService<TimestampInterceptor>()));
```

### Error Handling in Interceptors

An interceptor runs inside EF Core's operation pipeline. If it throws, the original operation fails and EF reports the interceptor exception to the
caller. Treat interceptor code as production pipeline code, not as a best-effort event handler:

*   **Validate configuration early.** Throw from `IDbContextOptionsExtension.Validate(...)` or application startup when required services, providers,
    or options are missing. Do not discover required configuration lazily in the middle of `SaveChanges` or query execution.
*   **Let correctness failures fail the operation.** If an auditing, tenancy, soft-delete, or security interceptor cannot apply its required rule,
    throw a concise exception rather than allowing partially incorrect data access.
*   **Avoid swallowing EF exceptions.** In failure callbacks such as `SaveChangesFailed`, log or add diagnostics, but do not hide the exception unless
    the interceptor is intentionally suppressing an operation through EF's interception result APIs.
*   **Keep side effects idempotent.** Retried commands or save operations can invoke the interceptor more than once. External side effects such as
    queue messages or HTTP calls should be avoided, guarded by idempotency keys, or moved outside the EF interceptor.
*   **Use structured logging for diagnostics.** In application interceptors, inject `ILogger<TInterceptor>` and prefer `[LoggerMessage]` for repeated log
    messages instead of embedding support-only details in exception text.

For example, a required tenant value should fail before a query or save can proceed:

```csharp
public sealed class TenantValidationInterceptor(ITenantProvider tenantProvider)
    : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        var tenantId = tenantProvider.FindTenantId();

        if (string.IsNullOrWhiteSpace(tenantId))
        {
            var exception = new InvalidOperationException(
                "Apply tenant filter: a tenant id is required.");
            exception.Data["Category"] = "Security";
            exception.Data["AffectedAudience"] = "Application users and tenant data owners";
            throw exception;
        }

        return result;
    }
}

// Register as a scoped interceptor with the tenant provider as a dependency:
services.AddScoped<ITenantProvider, MyTenantProvider>();
services.AddScoped<TenantValidationInterceptor>();
services.AddDbContext<AppDbContext>((sp, options) =>
    options.UseSqlServer(connectionString)
           .AddInterceptors(sp.GetRequiredService<TenantValidationInterceptor>()));
```

---

## 3. Dynamically Extending the Model

### What the Model Is and Why It Matters

In EF Core, the **model** (represented by the `IModel` metadata graph) is the in-memory description of your domain as EF Core understands it. It
catalogs every entity type, its properties, keys, indexes, and relationships, along with how each element maps to database constructs such as tables,
columns, schemas, and constraints. The model is distinct from both the database schema and your CLR classes: it is the metadata layer that binds the
two together.

EF Core consults this model for nearly everything it does at runtime—translating LINQ queries into SQL, materializing query results back into entity
instances, tracking changes for `SaveChanges`, and generating migrations. Because it is the single source of truth that connects your CLR types to the
database, an accurate model is a prerequisite for correct queries and updates.

The model is produced once during the **model building phase**, the first time a `DbContext` of a given type is used. EF Core runs the registered
conventions, applies your `OnModelCreating` configuration, and invokes `IModelCustomizer`, then finalizes the result into an immutable `IModel`.
The finalized model is cached and reused (see [Model Caching and Multiple Models](#model-caching-and-multiple-models)). Because it is read-only, any
structural change must happen while the model is still being built.

Extensions hook into this building phase to modify the model metadata before it is finalized, allowing for automatic multi-tenancy filters, audit
columns, or custom object support without requiring the user to hand-write the configuration. When the required shape varies by runtime or
configuration state, an extension must also influence how models are cached so that each variation gets its own model.

### Intercepting the ModelBuilder
To inject configuration without requiring the user to modify `OnModelCreating`, replace the **`IModelCustomizer`** service. The base class depends on
the provider family:

*   For relational providers, inherit from `RelationalModelCustomizer`. This preserves relational model customization before your extension adds
    relational metadata such as column defaults, table mappings, sequences, or check constraints.
*   For provider-agnostic or non-relational extensions, inherit from the base `ModelCustomizer` and use only provider-agnostic model APIs. Do not call
    relational APIs such as `HasDefaultValueSql`, `ToTable`, or relational annotation helpers unless the configured provider is relational.
*   For a specific non-relational provider, preserve that provider's own customizer if it exposes one you can inherit from. If it does not, prefer a
    convention plugin for provider-neutral model changes rather than replacing `IModelCustomizer`, because replacing the service can accidentally skip
    provider-specific customization.

The high-level model-building order is:

1.  EF creates the initial convention set, including any `IConventionSetPlugin` modifications.
2.  EF discovers entity types and applies conventions as model elements are added or changed.
3.  `IModelCustomizer.Customize` runs. The base implementation calls the context's `OnModelCreating` method.
4.  Your customizer logic runs after `base.Customize(...)` if you follow the pattern below.
5.  EF finalizes and caches the model under the current `IModelCacheKeyFactory` key.

Always call `base.Customize` first so `OnModelCreating` and provider customizer logic run before your extension:

```csharp
public sealed class MyModelCustomizer(ModelCustomizerDependencies dependencies)
    : RelationalModelCustomizer(dependencies)
{
    public override void Customize(ModelBuilder modelBuilder, DbContext context)
    {
        base.Customize(modelBuilder, context); // runs OnModelCreating first

        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            // Example: add a shadow audit column to every entity type.
            modelBuilder.Entity(entityType.ClrType)
                .Property<DateTime>("LastModifiedUtc")
                .HasDefaultValueSql("GETUTCDATE()");
        }
    }
}

// In IDbContextOptionsExtension.ApplyServices (EF Core registers IModelCustomizer as scoped):
services.Replace(ServiceDescriptor.Scoped<IModelCustomizer, MyModelCustomizer>());
```

For a non-relational or provider-agnostic customizer, use `ModelCustomizer` instead:

```csharp
public sealed class MyProviderAgnosticModelCustomizer(ModelCustomizerDependencies dependencies)
    : ModelCustomizer(dependencies)
{
    public override void Customize(ModelBuilder modelBuilder, DbContext context)
    {
        base.Customize(modelBuilder, context);

        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            // Example: add provider-agnostic metadata consumed by your extension later.
            entityType.SetAnnotation("MyExtension:Enabled", true);
        }
    }
}
```

To find out whether the configured provider is relational, prefer checking the options in `Validate(...)` and fail with a clear error before model
building starts. Relational provider extensions derive from `RelationalOptionsExtension`, so their presence means a relational provider was
configured:

```csharp
public void Validate(IDbContextOptions options)
{
    var isRelational = options.Extensions.OfType<RelationalOptionsExtension>().Any();

    if (!isRelational)
    {
        throw new InvalidOperationException(
            "Configure MyExtension: a relational database provider is required.");
    }

    // Success path: no exception means the configured provider supports the extension.
}
```

Inside runtime code that has a `DbContext`, `context.Database.IsRelational()` is also available. Use the options-based check in
`IDbContextOptionsExtension.Validate(...)` for extension configuration validation, and use `Database.IsRelational()` only when you already have a
context instance and need to branch at runtime.

*   **Execution**: `Customize` runs once per unique model cache key, the first time a `DbContext` of that type is instantiated. Subsequent
    instantiations reuse the cached model; see [Model Caching and Multiple Models](#model-caching-and-multiple-models) if your customizer behavior
    must vary with runtime configuration.

### Model Caching and Multiple Models

Model caching is a performance optimization. When EF Core instantiates a `DbContext` and performs the first operation, it builds the structural model
for that context type by running conventions, `IModelCustomizer`, and `OnModelCreating`. Because this is expensive, EF Core caches the resulting model
and reuses it for later context instances.

By default, the model cache assumes that one `DbContext` type has one model. Dynamic model extensions break that assumption when the model shape
varies by runtime or configuration state. You need multiple cached models for the same context type when:

1.  **Multi-tenancy with schema separation**: Different tenants share the same app infrastructure but map to different database schemas, such as
    `tenantA.Users` and `tenantB.Users`.
2.  **Feature toggles or dynamic modules**: Enabled modules change which entity types, properties, tables, or columns exist in the model.
3.  **Dynamic database columns at runtime**: Custom fields are represented as EF model properties or shadow properties.
4.  **Multiple database providers in a shared internal provider**: Running the same context type against PostgreSQL and SQL Server usually gets
    separate internal service providers and therefore separate model caches automatically. A custom cache key is only needed for unusual setups that
    deliberately share the internal provider or otherwise make provider-specific model differences invisible to EF Core's normal provider separation.

To tell EF Core to create and cache multiple models for the same context type, replace the default `IModelCacheKeyFactory` service.

#### 1. Define the Context and State

```csharp
public sealed class MultiTenantDbContext : DbContext
{
    public string TenantId { get; }

    public MultiTenantDbContext(
        DbContextOptions<MultiTenantDbContext> options,
        ITenantService tenantService)
        : base(options)
    {
        TenantId = tenantService.GetCurrentTenantId();
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        if (!string.IsNullOrEmpty(TenantId))
        {
            modelBuilder.HasDefaultSchema(TenantId);
        }
    }
}
```

#### 2. Create a Custom Model Cache Key

Create a custom object that incorporates the dynamic discriminator. EF Core compares these keys via `Equals` and `GetHashCode`. Value tuples are a
good choice here.

```csharp
public sealed class MultitenantModelCacheKeyFactory : IModelCacheKeyFactory
{
    public object Create(DbContext context, bool designTime)
    {
        if (context is MultiTenantDbContext dynamicContext)
        {
            return (context.GetType(), dynamicContext.TenantId, designTime);
        }

        return (context.GetType(), designTime);
    }
}
```

#### 3. Register the Custom Factory

```csharp
builder.Services.AddDbContext<MultiTenantDbContext>(options =>
{
    options.UseSqlServer(connectionString);
    options.ReplaceService<IModelCacheKeyFactory, MultitenantModelCacheKeyFactory>();
});
```

> **Note:** A cached `IModel` consumes memory. Caching tens of models, such as feature-toggle variants, is usually fine. Caching thousands of models,
> such as one model per large SaaS tenant, can consume substantial RAM. Design-time migrations also usually assume one model per context, so you might
> need to manage schema generation externally.

### Model and Metadata Annotations
Extensions frequently need to store custom metadata on model elements, such as entities or properties, to be read later by translators, migrations, or
conventions. EF Core distinguishes between two types of annotations:

1. **Standard (Design-Time) Annotations**: Added via `HasAnnotation` or `AddAnnotation`. These are part of the core model, serialized into compiled
   models (AOT), evaluated during design-time, and made available to `IMigrationsAnnotationProvider` to affect database schema generation. Use these
   for structural configuration, such as column mapping details or flag toggles.

   **What "serialized into compiled models" means:** `dotnet ef dbcontext optimize` generates C# source files that reproduce the entire model without
   running conventions or reflection at startup. Standard annotations appear in those files as `AddAnnotation("key", value)` calls on every annotated
   model element. Because the generator must emit each value as a C# literal, annotation values must be primitives, strings, enums, or types covered
   by a registered `IAnnotationCodeGenerator` (or `IRelationalAnnotationCodeGenerator` for relational providers) that produces the corresponding
   literal expression. Without a matching code generator, a custom-type annotation will either cause the generator to throw or be silently dropped.
   Non-representable values — compiled delegates, service instances, or complex object graphs — belong in runtime annotations, which are never emitted
   to the generated source files.

2. **Runtime Annotations**: Added via `AddRuntimeAnnotation`.
   *never* included in migrations or AOT compiled models. Use these to cache non-serializable objects, such as compiled delegates, instantiated
   service configurations, or fast-path lookup tables, to avoid recomputing them during query execution or `SaveChanges`. **Crucially, because EF Core
   caches the model, runtime annotations outlive the `DbContext` instance and are shared across all contexts. Therefore, any object stored in a
   runtime annotation must be thread-safe.**

**Annotation Examples:**
You can read or write annotations directly through the model builders or metadata objects:
```csharp
// Setting a standard annotation (affects design-time & runtime)
entityTypeBuilder.HasAnnotation("MyExtension:IsAuditable", true);

// Setting a runtime annotation (useful in model finalization steps for execution speed)
entityType.AddRuntimeAnnotation("MyExtension:AuditStrategy", new MyAuditStrategy());

// Reading an annotation downstream
bool isAuditable = entityType.FindAnnotation("MyExtension:IsAuditable")?.Value as bool? ?? false;
```

To provide a strongly-typed and discoverable experience for users or your internal code, the convention is to create C# extension methods on standard
EF Core builder interfaces, such as `EntityTypeBuilder`, or metadata interfaces such as `IReadOnlyEntityType` or `IMutableEntityType`:

```csharp
public static class MyExtensionMetadataBuilderExtensions
{
    // Write extension
    public static EntityTypeBuilder IsAuditable(
        this EntityTypeBuilder builder, 
        bool auditable = true)
    {
        builder.HasAnnotation("MyExtension:IsAuditable", auditable);
        return builder;
    }

    // Read extension
    public static bool IsAuditable(this IReadOnlyEntityType entityType)
        => 
        entityType.FindAnnotation("MyExtension:IsAuditable")?.Value as bool? ?? false;
}
```
This metadata is then queried internally by your `IMigrationsAnnotationProvider` to instruct migrations, or by your custom `IModelCustomizer` to
configure global query filters or shadow properties dynamically.

### Custom Conventions
Conventions react to model changes (e.g., an entity being added). Each convention interface defines a single `Process*` method that EF Core's
convention dispatcher calls when the corresponding model element is added or modified. Implement the interface and apply configuration through the
supplied builder:

```csharp
public sealed class MaxLengthConvention : IPropertyAddedConvention
{
    public void ProcessPropertyAdded(
        IConventionPropertyBuilder propertyBuilder,
        IConventionContext<IConventionPropertyBuilder> context)
    {
        if (propertyBuilder.Metadata.ClrType == typeof(string)
            && propertyBuilder.Metadata.GetMaxLength() is null)
        {
            propertyBuilder.HasMaxLength(256, fromDataAnnotation: false);
        }
    }
}
```

*   **Registration**: To inject conventions without requiring the user to modify `OnModelCreating`, implement **`IConventionSetPlugin`** and register
    it as an internal service in your `IDbContextOptionsExtension.ApplyServices`. EF Core resolves all `IConventionSetPlugin` registrations and calls
    `ModifyConventions` on each:

    ```csharp
    public sealed class MyConventionPlugin : IConventionSetPlugin
    {
        public ConventionSet ModifyConventions(ConventionSet conventionSet)
        {
            conventionSet.PropertyAddedConventions.Add(new MaxLengthConvention());
            return conventionSet;
        }
    }

    // In IDbContextOptionsExtension.ApplyServices:
    services.TryAddEnumerable(
        ServiceDescriptor.Singleton<IConventionSetPlugin, MyConventionPlugin>());
    ```

*   **Precedence**: The hierarchy is
    **Fluent API (Highest) → Data Annotations → Conventions (Lowest)**.

    In convention code, pass `fromDataAnnotation: false` so EF records the value
    as a **convention** value (lowest precedence). Then user configuration can
    override it naturally:

    ```csharp
    // Convention (lowest): applies only as a default.
    propertyBuilder.HasMaxLength(256, fromDataAnnotation: false);

    // User Data Annotation (higher): wins over the convention.
    [MaxLength(100)]
    public string Name { get; set; } = "";

    // User Fluent API (highest): wins over both.
    modelBuilder.Entity<MyEntity>()
        .Property(e => e.Name)
        .HasMaxLength(50);
    ```

---

## 4. Internal vs. External Dependency Injection

EF Core uses a **Two-Container Model** to prevent application services from conflicting with framework services.

*   **Application DI (External)**: Managed by the user (e.g., ASP.NET Core). It handles the `DbContext` lifetime.
*   **Internal DI (Framework)**: Private to EF Core. It resolves services like `IQueryCompiler`.

### Interaction and Bridging
Extensions in the internal DI cannot see services in the application DI by default. To consume an application service:
1.  **Direct Injection**: EF Core automatically bridges common services like `ILoggerFactory` or `IMemoryCache` into the internal provider. Declare
    them as constructor parameters in your internal service; EF Core's DI infrastructure resolves them automatically:

    ```csharp
    public sealed class MyQueryService(ILoggerFactory loggerFactory)
    {
        private readonly ILogger _logger = loggerFactory.CreateLogger<MyQueryService>();
    }
    ```

2.  **Manual Bridging**: For application services not bridged automatically, prefer to inject a DI-created interceptor or options object into the
    `DbContext` constructor and forward that existing instance to EF Core via `OnConfiguring`. Do not create a new **singleton interceptor** instance
    inside `OnConfiguring`. `OnConfiguring` is called once per `DbContext` **instance** — not once per model — so it runs every time a context is
    constructed, regardless of whether the model is already cached. Each call therefore produces a new interceptor object with a different identity.
    EF Core includes singleton interceptor instances (`ISingletonInterceptor` implementations such as `IQueryExpressionInterceptor`,
    `IMaterializationInterceptor`, and `IIdentityResolutionInterceptor`) in the `CoreOptionsExtension` service-provider hash. A new object identity
    means a different hash, which causes a cache miss in `ServiceProviderCache`, which causes EF to build yet another internal service provider.
    Enough misses trigger `ManyServiceProvidersCreatedWarning` and steadily grow memory. Ordinary per-context interceptors (`IDbCommandInterceptor`,
    `ISaveChangesInterceptor`, etc.) are **not** included in the service-provider hash, so allocating them per-instance is merely wasteful, not a
    cache-correctness problem:

    ```csharp
    public sealed class AppDbContext(
        DbContextOptions<AppDbContext> options,
        MyInterceptor interceptor)
        : DbContext(options)
    {
        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
            => 
            optionsBuilder.AddInterceptors(interceptor);
    }
    ```

    Register `MyInterceptor` in application DI with the lifetime appropriate for the services it consumes. Singleton interceptors are best when the
    interceptor has no scoped dependencies. Scoped interceptors are acceptable for context-specific application services, provided the same resolved
    instance is reused for the lifetime of the context.

    For **per-call behavior** — values that must flow from the call site into an interceptor for a single `SaveChanges` / `SaveChangesAsync`
    invocation — use the **Ambient Context** pattern scoped to one `DbContext` instance:

    *   A mutable state object is registered as a **scoped** service inside EF Core's *internal* container (via
        `IDbContextOptionsExtension.ApplyServices`). "Scoped" in EF's internal container means *per `DbContext` instance*, not per HTTP request.
    *   The call site sets state on that object immediately before calling `SaveChanges` / `SaveChangesAsync`.
    *   The interceptor reads state from the same object, which EF Core injects through constructor parameters.
    *   A `finally` block clears state so pooled or long-lived contexts cannot leak it to the next operation.

    **Why not use `AddScoped` in ASP.NET Core's container?** ASP.NET Core's request scope is shared by everything resolved during a single HTTP
    request. If the application creates or resolves multiple `DbContext` instances within the same request — directly or through parallel work — they
    all see the same scoped service instance. State written for one context is visible to the other, causing cross-talk. EF Core's internal scoped
    service is isolated to exactly one `DbContext` instance, so no cross-talk is possible.

    The following example shows the complete pattern: a state object, a `DbContext` helper method that sets and clears the state, an interceptor that
    reads it, and the registration wiring.

    ```csharp
    // 1. State object — one instance per DbContext instance.
    public sealed class SaveChangesState
    {
        public string? OperationTag { get; set; }
    }

    // 2. DbContext extension method — sets state, saves, then clears state.
    public static class AppDbContextExtensions
    {
        public static Task<int> SaveChangesWithTagAsync(
            this AppDbContext context,
            string operationTag,
            CancellationToken cancellationToken = default)
        {
            var state = context.GetService<SaveChangesState>();
            state.OperationTag = operationTag;
            try
            {
                return context.SaveChangesAsync(cancellationToken);
            }
            finally
            {
                state.OperationTag = null; // prevent state leak in pooled contexts
            }
        }
    }

    // 3. Interceptor — reads state set by the call site.
    public sealed class TaggedSaveInterceptor(SaveChangesState saveState)
        : SaveChangesInterceptor
    {
        public override InterceptionResult<int> SavingChanges(
            DbContextEventData eventData, InterceptionResult<int> result)
        {
            if (saveState.OperationTag is { } tag)
            {
                // Use tag — e.g., write an audit entry or set a command tag.
            }
            return result;
        }

        public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
            DbContextEventData eventData,
            InterceptionResult<int> result,
            CancellationToken cancellationToken = default)
        {
            if (saveState.OperationTag is { } tag)
            {
                // Use tag.
            }
            return new ValueTask<InterceptionResult<int>>(result);
        }
    }

    // 4. Registration — both services are scoped to one DbContext instance.
    // Call this inside IDbContextOptionsExtension.ApplyServices.
    services.AddScoped<SaveChangesState>();
    services.AddScoped<TaggedSaveInterceptor>();
    // The interceptor must also be added to DbContextOptions, for example via
    // a UseX method that calls options.AddInterceptors(sp.GetRequiredService<TaggedSaveInterceptor>())
    // where sp is the EF Core internal service provider, not the application one.
    ```

    Because both `SaveChangesState` and `TaggedSaveInterceptor` are resolved from EF Core's internal scoped container, each `DbContext` instance
    receives its own independent pair — the state written by one context cannot be read by another.

3.  **Priority**: Options defined in **`OnConfiguring`** take highest priority and override any settings provided via the application DI. Use
    `OnConfiguring` only for settings specific to the context instance; otherwise it may inadvertently undo host-level configuration.

### Extension Registration
Extensions register themselves by implementing **`IDbContextOptionsExtension`** from `Microsoft.EntityFrameworkCore.Infrastructure`. This is the
low-level contract behind provider methods such as `UseSqlServer()` and library methods such as `UseMyExtension()`. It has three members:

*   **`ApplyServices(IServiceCollection services)`**: Register your custom services into the internal `ServiceCollection` here.
*   **`Validate(IDbContextOptions options)`**: Throw if the extension's configuration is inconsistent with other options.
*   **`Info`** (returns `DbContextOptionsExtensionInfo`): Exposes extension metadata. EF Core uses this metadata for internal service-provider
    caching, diagnostic logging, and debug output.

Common internal service lifetimes are:

| Extension point | Usual lifetime | Notes |
| :--- | :--- | :--- |
| `IConventionSetPlugin` | Singleton | Register with `TryAddEnumerable`; implementations must not store scoped or per-context state. |
| `IModelCustomizer` | Scoped | Replace with a scoped descriptor to match EF Core's registration. |
| `IModelCacheKeyFactory` | Singleton | Keys must be immutable, deterministic, and include `designTime`. |
| `IModelRuntimeInitializer` | Singleton | Runtime annotations or cached objects added here must be thread-safe. |
| Query, method-call, and member translator plugins | Singleton | Use `TryAddEnumerable`; avoid scoped dependencies. |
| Migrations SQL or annotation services | Singleton | Keep generated SQL and annotations deterministic for design-time tooling. |
| Ordinary interceptors (`IDbCommandInterceptor`, `ISaveChangesInterceptor`) | Application-defined | May be per context, scoped, or singleton depending on state and dependencies. |
| Singleton interceptors (`IMaterializationInterceptor`, `IQueryExpressionInterceptor`) | Singleton | Shared by EF internal service providers; never store mutable per-context state. |

### `DbContextOptionsExtensionInfo`

`DbContextOptionsExtensionInfo` is the metadata object attached to an `IDbContextOptionsExtension`. It does not register services itself; it tells EF
Core how to compare extension instances and how to describe them. Keep it nested inside the extension unless the metadata is shared by multiple
extension types.

*   **`IsDatabaseProvider`**
    *   **Purpose**: Identifies whether the extension represents a database provider.
    *   **Implementation guidance**: Return `true` only for provider extensions.
        Most add-on extensions return `false`.
*   **`LogFragment`**
    *   **Purpose**: Adds a short fragment to EF Core option logs.
    *   **Implementation guidance**: Keep it stable and concise, such as
        `"using MyExtension "`. Include important user-visible options only.
*   **`GetServiceProviderHashCode()`**
    *   **Purpose**: Contributes to EF Core's internal service-provider cache key.
    *   **Implementation guidance**: Include only values that change services
        registered in `ApplyServices`; do not include per-request state.
*   **`ShouldUseSameServiceProvider(DbContextOptionsExtensionInfo other)`**
    *   **Purpose**: Performs equality for internal service-provider reuse.
    *   **Implementation guidance**: Compare the same service-shaping values
        used by `GetServiceProviderHashCode()`.
*   **`PopulateDebugInfo(IDictionary<string, string> debugInfo)`**
    *   **Purpose**: Adds deterministic debug entries used by diagnostics and
        service-provider-cache validation.
    *   **Implementation guidance**: Use unique keys, usually prefixed with the
        extension name, and stable string values.

The rule is: **service-shaping state belongs in `GetServiceProviderHashCode()` and `ShouldUseSameServiceProvider()`; model-shaping state also belongs
in the model cache key; runtime state belongs in neither.** For example, a flag that decides whether `ApplyServices` registers `SqlRewriter` must be
represented in `DbContextOptionsExtensionInfo`. A tenant id read per request must not be represented there, or EF Core will create unnecessary
internal service providers.

Use these rules when implementing the two service-provider comparison methods:

*   Treat `GetServiceProviderHashCode()` and `ShouldUseSameServiceProvider(...)` as a matched pair. If
    `ShouldUseSameServiceProvider(...)` can return `true` for two extension infos, both infos should return the same service-provider hash code.
*   Put every service-shaping option in both methods. Examples include provider selection, compatibility level, SQL translation mode, a flag that
    conditionally registers a service in `ApplyServices`, or the concrete implementation type used for an internal service.
*   Exclude options that do not change the internal service provider. Examples include command timeout, logging text, tenant id, current user id,
    request correlation id, or values read later from scoped application services.
*   Compare values semantically, not by accident. For strings, use `StringComparer.Ordinal`; for types, compare the `Type` value; for collections,
    compare normalized content in a stable order rather than collection object identity.
*   Avoid including service instances unless the instance itself changes the internal service graph. Prefer storing a service type, enum, or immutable
    option value instead. Instance identity is exactly what causes cache bloat when a new singleton interceptor is created for each options build.
*   Hash collisions are allowed, but equality must be precise. `GetServiceProviderHashCode()` is a fast bucket selector; EF still calls
    `ShouldUseSameServiceProvider(...)` to decide whether a cached provider can actually be reused.
*   Keep `PopulateDebugInfo(...)` aligned with the same service-shaping state. It is not the cache key, but it makes cache-key differences visible in
    diagnostics.

A minimal skeleton:

```csharp
public sealed class MyExtension : IDbContextOptionsExtension
{
    private DbContextOptionsExtensionInfo? _info;

    public DbContextOptionsExtensionInfo Info
        => 
        _info ??= new ExtensionInfo(this);

    public void ApplyServices(IServiceCollection services)
        => 
        services.TryAddSingleton<IMyService, MyService>();

    public void Validate(IDbContextOptions options) { }

    private sealed class ExtensionInfo(IDbContextOptionsExtension extension)
        : DbContextOptionsExtensionInfo(extension)
    {
        public override bool IsDatabaseProvider => false;
        public override string LogFragment => "using MyExtension ";
        public override int GetServiceProviderHashCode() => 0; // constant: no config state
        public override bool ShouldUseSameServiceProvider(
            DbContextOptionsExtensionInfo other) => other is ExtensionInfo;
        public override void PopulateDebugInfo(
            IDictionary<string, string> debugInfo)
            => 
            debugInfo[nameof(MyExtension)] = "1";
    }
}

// Expose via a UseX extension method.
public static DbContextOptionsBuilder UseMyExtension(
    this DbContextOptionsBuilder optionsBuilder)
{
    if (optionsBuilder.Options.FindExtension<MyExtension>() is null)
    {
        ((IDbContextOptionsBuilderInfrastructure)optionsBuilder)
            .AddOrUpdateExtension(new MyExtension());
    }
    return optionsBuilder;
}
```

When the extension has options, make the extension immutable and expose `With...` methods that clone it with one changed value. This matches EF
Core's own options-extension pattern and prevents accidental mutation after an options snapshot has been created:

```csharp
public sealed class MyExtension : IDbContextOptionsExtension
{
    private DbContextOptionsExtensionInfo? _info;

    public bool EnableSqlRewriting { get; private init; }

    public DbContextOptionsExtensionInfo Info
        => 
        _info ??= new ExtensionInfo(this);

    public MyExtension WithSqlRewriting(bool enabled)
        => 
        new MyExtension { EnableSqlRewriting = enabled };

    public void ApplyServices(IServiceCollection services)
    {
        if (EnableSqlRewriting)
        {
            services.TryAddSingleton<SqlRewriter>();
        }
    }

    public void Validate(IDbContextOptions options) { }

    private sealed class ExtensionInfo(MyExtension extension)
        : DbContextOptionsExtensionInfo(extension)
    {
        private readonly MyExtension _extension = extension;

        public override bool IsDatabaseProvider => false;

        public override string LogFragment
            => 
            _extension.EnableSqlRewriting ? "using MyExtension:SqlRewriting " : "using MyExtension ";

        public override int GetServiceProviderHashCode()
            => 
            HashCode.Combine(_extension.EnableSqlRewriting);

        public override bool ShouldUseSameServiceProvider(DbContextOptionsExtensionInfo other)
            => 
            other is ExtensionInfo otherInfo
                && _extension.EnableSqlRewriting == otherInfo._extension.EnableSqlRewriting;

        public override void PopulateDebugInfo(IDictionary<string, string> debugInfo)
            => 
            debugInfo["MyExtension:EnableSqlRewriting"] = _extension.EnableSqlRewriting.ToString();
    }
}
```

If a `UseX` method changes extension options, clone the existing extension instead of mutating it:

```csharp
public static DbContextOptionsBuilder UseMyExtension(
    this DbContextOptionsBuilder optionsBuilder,
    bool enableSqlRewriting)
{
    var extension = optionsBuilder.Options.FindExtension<MyExtension>() ?? new MyExtension();
    extension = extension.WithSqlRewriting(enableSqlRewriting);

    ((IDbContextOptionsBuilderInfrastructure)optionsBuilder)
        .AddOrUpdateExtension(extension);

    return optionsBuilder;
}
```

---

## 5. Logging and Diagnostics

For low-level EF Core extension/provider services, use the framework's internal `IDiagnosticsLogger<TCategoryName>` so diagnostics align with EF Core's
event model and integrate with `DbContextOptions.LogTo()` and `DiagnosticSource` telemetry. For application interceptors, continue using standard
`ILogger<TInterceptor>` from DI.

### Using `IDiagnosticsLogger<T>`
Inject the logger using one of the pre-defined categories in `DbLoggerCategory`. For instance,
`IDiagnosticsLogger<DbLoggerCategory.Database.Command>` or `IDiagnosticsLogger<DbLoggerCategory.Infrastructure>`:

```csharp
public sealed class MyExtensionService(
    IDiagnosticsLogger<DbLoggerCategory.Infrastructure> logger)
{
    public void DoWork()
    {
        // Logs directly to EF Core's configured logging stream
        logger.Logger.LogInformation("MyExtension is performing work.");
    }
}
```

### Which Logger to Use

Choose the logger type based on where the code runs:

*   **Application interceptors** (`IDbCommandInterceptor`, `ISaveChangesInterceptor`, etc.): use `ILogger<TInterceptor>` and optional
    `[LoggerMessage]` source-generated methods.
*   **EF Core extension/provider internals** (translators, model validators, SQL generators, extension services): use
    `IDiagnosticsLogger<TCategoryName>`.

`IDiagnosticsLogger<TCategoryName>` is not just an `ILogger` replacement; it also carries EF diagnostics context such as event definitions and
`DiagnosticSource` access. This is why EF internal extension points often expose it directly in method signatures.

### Category Selection Tips

Use the closest `DbLoggerCategory` to the feature being implemented:

*   `DbLoggerCategory.Database.Command` for command/SQL execution work.
*   `DbLoggerCategory.Query` for translation and query pipeline behavior.
*   `DbLoggerCategory.Model` or `DbLoggerCategory.Model.Validation` for model building/validation.
*   `DbLoggerCategory.Infrastructure` for provider/extension infrastructure and setup paths.

Matching categories keeps log filtering and `LogTo` output predictable for consumers.

### Emitting Telemetry
To support APM tools like OpenTelemetry or Application Insights, emit events through `DiagnosticSource`:

```csharp
if (logger.DiagnosticSource.IsEnabled("MyExtension.EventName"))
{
    logger.DiagnosticSource.Write("MyExtension.EventName", new { Payload = "data" });
}
```

---

## 6. Query Translation (LINQ to SQL Customization)

Custom SQL translators are only one query-extensibility option. Depending on the shape of the customization, you can:

*   **Translate directly to SQL** by implementing and registering **`IMethodCallTranslatorPlugin`** or **`IMemberTranslatorPlugin`** when the
    method/property has a provider-specific SQL representation (for example, `string.StartsWith()` or a custom database function).
*   **Substitute a method call or property access with another LINQ expression tree** using **`IQueryExpressionInterceptor`** when the helper is only
    a reusable .NET facade over logic EF can already translate. Rather than substituting expressions manually, the interceptor expands the helper's
    body into standard query nodes before translation—without inventing a new SQL construct.

Choose the lightest mechanism that fits:

*   If the target database needs a new SQL expression, use a translator plugin.
*   If the logic can already be expressed in normal LINQ over mapped members, prefer expression substitution/inlining via `IQueryExpressionInterceptor`.

### Example: Method Call Translation
Implement `IMethodCallTranslator` to return a `SqlFunctionExpression` or other `SqlExpression` representing the operation, then expose it via a
plugin:

```csharp
public sealed class MyMethodTranslatorPlugin(
    ISqlExpressionFactory sqlExpressionFactory) : IMethodCallTranslatorPlugin
{
    public IEnumerable<IMethodCallTranslator> Translators { get; } 
        = [new MyMethodTranslator(sqlExpressionFactory)];
}

public sealed class MyMethodTranslator(ISqlExpressionFactory sqlExpressionFactory)
    : IMethodCallTranslator
{
    public SqlExpression? Translate(
        SqlExpression? instance,
        MethodInfo method,
        IReadOnlyList<SqlExpression> arguments,
        IDiagnosticsLogger<DbLoggerCategory.Query> logger)
    {
        if (method.Name == "MyCustomDbFunction")
        {
            return sqlExpressionFactory.Function(
                name: "MyCustomDbFunction",
                schema: "dbo",
                arguments: arguments,
                nullable: true,
                argumentsPropagateNullability: [true],
                returnType: method.ReturnType);
        }
        return null;
    }
}
```

Register the plugin in your options extension:

```csharp
services.TryAddEnumerable(ServiceDescriptor.Singleton<IMethodCallTranslatorPlugin, MyMethodTranslatorPlugin>());
```

---

## 7. Custom Type Mapping

To map custom structs, classes, or domain-specific primitives (e.g., Strongly-Typed IDs, NodaTime, JSON wrapper types, provider-specific UDTs) to
database columns deeply, you can override relational type mappings.

Most applications should start with a **`ValueConverter`**. A converter is enough when you only need to transform one CLR value into another CLR value
that EF and the provider already understand (for example, `OrderId` <-> `Guid`, or `Money` <-> `decimal`). Reach for custom relational type mapping
only when you also need to control one or more of the following:

```csharp
modelBuilder.Entity<Order>()
    .Property(x => x.OrderId)
    .HasConversion(
        id => id.Value,
        value => new OrderId(value));
```

*   the exact store type name (`jsonb`, `geography`, `hierarchyid`, etc.)
*   ADO.NET parameter configuration such as `DbType` or provider-specific parameter metadata
*   SQL literal generation for migrations, default values, or inline constants
*   provider-specific read/write behavior that a converter alone cannot express

The extension point is **`IRelationalTypeMappingSourcePlugin`**. EF asks every registered plugin whether it can satisfy a mapping request. In your
plugin:

*   inspect the incoming mapping request (`ClrType`, store type name, Unicode/fixed-length/size/precision facets, and similar metadata)
*   return `null` for requests you do not own so the provider's normal mapping pipeline can continue
*   return a **`RelationalTypeMapping`** (or a more specific base such as the string, integer, or GUID relational mappings) for requests you do own

Your custom **`RelationalTypeMapping`** is responsible for the relational details of the type, such as:

*   store type name and facets
*   the `CoreTypeMapping` pieces attached to it, including any `ValueConverter` and `ValueComparer`
*   parameter creation/configuration for ADO.NET
*   SQL literal rendering for non-parameterized constants
*   cloning itself when EF needs the same mapping with different facets

That last point is important: EF often reuses the same conceptual mapping with different sizes, precisions, or store-type names. A robust mapping type
must preserve its behavior when cloned with updated parameters rather than assuming one fixed instance forever.

Register the plugin exactly like the query translator:
```csharp
services.TryAddEnumerable(ServiceDescriptor.Singleton<IRelationalTypeMappingSourcePlugin, MyTypeMappingPlugin>());
```

In practice, provider/package authors use this extension point far more often than ordinary application code. If your goal is only to make one model
property persist cleanly, prefer `.HasConversion(...)` or a value converter first. Use a full type-mapping plugin when you are effectively introducing
or formalizing a database type for the provider.

---

## 8. Migrations and SQL Generation

When an extension introduces custom annotations on the model, you usually need custom schema generation logic to persist those changes to the
database.

1.  **`IMigrationsAnnotationProvider`**: Inherit the provider-specific implementation (e.g., `SqlServerMigrationsAnnotationProvider`) to pass specific
    metadata annotations down to the migrations pipeline.
2.  **`IMigrationsSqlGenerator`**: Inherit the database-specific SQL generator (e.g., `SqlServerMigrationsSqlGenerator`) and override logical
    operations like `Generate(CreateTableOperation operation, ...)`. Read your custom annotations from the operation and append the raw SQL to the
    `MigrationCommandListBuilder`.

For application code, register targeted replacements with `.ReplaceService` in options configuration:

```csharp
optionsBuilder.ReplaceService<IMigrationsSqlGenerator, MyMigrationsSqlGenerator>();
optionsBuilder.ReplaceService<IMigrationsAnnotationProvider, MyMigrationsAnnotationProvider>();
```

For a packaged `IDbContextOptionsExtension`, do this in `ApplyServices`, which works with `IServiceCollection`. Register or replace internal services
there with service-collection APIs:

```csharp
services.Replace(
    ServiceDescriptor.Singleton<IMigrationsSqlGenerator, MyMigrationsSqlGenerator>());
services.Replace(
    ServiceDescriptor.Singleton<IMigrationsAnnotationProvider, MyMigrationsAnnotationProvider>());
```

---

## 9. Design-Time vs. Run-Time

EF Core does not behave identically when you execute application code and when you run tooling. It distinguishes two **modes**, and the active mode
changes which model EF builds, which services are available, and how your extension must behave. Treating both modes the same is one of the most
common sources of extension bugs that only surface when a user runs `dotnet ef ...`.

*   **Run-time mode** is the normal application path: resolving a `DbContext` from DI, executing queries, and calling `SaveChanges`. EF optimizes for
    speed and low memory here.
*   **Design-time mode** is the tooling path: `dotnet ef migrations add`, `dotnet ef migrations script`, `dotnet ef dbcontext scaffold`,
    `dotnet ef dbcontext optimize`, and `dotnet ef database update`, plus the equivalent Package Manager Console (PMC) commands. The EF tools load
    your startup project in a separate process, construct the `DbContext` (often through `IDesignTimeDbContextFactory<TContext>`), and inspect or
    transform the model without necessarily connecting to a live database. (Source: design-time context creation,
    <https://learn.microsoft.com/ef/core/cli/dbcontext-creation>; design-time services, <https://learn.microsoft.com/ef/core/cli/services>.)

### The `designTime` Flag and the Two Model Flavors

EF Core builds the model in two flavors and caches them **separately**:

1.  **Design-time model**: the complete model, retaining every annotation, relational schema detail, and convention output that tooling needs to
    generate correct migrations, SQL scripts, and scaffolded code.
2.  **Runtime model**: an optimized, trimmed model used while the application runs. Conventions are not re-run, reflection is minimized, and
    runtime-only state (compiled delegates, fast-path lookups) is applied. Design-time-only data may be dropped to save memory.

A `bool designTime` parameter flows through the services that participate in model construction so each one can produce the correct flavor:

*   `IModelCacheKeyFactory.Create(DbContext context, bool designTime)` — EF caches the design-time and runtime models under different keys. The
    default `ModelCacheKey` already incorporates `designTime`, so any custom key **must** include it as well. Otherwise tooling can receive the
    optimized runtime model (missing schema metadata), or the application can receive the heavier design-time model.
*   `IModelRuntimeInitializer.Initialize(IModel model, bool designTime, ...)` — when `designTime` is `true`, EF keeps the full model for tooling; when
    `false`, it finalizes the optimized runtime model. Apply runtime-only metadata only on the runtime path (guard it with `if (!designTime)`),
    because that state is neither serializable into compiled models nor needed by tooling.

(EF Core source: `src/EFCore/Infrastructure/ModelCacheKeyFactory.cs` and `src/EFCore/Infrastructure/ModelRuntimeInitializer.cs` in the
[dotnet/efcore repository](https://github.com/dotnet/efcore).)

### Why Design-Time Mode Is Needed

*   **Tooling needs the full model.** Migrations and scaffolding must see all relational schema metadata — tables, columns, keys, check constraints,
    and `IMigrationsAnnotationProvider` annotations. The runtime model intentionally drops much of this for performance, so reusing it for tooling
    would produce incomplete or incorrect migrations.
*   **Runtime needs a lean model.** Applications pay a startup and memory cost for every piece of metadata kept on the model. Stripping
    design-time-only data and precomputing runtime state keeps queries and `SaveChanges` fast; this is also what `dotnet ef dbcontext optimize`
    (compiled models) bakes in ahead of time.
*   **Different host and lifetime.** Design-time runs in the EF tools process, not your application host. The app's DI container, configuration, and
    ambient services (HTTP context, current user, a live tenant resolver) may be absent. EF may construct the context through
    `IDesignTimeDbContextFactory<TContext>` instead of your normal composition root.
*   **No guaranteed database.** Generating a migration or scaffolding code must succeed without connecting to the target database, so model building
    cannot depend on querying it.

### Design-Time Services (`IDesignTimeServices`)

Tasks like generating code for `dotnet ef dbcontext scaffold` or generating migrations require services that exist **only** in the design-time service
collection. In an application startup project, provide a class implementing **`IDesignTimeServices`**; EF tooling discovers it from the startup
assembly and runs it. Provider and reusable extension assemblies usually need explicit design-time service discovery wiring, such as the provider's
design-time services attribute or package-specific tooling integration, instead of relying on arbitrary assembly scanning.

```csharp
public sealed class MyExtensionDesignTimeServices : IDesignTimeServices
{
    public void ConfigureDesignTimeServices(IServiceCollection services)
    {
        // Add services used ONLY by dotnet ef CLI and PMC paths, 
        // such as an IProviderConfigurationCodeGenerator.
        services.AddSingleton<IMyDesignOnlyService, MyDesignOnlyService>();
    }
}
```

### How Your Extension Should Account for Design-Time Mode

*   **Thread the `designTime` flag through.** Honor it in `IModelCacheKeyFactory` (include it in the cache key) and in `IModelRuntimeInitializer`
    (apply runtime-only state only when `!designTime`). See [Designing AOT-Friendly Extensions](#11-designing-aot-friendly-extensions) for the
    initializer pattern.
*   **Put schema-affecting configuration in standard annotations.** Standard annotations are serialized into compiled models and exposed to
    `IMigrationsAnnotationProvider`; runtime annotations are not. See [Model and Metadata Annotations](#model-and-metadata-annotations).
*   **Keep model building deterministic.** Do not vary the model by ambient runtime state such as the current time (`TimeProvider`), environment, or a
    live tenant. Tooling must produce a stable, canonical model, so dynamic models need a sensible design-time default. The `designTime` flag in the
    cache key keeps the canonical tooling model separate from per-request runtime models.
*   **Never open a database connection during model building.** Conventions, `OnModelCreating`, and runtime initialization all run at design time
    without a guaranteed database; connecting there breaks migrations and scaffolding.
*   **Make the context constructible at design time.** If your extension requires services the tools process cannot supply, provide an
    `IDesignTimeDbContextFactory<TContext>` (or document one) so EF can build the context for tooling without your full application host.
*   **Register code generation through `IDesignTimeServices`.** Keep CLI/PMC-only services (such as `IProviderConfigurationCodeGenerator`) out of
    `ApplyServices` and in the design-time collection, and ensure discovery wiring for provider and library assemblies.

---

## 10. Interoperability and Extension Safety

To ensure your extension does not break others:

### `Add*` vs `TryAdd*` in `ApplyServices`

Use this decision matrix when registering internal services:

| Registration API | Use when | Why |
| :--- | :--- | :--- |
| `TryAdd*` (`TryAddSingleton`, `TryAddScoped`, etc.) | Registering a default implementation for a single service contract. | Keeps first registration and avoids overriding another extension unexpectedly. |
| `TryAddEnumerable` | Registering plug-in style multi-service contracts (for example, `IConventionSetPlugin`). | Adds your implementation only if the same implementation type is not already present. |
| `Add*` (`AddSingleton`, etc.) | You intentionally want another registration, or you own the entire internal provider composition. | Always appends a registration and can change behavior by ordering or by last-registration wins resolution. |

Practical guidance:

*   In reusable libraries, prefer `TryAdd*` / `TryAddEnumerable` by default.
*   Use `Add*` only when you intentionally require replacement/stacking semantics.
*   For application-level targeted replacement, prefer `ReplaceService<TService, TImplementation>()` in options configuration.

Example for a multi-registration contract:

```csharp
// Correct: preserves any IConventionSetPlugin already registered by another extension.
services.TryAddEnumerable(
    ServiceDescriptor.Singleton<IConventionSetPlugin, MyConventionPlugin>());

// Use only when intentional: always adds another registration.
services.AddSingleton<IConventionSetPlugin, MyConventionPlugin>();
```

*   **Inherit, Don't Replace**: Instead of providing a fresh implementation of a core service, inherit from the framework's base class (e.g.,
    `RelationalModelValidator`) and call `base` methods. This ensures the framework's own logic and any prior extension's work continue to run:

    ```csharp
    public sealed class MyModelValidator(
        ModelValidatorDependencies dependencies,
        RelationalModelValidatorDependencies relationalDependencies)
        : RelationalModelValidator(dependencies, relationalDependencies)
    {
        public override void Validate(
            IModel model,
            IDiagnosticsLogger<DbLoggerCategory.Model.Validation> logger)
        {
            base.Validate(model, logger); // run all existing validations first
            // Add extension-specific checks here.
        }
    }

    // In ApplyServices:
    services.Replace(ServiceDescriptor.Singleton<IModelValidator, MyModelValidator>());
    ```

*   **Check for Presence**: Use `optionsBuilder.Options.FindExtension<T>()` in your `UseX` extension method to avoid duplicate registrations, as
    demonstrated in the `UseMyExtension` example in Section 4.

*   **Provider Awareness**: Avoid brittle provider-name or type-name string checks when an extension supports only one database provider. Prefer a
    provider-specific `UseX` method, provider-specific service registration, or a direct dependency on the provider package. The practical place to
    enforce provider requirements is `Validate`, where all registered options extensions are available. If you intentionally depend on a provider
    extension type, keep the check explicit and document that it uses provider internals:

    ```csharp
    public void Validate(IDbContextOptions options)
    {
        if (options.FindExtension<SqlServerOptionsExtension>() is null)
        {
            throw new InvalidOperationException(
                "MyExtension requires the SQL Server provider.");
        }
    }
    ```

    The example above requires referencing the SQL Server provider extension type from its provider package. Avoid reflection-based checks such as
    `e.GetType().Name == "SqlServerOptionsExtension"`; they are implementation-sensitive and can break silently.

---

## 11. Designing AOT-Friendly Extensions

With EF Core 10's push for **Ahead-of-Time (AOT)** compatibility, extensions must follow strict rules:

*   **Eliminate Runtime Code Generation**: Avoid `Expression.Compile()` or dynamic IL emission at runtime; both fail under AOT. Replace
    runtime-compiled delegates with static method references or pre-compiled lambdas built once during startup:

    ```csharp
    // Before: fails under AOT — compiled dynamically at runtime.
    var param = Expression.Parameter(typeof(MyEntity));
    Func<MyEntity, int> getter = Expression.Lambda<Func<MyEntity, int>>(
        Expression.Property(param, "Id"), param).Compile();

    // After: AOT-safe static delegate.
    static int GetId(MyEntity e) => e.Id;
    ```

*   **Support Compiled Models**: Ensure your extension is compatible with **Compiled Models** (`dotnet ef dbcontext optimize`), which pre-generate the
    runtime model to eliminate convention runs and reflection at startup. Standard model metadata added during model building, such as annotations,
    entity types, properties, and keys, is captured in compiled models. Use **`IModelRuntimeInitializer`** only for runtime-only metadata or
    non-serialized state that your extension must reconstruct after EF loads the compiled model:

    ```csharp
    public sealed class MyModelRuntimeInitializer(
        ModelRuntimeInitializerDependencies dependencies)
        : ModelRuntimeInitializer(dependencies)
    {
        public override IModel Initialize(
            IModel model,
            bool designTime,
            IDiagnosticsLogger<DbLoggerCategory.Model>? logger)
        {
            model = base.Initialize(model, designTime, logger);
            // Re-apply runtime-only metadata your extension requires.
            return model;
        }
    }

    // In ApplyServices:
    services.AddSingleton<IModelRuntimeInitializer, MyModelRuntimeInitializer>();
    ```

*   **Support Compiled Relational Models**: For relational extensions, ensure all database-specific metadata your extension contributes (table
    mappings, column annotations, check constraints) is resolved during the model-building phase so that `dotnet ef dbcontext optimize` captures it in
    the generated files. Apply runtime-only relational metadata by inheriting from `RelationalModelRuntimeInitializer` and registering it in
    `ApplyServices`, following the same pattern as `IModelRuntimeInitializer` above.

*   **Trimmer Compatibility**: Avoid heavy reflection that might cause the .NET Trimmer to remove code required by your extension at runtime. Where
    reflection is unavoidable, annotate the API with `[DynamicallyAccessedMembers]` to communicate exactly which members the trimmer must preserve,
    then verify with `<PublishTrimmed>true</PublishTrimmed>` in a test publish:

    ```csharp
    public object? Materialize(
        [DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors)]
        Type entityType)
        => Activator.CreateInstance(entityType);
    ```

    The trimmer performs dataflow analysis, so propagate the attribute at every level where the type is passed as a parameter; annotating only the
    outermost entry point is not sufficient.

*   **Prefer explicit metadata over runtime discovery.** Reflection-based conventions that scan every CLR type, property, or method at startup are
    fragile under trimming and can be skipped entirely when a compiled model is used. Capture structural decisions as standard model annotations
    during model building, then consume those annotations from translators, migrations services, or runtime initializers.

*   **Treat shadow properties as structural model data.** Shadow properties are compatible with compiled models when they are added during model
    building with stable names and types. Do not add them from ambient runtime state after the model is finalized; instead, include the discriminator
    in the model cache key or expose explicit configuration APIs so compiled models and migrations see the same structure.

*   **Test the publish path.** A reusable extension should have at least one sample or test application that runs `dotnet ef dbcontext optimize` and a
    trimmed publish. This validates both the design-time model and the runtime model that AOT applications will use.

---

## 12. Gotchas and Antipatterns

*   **Service Provider Bloat**: EF Core caches one internal service provider per unique service-provider-affecting option set. Do not confuse this
    with merely allocating an ordinary interceptor. For example, the official EF Core interceptor documentation shows a command interceptor created
    inline (source: <https://learn.microsoft.com/ef/core/logging-events-diagnostics/interceptors>):

    ```csharp
    public class ExampleContext : BlogsContext
    {
        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
            => 
            optionsBuilder.AddInterceptors(new TaggedQueryCommandInterceptor());
    }
    ```

    That pattern is acceptable for ordinary per-context interceptors such as `IDbCommandInterceptor`, `IDbConnectionInterceptor`,
    `IDbTransactionInterceptor`, and `ISaveChangesInterceptor`, especially when the interceptor is lightweight or intentionally holds context-specific
    state.

    The dangerous case is an EF **singleton interceptor** (`ISingletonInterceptor`), such as `IQueryExpressionInterceptor`,
    `IMaterializationInterceptor`, or `IIdentityResolutionInterceptor`. EF's internal `ServiceProviderCache` is keyed by the completed
    `IDbContextOptions`. The options hash is derived from each registered `IDbContextOptionsExtension.Info.GetServiceProviderHashCode()` value, and
    equality is checked with `IDbContextOptionsExtension.Info.ShouldUseSameServiceProvider(...)`. Ordinary interceptors are not included in that
    service-provider hash. Singleton interceptors are included because EF resolves them from the internal service provider and must know whether the
    provider can be reused.

    In practice, `CoreOptionsExtension` includes the singleton-interceptor instances in its service-provider-affecting comparison. If you call
    `new MyQueryExpressionInterceptor()` each time options are built, each interceptor object has a different identity, so the options extension no
    longer compares as equivalent to previous options. EF then misses the internal provider cache, builds another internal service provider, and after
    enough misses logs `ManyServiceProvidersCreatedWarning`. Relevant EF Core source files: `src/EFCore/Internal/ServiceProviderCache.cs`,
    `src/EFCore/DbContextOptions.cs`, and `src/EFCore/Infrastructure/Internal/CoreOptionsExtension.cs` in the
    [dotnet/efcore repository](https://github.com/dotnet/efcore).

    To surface this problem immediately during development rather than waiting for the warning to appear in logs, configure EF Core to throw an
    exception whenever `ManyServiceProvidersCreatedWarning` fires:

    ```csharp
    services.AddDbContext((sp, options) =>
        options.UseSqlServer(connectionString)
               .ConfigureWarnings(w =>
                   w.Throw(CoreEventId.ManyServiceProvidersCreatedWarning)));
    ```

    With this in place, the first context construction that would push the cache over the threshold throws an `InvalidOperationException` with a
    message that identifies the option differences causing the bloat. Treat this as a test-environment default; it keeps the antipattern from
    silently accumulating in staging before reaching production. In production, `Ignore` or `Log` is more appropriate unless you have automated
    alerting on the log event.

    To discover whether a specific option, extension, or interceptor affects the service-provider cache key:

    1.  Check whether the type is registered through an `IDbContextOptionsExtension`. EF asks every extension's `Info` object for
        `GetServiceProviderHashCode()` and `ShouldUseSameServiceProvider(...)`; any state used by those methods affects internal provider reuse.
    2.  For EF built-in options, inspect the relevant `ExtensionInfo` implementation in the EF Core source. For general options and interceptors,
        start with `CoreOptionsExtension.ExtensionInfo`; for relational provider options, inspect the provider's options extension, such as
        `SqlServerOptionsExtension.ExtensionInfo` or `SqliteOptionsExtension.ExtensionInfo`.
    3.  For interceptors, check whether the interceptor interface derives from `ISingletonInterceptor`. Singleton interceptors are service-provider
        affecting; ordinary interceptors are not. When in doubt, inspect `CoreOptionsExtension.ExtensionInfo` and search for the collection where the
        interceptor is stored.
    4.  For custom extensions, include only service-registration-affecting state in `GetServiceProviderHashCode()` and
        `ShouldUseSameServiceProvider(...)`. If a value changes the services registered in `ApplyServices`, it belongs in the service-provider cache
        key. If it only affects runtime behavior after the provider is built, it usually does not. Use the implementation rules in
        [`DbContextOptionsExtensionInfo`](#dbcontextoptionsextensioninfo) to keep hash and equality behavior consistent.
    5.  During development, configure EF to fail fast on this warning (see the snippet introduced above). Treat
        `ManyServiceProvidersCreatedWarning` as evidence that some service-provider-affecting option varies across otherwise equivalent context
        configurations.

    Wrong:

    ```csharp
    services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer(connectionString)
               .AddInterceptors(new MyQueryExpressionInterceptor()));

    public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
        : DbContext(options)
    {
        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
            => optionsBuilder.AddInterceptors(new MyQueryExpressionInterceptor());
    }
    ```

    Correct:

    ```csharp
    services.AddSingleton<MyQueryExpressionInterceptor>();
    services.AddDbContext<AppDbContext>((sp, options) =>
        options.UseSqlServer(connectionString)
               .AddInterceptors(sp.GetRequiredService<MyQueryExpressionInterceptor>()));
    ```

    Reuse one singleton interceptor instance unless the interceptor is one of the ordinary per-context interceptor types and intentionally needs
    per-context state.

*   **Breaking the Model Cache**: If your extension modifies the model dynamically (e.g., based on a flag stored in `MyExtension`), you **must**
    replace `IModelCacheKeyFactory` to incorporate that state into the cache key. Without this, all contexts sharing the same `DbContext` type share
    one cached model regardless of the flag:

    ```csharp
    public sealed class MyModelCacheKeyFactory : IModelCacheKeyFactory
    {
        public object Create(DbContext context, bool designTime)
        {
            var flag = context.GetService<IDbContextOptions>()
                .FindExtension<MyExtension>()?.Flag ?? false;
            return (context.GetType(), designTime, flag);
        }
    }

    // In ApplyServices:
    services.Replace(ServiceDescriptor.Singleton<IModelCacheKeyFactory, MyModelCacheKeyFactory>());
    ```

*   **Thread Safety**: Singleton interceptors such as `IMaterializationInterceptor` are shared across all contexts and threads. Never store mutable
    per-request or per-context state in their instance fields, because concurrent contexts would race on it. Read what you need from the arguments EF
    passes to each interception method instead — `IMaterializationInterceptor` methods receive a `MaterializationInterceptionData` value that exposes
    the `IEntityType` being materialized and its property values. When an interception method needs a scoped service, resolve it from the active
    context rather than caching it: methods whose event data derives from `DbContextEventData` expose `Context`, so call
    `eventData.Context.GetService<ITenantProvider>()` inside the method body rather than storing a `_tenantId` field on the interceptor.

    Wrong:

    ```csharp
    public sealed class BadMaterializationInterceptor : IMaterializationInterceptor
    {
        private string? _tenantId;

        public object InitializedInstance(
            MaterializationInterceptionData materializationData,
            object entity)
        {
            _tenantId = materializationData.Context?.GetService<ITenantProvider>()
                .FindTenantId();
            return entity;
        }
    }
    ```

    Correct:

    ```csharp
    public sealed class TenantAwareMaterializationInterceptor : IMaterializationInterceptor
    {
        public object InitializedInstance(
            MaterializationInterceptionData materializationData,
            object entity)
        {
            var tenantId = materializationData.Context?.GetService<ITenantProvider>()
                .FindTenantId();

            if (entity is ITenantScoped tenantScoped)
            {
                tenantScoped.TenantId = tenantId;
            }

            return entity;
        }
    }
    ```

---

## 13. Testing Extensions

### Unit Testing (InMemory)
Use the **InMemory provider** for fast testing of logic that does not depend on relational features. Create a fresh `DbContextOptions` instance per
test using a unique database name; reusing the same name across tests produces shared, polluted state. The `AppDbContext` in these tests
exposes an options-only constructor and obtains timestamps from the registered `TimestampInterceptor`, not the `TimeProvider`-injecting
constructor shown in Section 1:

```csharp
private static AppDbContext CreateContext()
{
    var options = new DbContextOptionsBuilder<AppDbContext>()
        .UseInMemoryDatabase(Guid.NewGuid().ToString()) // unique name per test
        .AddInterceptors(new TimestampInterceptor(TimeProvider.System))
        .Options;
    return new AppDbContext(options);
}

[Fact]
public async Task TimestampInterceptor_sets_CreatedAt_on_insert()
{
    await using var context = CreateContext();
    var entity = new Order();
    context.Orders.Add(entity);
    await context.SaveChangesAsync();
    Assert.NotEqual(default, entity.CreatedAt);
}
```

### Functional Testing
Run end-to-end tests against real providers (SQLite/SQL Server) to verify generated SQL and relational behavior. SQLite in-memory mode is preferred
for speed and zero infrastructure; switch to SQL Server only for features SQLite does not support (e.g., `ROWVERSION`, temporal tables). Keep the
underlying connection open for the test's lifetime — closing it drops the in-memory database:

```csharp
private static AppDbContext CreateContext()
{
    var connection = new SqliteConnection("Data Source=:memory:");
    connection.Open(); // must stay open; closing drops the in-memory database

    var options = new DbContextOptionsBuilder<AppDbContext>()
        .UseSqlite(connection)
        .AddInterceptors(new TimestampInterceptor(TimeProvider.System))
        .Options;

    var context = new AppDbContext(options);
    context.Database.EnsureCreated();
    return context;
}
```

### Specification Tests (The Gold Standard)
EF Core publishes its own test suites as NuGet packages (e.g., `Microsoft.EntityFrameworkCore.Specification.Tests`). These classes validate
conformance across the full EF Core query and update surface.

*   **Implementation**: Reference the spec-test package, then inherit from provider-specific base classes such as
    `NorthwindQueryRelationalTestBase<TFixture>`. Your fixture configures the database connection; the base class supplies thousands of parameterized
    test cases. Override individual test methods to skip known unsupported behaviors:

    ```csharp
    public class MyProviderQueryTests(MyNorthwindFixture fixture)
        : NorthwindQueryRelationalTestBase<MyNorthwindFixture>(fixture)
    {
        // Indicate a known unsupported behavior without failing the suite.
        public override Task GroupBy_aggregate_in_subquery_doesnt_throw(bool async)
            => Task.CompletedTask;
    }
    ```

*   **Baseline Assertions**: Capture and assert exact SQL using EF's `TestSqlLoggerFactory`, available on the test fixture:

    ```csharp
    var sql = Fixture.TestSqlLoggerFactory.SqlStatements.Last();
    Assert.Equal(
        """
        SELECT "o"."Id", "o"."Total"
        FROM "Orders" AS "o"
        WHERE "o"."Total" > 100.0
        """,
        sql);
    ```

    When a deliberate change alters expected SQL across many tests, set the `EF_TEST_REWRITE_BASELINES=1` environment variable before running to
    bulk-overwrite the `.txt` baseline files, then review and commit the diff:

    ```bash
    $env:EF_TEST_REWRITE_BASELINES = "1"
    dotnet test --filter Category=Specification
    ```

---

## Sources and Further Reading

*   EF Core source: `IDbContextOptionsExtension` contract —
    <https://github.com/dotnet/efcore/blob/main/src/EFCore/Infrastructure/IDbContextOptionsExtension.cs>
*   EF Core source: `DbContextOptionsExtensionInfo` metadata base class —
    <https://github.com/dotnet/efcore/blob/main/src/EFCore/Infrastructure/DbContextOptionsExtensionInfo.cs>
*   EF Core documentation: Interceptors — <https://learn.microsoft.com/ef/core/logging-events-diagnostics/interceptors>
*   EF Core documentation: DbContext configuration and options — <https://learn.microsoft.com/ef/core/dbcontext-configuration/>
*   EF Core documentation: Compiled models — <https://learn.microsoft.com/ef/core/performance/advanced-performance-topics#compiled-models>
*   EF Core documentation: Design-time DbContext creation — <https://learn.microsoft.com/ef/core/cli/dbcontext-creation>
*   EF Core documentation: Design-time services — <https://learn.microsoft.com/ef/core/cli/services>
*   EF Core source: `ModelCacheKeyFactory` (design-time aware cache key) —
    <https://github.com/dotnet/efcore/blob/main/src/EFCore/Infrastructure/ModelCacheKeyFactory.cs>
*   EF Core source: `ModelRuntimeInitializer` (`designTime` model finalization) —
    <https://github.com/dotnet/efcore/blob/main/src/EFCore/Infrastructure/ModelRuntimeInitializer.cs>
