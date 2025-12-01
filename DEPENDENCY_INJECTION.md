# Dependency Injection Patterns and Conventions

This document describes Verji's patterns and conventions for dependency injection across backend microservices using Microsoft.Extensions.DependencyInjection and Autofac.

## Overview

Verji backend services use a **dual container approach**:

- **Microsoft.Extensions.DependencyInjection (ServiceCollection)** - For framework/library integrations that provide `AddXxx()` extension methods
- **Autofac** - For all custom/home-grown services and advanced DI patterns

### When to Use Each Container

| Container | Use For | Examples |
|-----------|---------|----------|
| **ServiceCollection** | Framework integrations with existing `AddXxx()` methods | `AddDbContext()`, `AddAuthentication()`, `AddCors()`, `AddSwaggerGen()`, `AddVerjiHealthChecks()` |
| **Autofac** | Custom services, repositories, handlers, SDK clients | Repositories, connectors, domain services, API clients, sagas |

**Key Principle**: If a library provides an `AddXxx()` extension method for `IServiceCollection`, use it in `ConfigureServices`. All other registrations (your own code) go in `ConfigureContainer` with Autofac.

### Why Autofac for Custom Services?

Autofac provides powerful features for organizing and managing custom service registrations:
- **Keyed/Named Services** - Register multiple implementations of the same interface, resolved by key
- **IIndex<K,V>** - Runtime resolution of services by key without service locator pattern
- **Modules** - Organize related registrations into reusable units
- **Configuration-driven registration** - Build registrations from config files
- **Attribute filtering** - Inject specific implementations using attributes
- **Better control** - Fine-grained lifetime management and registration patterns

## Startup Class Structure

All Verji services follow this standard Startup pattern with two configuration methods:

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }

    public Startup(IHostEnvironment env, IConfiguration config)
    {
        Configuration = config;
        // ... logging setup
    }

    // Called first: Framework/library integrations with existing AddXxx() methods
    public void ConfigureServices(IServiceCollection services)
    {
        // Use ServiceCollection for frameworks that provide AddXxx() extensions
        services.AddDbContext<IdentityDbContext>(options => ...);  // EF Core
        services.AddAuthentication(...);                            // ASP.NET Auth
        services.AddCors(options => ...);                           // CORS middleware
        services.AddSwaggerGen(...);                                // Swagger/OpenAPI
        services.AddVerjiHealthChecks(Configuration);               // Health checks
        services.AddVerjiCaching(Configuration);                    // Caching
    }

    // Called second: All custom/home-grown services via Autofac
    public void ConfigureContainer(ContainerBuilder builder)
    {
        // Serilog integration
        builder.RegisterLogger();

        // Configuration as injectable interface
        builder.Register(_ => Configuration).AsImplementedInterfaces();

        // Autofac modules for organized registrations
        builder.RegisterModule(new MartenIoCModule(Configuration));

        // Custom services via extension methods
        builder.AddMappers(Configuration);           // AutoMapper profiles
        builder.AddAccountServices(Configuration);   // Our repositories, connectors
        builder.AddVerjiMatrixSdk(Configuration, "VerjiSdk_Internal");  // Our SDK clients

        // Polly policies (instance registration)
        var pollyReg = new PolicyRegistry();
        pollyReg.RegisterVerjiMatrixSdkPolicies();
        builder.RegisterInstance(pollyReg)
            .AsSelf()
            .AsImplementedInterfaces()
            .SingleInstance();
    }
}
```

### Execution Order

1. `ConfigureServices` runs first - framework integrations via `IServiceCollection`
2. `ConfigureContainer` runs second - custom services via Autofac
3. Autofac builds the final container incorporating both

**Key Points**:
- Services registered in `ConfigureServices` are available in Autofac (they're merged)
- Autofac-specific features (keyed services, IIndex, modules) require `ConfigureContainer`
- When in doubt: framework = `ConfigureServices`, custom code = `ConfigureContainer`

## Basic Registration Patterns

### Type Registration

```csharp
// Register type with all implemented interfaces
builder.RegisterType<MartenVmxUserRepository>()
    .AsImplementedInterfaces()
    .InstancePerLifetimeScope();

// Register as self (concrete type)
builder.RegisterType<GbFeatureManager>()
    .AsSelf()
    .AsImplementedInterfaces();

// Register as specific interface
builder.RegisterType<DefaultSessionResolver>()
    .As<IResolveSessions>()
    .InstancePerLifetimeScope();
```

### Factory Registration

```csharp
// Lambda factory for complex construction
builder.Register(ctx =>
{
    var configSection = config.GetSection("GrowthBookContext");
    var jToken = ConfigUtils.BuildJson(configSection);
    var jsonString = JsonConvert.SerializeObject(jToken);
    return JsonConvert.DeserializeObject<GbContext>(jsonString);
});

// Factory using resolved dependencies
builder.Register(c => c.Resolve<MapperConfiguration>().CreateMapper(c.Resolve))
    .As<IMapper>()
    .SingleInstance();
```

### Instance Registration

```csharp
// Register existing instance
builder.RegisterInstance(new LiteralTenancyResolver(tenantId))
    .AsImplementedInterfaces()
    .SingleInstance();

// Register configuration object
builder.RegisterInstance(pollyReg)
    .AsSelf()
    .AsImplementedInterfaces()
    .SingleInstance();
```

### Lifetime Scopes

| Scope | Description | Use When |
|-------|-------------|----------|
| `SingleInstance()` | One instance for application lifetime | Stateless services, configuration, caches |
| `InstancePerLifetimeScope()` | One instance per scope (e.g., HTTP request) | Repositories, DbContext, scoped services |
| `InstancePerDependency()` | New instance every resolution (default) | Lightweight, stateless helpers |

```csharp
// Singleton
builder.RegisterType<PolicyRegistry>().SingleInstance();

// Scoped (per request in web apps)
builder.RegisterType<MartenUserRepository>().InstancePerLifetimeScope();

// Transient (new each time)
builder.RegisterType<RequestValidator>().InstancePerDependency();
```

## Autofac Modules Pattern

Modules encapsulate related registrations, making them reusable and testable.

### Creating a Module

```csharp
using Autofac;
using Module = Autofac.Module;

public class CsApiClientModule : Module
{
    private readonly string _baseUrl;
    private readonly HttpMessageHandler _handler;
    private readonly string _apiVariant;
    private readonly string _serializationPreset;

    public CsApiClientModule(IConfigurationSection configSection, HttpMessageHandler? handler = null)
    {
        if (!configSection.Exists())
            throw new Exception("Provided config section does not exist");

        _baseUrl = configSection["BaseUrl"]
            ?? throw new Exception("Unable to find setting 'BaseUrl' in config section");
        _apiVariant = configSection["Variant"]
            ?? throw new Exception("Unable to find setting 'Variant' in config section");
        _serializationPreset = configSection["SerializationPreset"]
            ?? SerializationPreset.UnionConverter;

        _handler = handler ?? new HttpClientHandler { AllowAutoRedirect = false };

        Log.Information(
            "CsApiClientModule(): Preparing registration with Variant='{ApiVariant}', BaseUrl='{BaseUrl}'",
            _apiVariant, _baseUrl);
    }

    protected override void Load(ContainerBuilder builder)
    {
        var client = new HttpClient(_handler) { BaseAddress = new Uri(_baseUrl) };
        var settings = new RefitSettings
        {
            UrlParameterFormatter = new VerjiSkipNullQueryValuesFormatter(),
            ContentSerializer = new SystemTextJsonContentSerializer(
                JsonSerializerOptionsPresets.GetSerializationPreset(_serializationPreset))
        };

        // Register raw API client with keying
        builder.Register(_ => RestService.For<ICsApiRaw>(client, settings))
            .As<ICsApiRaw>()
            .Keyed<ICsApiRaw>(_apiVariant)
            .InstancePerLifetimeScope();

        // Register wrapper client with variant parameter
        builder.RegisterType<CsApiClient>()
            .As<ICsApiClient>()
            .WithParameter(new NamedParameter("apiVariant", _apiVariant))
            .WithParameter(new NamedParameter("serializationPreset", _serializationPreset))
            .Keyed<ICsApiClient>(_apiVariant)
            .InstancePerLifetimeScope();
    }
}
```

### Using Modules

```csharp
public void ConfigureContainer(ContainerBuilder builder)
{
    // Register module with configuration
    builder.RegisterModule(new MartenIoCModule(Configuration));
    builder.RegisterModule(new CasbinIoCModule(Configuration, false, false));

    // Modules can also be registered from config sections
    var csApiConfig = Configuration.GetSection("VerjiSdk_Internal:CsApi");
    builder.RegisterModule(new CsApiClientModule(csApiConfig));
}
```

## Extension Method Pattern

Group related registrations into fluent extension methods for better organization.

### Creating Extension Methods

```csharp
public static class ConfigExtensions
{
    public static ContainerBuilder AddPersonIdProviders(this ContainerBuilder builder, IConfiguration config)
    {
        builder.RegisterType<OrganizationPersonIdProvider>()
            .WithParameter("orgAssociation", OrganizationAssociation.Client)
            .Keyed<BasePersonIdProvider>("AllOrganizationPersons")
            .AsSelf()
            .AsImplementedInterfaces();

        builder.RegisterType<RoomMembersPersonIdProvider>()
            .Keyed<BasePersonIdProvider>("RoomMembers")
            .AsSelf()
            .AsImplementedInterfaces();

        builder.RegisterType<SystemManagedSpaceMembersPersonIdProvider>()
            .Keyed<BasePersonIdProvider>("SystemManagedSpaceMembers")
            .AsSelf()
            .AsImplementedInterfaces();

        return builder;  // Enable fluent chaining
    }

    public static ContainerBuilder AddLicensingServices(this ContainerBuilder builder, IConfiguration config)
    {
        builder.RegisterType<SigningLicenseRepository>()
            .AsSelf()
            .AsImplementedInterfaces()
            .InstancePerLifetimeScope()
            .Keyed<BaseLicenseRepository>(Mn.Signing.ToLowerInvariant());

        builder.RegisterType<OnboardingLicenseRepository>()
            .AsSelf()
            .AsImplementedInterfaces()
            .InstancePerLifetimeScope()
            .Keyed<BaseLicenseRepository>(Mn.Onboarding.ToLowerInvariant());

        return builder;
    }
}
```

### Using Extension Methods

```csharp
public void ConfigureContainer(ContainerBuilder builder)
{
    // Fluent registration chain
    builder
        .AddMappers(Configuration)
        .AddSessionResolver(Configuration)
        .AddPersonIdProviders(Configuration)
        .AddLicensingServices(Configuration)
        .AddVerjiMatrixSdk(Configuration, "VerjiSdk_Internal");
}
```

## Keyed/Named Dependencies (Advanced)

Keyed services allow multiple implementations of the same interface, distinguished by a key.

### Registration with Keys

```csharp
// Register multiple PersonIdProviders with different keys
builder.RegisterType<OrganizationPersonIdProvider>()
    .WithParameter("orgAssociation", OrganizationAssociation.Client)
    .Keyed<BasePersonIdProvider>("AllOrganizationPersons")
    .AsSelf()
    .AsImplementedInterfaces();

builder.RegisterType<RoomMembersPersonIdProvider>()
    .Keyed<BasePersonIdProvider>("RoomMembers")
    .AsSelf()
    .AsImplementedInterfaces();

// Register API clients by variant
builder.RegisterType<CsApiClient>()
    .As<ICsApiClient>()
    .Keyed<ICsApiClient>("Internal")
    .InstancePerLifetimeScope();

// Register with module name as key
builder.RegisterType<SigningLicenseRepository>()
    .Keyed<BaseLicenseRepository>("signing")
    .InstancePerLifetimeScope();
```

### Resolution with IIndex<TKey, TValue>

`IIndex<TKey, TValue>` provides dictionary-like access to keyed services without service locator anti-pattern:

```csharp
using Autofac.Features.Indexed;

public class CsApiClient : ICsApiClient
{
    private readonly ICsApiRaw _api;
    private readonly JsonSerializerOptions _jsonSerializerOptions;

    public CsApiClient(
        string apiVariant,
        IIndex<string, ICsApiRaw> apiRawIndex,
        string serializationPreset,
        IIndex<string, JsonSerializerOptions> jsonSerializerOptionsIndex,
        IReadOnlyPolicyRegistry<string> pollyReg)
    {
        // Resolve specific implementation by key
        _api = apiRawIndex[apiVariant];
        _jsonSerializerOptions = jsonSerializerOptionsIndex[serializationPreset];
    }

    public async Task<RestResultWrapper<CreateRoomResponse>> CreateRoom(
        CreateRoomRequest request,
        string accessToken,
        string policyName = "CsApiPolicy",
        CancellationToken? ct = null)
    {
        var policy = pollyReg.Get<AsyncPolicy<HttpResponseMessage>>(policyName);
        // Use _api which was resolved by variant key
        var reqResult = await policy.ExecuteAndCaptureAsync(
            (_, cToken) => _api.CreateRoom(request, $"Bearer {accessToken}", cToken),
            new Context("CsApiClient.CreateRoom()"),
            ct ?? CancellationToken.None);
        return await reqResult.ExtractAsync<CreateRoomResponse>(Log.Logger, _jsonSerializerOptions);
    }
}
```

### Runtime Resolution Pattern

```csharp
public class SyncLicenseWithPersonIdSources<TLicense> : ISyncLicenseWithTenantMarker
{
    private readonly IIndex<string, BaseLicenseRepository> _licenseRepoIndex;
    private readonly IIndex<string, BasePersonIdSource> _personIdSourceIndex;

    public SyncLicenseWithPersonIdSources(
        ILogger logger,
        IIndex<string, BaseLicenseRepository> licenseRepoIndex,
        IIndex<string, BasePersonIdSource> personIdSourceIndex)
    {
        _licenseRepoIndex = licenseRepoIndex;
        _personIdSourceIndex = personIdSourceIndex;
    }

    public async Task<(int, int, int, string)> StartSyncAsync(
        IDocumentSession session,
        string moduleName,
        string entitlementGroup)
    {
        // Resolve repositories dynamically by module name
        var licenseRepo = _licenseRepoIndex[moduleName.ToLowerInvariant()];
        var personIdSource = _personIdSourceIndex[entitlementGroup];

        var targetMemberIds = await personIdSource.GetPersonIds(session, ...);
        // ... sync logic
    }
}
```

## Attribute-Based Filtering

Use `[KeyFilter]` attribute for constructor parameter injection of keyed services.

### Registration Setup

Controllers using `[KeyFilter]` must be registered with `.WithAttributeFiltering()`:

```csharp
public void ConfigureContainer(ContainerBuilder builder)
{
    // Register controllers with attribute filtering enabled
    var controllers = typeof(Startup).Assembly
        .GetTypes()
        .Where(t => t.BaseType == typeof(ControllerBase))
        .ToArray();

    builder.RegisterTypes(controllers).WithAttributeFiltering();

    // Register keyed configuration
    builder.Register(_ =>
    {
        var seeds = Configuration.GetSection("SuperuserConfig").Get<AclGroupSeed[]>();
        return seeds ?? Array.Empty<AclGroupSeed>();
    })
    .Keyed<AclGroupSeed[]>("SuperuserConfig")
    .SingleInstance();
}
```

### Using KeyFilter in Controllers

```csharp
using Autofac.Features.AttributeFilters;

[ApiVersion("1.1")]
[Route("api/v{version:apiVersion}/acl")]
[ApiController]
public class AccessControlController : ControllerBase
{
    private readonly IMapper _mapper;
    private readonly IAccessControlConnector _aclConnector;
    private readonly AclGroupSeed[] _superuserConfig;

    public AccessControlController(
        IMapper mapper,
        IAccessControlConnector aclConnector,
        [KeyFilter("SuperuserConfig")] AclGroupSeed[] superuserConfig,
        ILogger logger)
    {
        _mapper = mapper;
        _aclConnector = aclConnector;
        _superuserConfig = superuserConfig;
    }

    [HttpGet("tenant/{tenantId}/permissions/{moduleName}")]
    public async Task<IActionResult> GetModulePermissions(string tenantId, string moduleName)
    {
        var userId = User.Identity.GetSubjectId();
        var aclInfo = await _aclConnector.GetModulePermissions(
            userId, tenantId, moduleName, _superuserConfig);
        return Ok(aclInfo);
    }
}
```

### When to Use KeyFilter vs IIndex

| Use Case | Approach |
|----------|----------|
| Known key at compile time | `[KeyFilter("key")]` |
| Key determined at runtime | `IIndex<string, T>` |
| Single specific implementation needed | `[KeyFilter("key")]` |
| Need to iterate or select from multiple | `IIndex<string, T>` |
| Controller constructor injection | `[KeyFilter("key")]` |
| Service with dynamic resolution | `IIndex<string, T>` |

## Configuration-Driven Registrations

### SDK Variant Pattern

Register SDK clients based on configuration sections:

```csharp
public static ContainerBuilder AddVerjiMatrixSdk(
    this ContainerBuilder builder,
    IConfiguration config,
    string configSectionName)
{
    Log.Information("Adding VerjiMatrixSdk services using config section '{ConfigSectionName}'",
        configSectionName);

    var sdkConfigSection = config.GetSection(configSectionName);

    // Register serialization presets (keyed by preset name)
    foreach (var preset in SerializationPreset.All)
    {
        builder.Register(_ => JsonSerializerOptionsPresets.GetSerializationPreset(preset))
            .Keyed<JsonSerializerOptions>(preset)
            .SingleInstance();
    }

    // Register API clients from config subsections
    var csApiConfig = sdkConfigSection.GetSection("CsApi")
        ?? throw new Exception($"No CsApi section in {configSectionName}");
    builder.AddCsApiClient(csApiConfig);

    var adminApiConfig = sdkConfigSection.GetSection("AdminApi")
        ?? throw new Exception($"No AdminApi section in {configSectionName}");
    builder.AddAdminApiClient(adminApiConfig);

    var vmxServicesApiConfig = sdkConfigSection.GetSection("VmxServicesApi")
        ?? throw new Exception($"No VmxServicesApi section in {configSectionName}");
    builder.AddVmxServicesApiClient(vmxServicesApiConfig);

    // Register SDK implementations
    builder.RegisterType<VerjiSdk>()
        .AsImplementedInterfaces()
        .InstancePerLifetimeScope();

    builder.RegisterType<VerjiSdkV2>()
        .WithParameter("apiVariant", ApiVariant.Internal)
        .AsImplementedInterfaces()
        .InstancePerLifetimeScope();

    return builder;
}
```

### Configuration File Structure

```json
{
  "VerjiSdk_Internal": {
    "CsApi": {
      "Variant": "Internal",
      "LogRequests": false,
      "BaseUrl": "http://synapse-svc",
      "BypassCertificateValidationChainCheck": true,
      "SerializationPreset": "UnionConverter"
    },
    "AdminApi": {
      "Variant": "Internal",
      "LogRequests": false,
      "BaseUrl": "http://synapse-svc",
      "BypassCertificateValidationChainCheck": true
    },
    "VmxServicesApi": {
      "Variant": "Internal",
      "LogRequests": false,
      "BaseUrl": "http://vmx-services-svc",
      "BypassCertificateValidationChainCheck": true
    }
  }
}
```

### ApiVariant Constants

```csharp
public static class ApiVariant
{
    public const string Internal = nameof(Internal);  // Cluster-internal services
    public const string External = nameof(External);  // Externally exposed services
}
```

## Common Patterns

### Serilog Logger Registration

```csharp
using AutofacSerilogIntegration;

public void ConfigureContainer(ContainerBuilder builder)
{
    // Automatically injects ILogger into constructors
    builder.RegisterLogger();
}
```

### AutoMapper Registration

```csharp
public static ContainerBuilder AddMappers(this ContainerBuilder builder, IConfiguration config)
{
    builder.Register(_ => new MapperConfiguration(cfg =>
    {
        cfg.AddProfile(new VerjiSdkMappings());
        // Add more profiles as needed
    })).AsSelf().SingleInstance();

    builder.Register(c => c.Resolve<MapperConfiguration>().CreateMapper(c.Resolve))
        .As<IMapper>()
        .SingleInstance();

    return builder;
}
```

### Polly Policy Registry

```csharp
public void ConfigureContainer(ContainerBuilder builder)
{
    var pollyReg = new PolicyRegistry();

    // Register retry policies
    pollyReg.RegisterVerjiMatrixSdkPolicies();
    pollyReg.RegisterMisApiPolicies();
    pollyReg.RegisterRetryOnRateLimitPolicy();

    // NoOp policy for testing
    pollyReg.Add("HttpNoOpPolicy", Policy.NoOpAsync<HttpResponseMessage>());

    builder.RegisterInstance(pollyReg)
        .AsSelf()
        .AsImplementedInterfaces()
        .SingleInstance();
}
```

### Configuration as Injectable Interface

```csharp
// Makes IConfiguration available for injection
builder.Register(_ => Configuration).AsImplementedInterfaces();
```

## Anti-Patterns

### DO NOT: Register Custom Services in ServiceCollection

```csharp
// WRONG: Custom services should not go in ConfigureServices
public void ConfigureServices(IServiceCollection services)
{
    services.AddScoped<IUserRepository, MartenUserRepository>();  // Wrong place!
    services.AddScoped<IAccountConnector, AccountConnector>();    // Wrong place!
}

// CORRECT: Custom services go in ConfigureContainer with Autofac
public void ConfigureContainer(ContainerBuilder builder)
{
    builder.RegisterType<MartenUserRepository>()
        .AsImplementedInterfaces()
        .InstancePerLifetimeScope();

    builder.RegisterType<AccountConnector>()
        .AsImplementedInterfaces()
        .InstancePerLifetimeScope();
}
```

### DO NOT: Use ServiceCollection for Keyed Services

```csharp
// WRONG: ServiceCollection doesn't support keyed services well
services.AddScoped<IPersonIdProvider, RoomMembersPersonIdProvider>();
services.AddScoped<IPersonIdProvider, OrganizationPersonIdProvider>();
// Which one gets resolved? Undefined behavior!

// CORRECT: Use Autofac with keys
builder.RegisterType<RoomMembersPersonIdProvider>()
    .Keyed<IPersonIdProvider>("RoomMembers");
builder.RegisterType<OrganizationPersonIdProvider>()
    .Keyed<IPersonIdProvider>("Organization");
```

### DO NOT: Forget WithAttributeFiltering

```csharp
// WRONG: KeyFilter won't work without WithAttributeFiltering
builder.RegisterType<MyController>();

public class MyController
{
    public MyController([KeyFilter("key")] IService service) { }  // Won't resolve!
}

// CORRECT: Enable attribute filtering
builder.RegisterTypes(controllers).WithAttributeFiltering();
```

### DO NOT: Mix Lifetime Scopes Incorrectly

```csharp
// WRONG: Singleton depending on scoped service
builder.RegisterType<MySingleton>()
    .SingleInstance();  // Lives forever

public class MySingleton
{
    // IDocumentSession is scoped - will be disposed while MySingleton still references it!
    public MySingleton(IDocumentSession session) { }
}

// CORRECT: Use factory pattern or match lifetimes
builder.RegisterType<MySingleton>()
    .SingleInstance();

public class MySingleton
{
    private readonly IDocumentStore _store;
    public MySingleton(IDocumentStore store)  // Store is singleton
    {
        _store = store;
    }

    public void DoWork()
    {
        using var session = _store.LightweightSession();  // Create scoped session when needed
    }
}
```

### DO NOT: Service Locator Pattern

```csharp
// WRONG: Resolving from container directly
public class MyService
{
    private readonly ILifetimeScope _scope;
    public MyService(ILifetimeScope scope)
    {
        _scope = scope;
    }

    public void DoWork(string key)
    {
        var service = _scope.ResolveKeyed<IService>(key);  // Service locator anti-pattern
    }
}

// CORRECT: Use IIndex for keyed resolution
public class MyService
{
    private readonly IIndex<string, IService> _services;
    public MyService(IIndex<string, IService> services)
    {
        _services = services;
    }

    public void DoWork(string key)
    {
        var service = _services[key];  // Clean, testable
    }
}
```

## Checklist

### Basic Setup
- [ ] Startup has both `ConfigureServices` and `ConfigureContainer` methods
- [ ] Framework integrations (AddDbContext, AddAuthentication, etc.) in `ConfigureServices`
- [ ] Custom/home-grown services registered in `ConfigureContainer` with Autofac
- [ ] `builder.RegisterLogger()` called for Serilog integration
- [ ] Configuration registered as interface: `builder.Register(_ => Configuration).AsImplementedInterfaces()`

### Registration Best Practices
- [ ] Services with DB access use `InstancePerLifetimeScope()`
- [ ] Stateless configuration/caches use `SingleInstance()`
- [ ] Related registrations grouped into extension methods
- [ ] Complex registrations encapsulated in Autofac Modules

### Keyed Services
- [ ] Multiple implementations of same interface use `.Keyed<T>(key)`
- [ ] Runtime resolution uses `IIndex<TKey, TValue>` injection
- [ ] Controllers with `[KeyFilter]` registered with `.WithAttributeFiltering()`

### Configuration-Driven
- [ ] SDK variants read from config sections
- [ ] Config validation throws on missing required settings
- [ ] Startup logs which config sections are being used

### Testing Considerations
- [ ] `IIndex<K,V>` can be mocked using `Mock<IIndex<string, T>>().Setup(x => x[key]).Returns(mockService)`
- [ ] Modules can be tested independently
- [ ] Configuration-driven registrations work with test configuration

## Reference Implementations

Study these files for complete examples:

- **ESB Host DI Setup**: `Verji.ItOps.Matrix.Esb.Host/Startup.cs`
- **API Host DI Setup**: `Verji.Account.Api/Startup.cs`
- **SDK Registration Patterns**: `Verji.ItOps.Matrix.Sdk/VerjiSdk/Config/Extensions.cs`
- **Autofac Module Example**: `Verji.ItOps.Matrix.Sdk/Api/Config/CsApiClientModule.cs`
- **Keyed Registrations**: `Verji.ItOps.Esb.Host/Config/ConfigExtensions.cs`
- **IIndex Usage**: `Verji.ItOps.Matrix.Sdk/Api/ClientServer/CsApiClient.cs`
