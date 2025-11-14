# Logging Patterns and Conventions

This document describes Verji's patterns and conventions for implementing structured logging across backend microservices using Serilog.

## Overview

All Verji backend services use **Serilog** for structured logging. Structured logging enables:

- **Queryable logs** - Search by specific field values (JobId, TenantId, etc.)
- **Distributed tracing** - Track requests across services via CorrelationId
- **Contextual enrichment** - Automatic inclusion of Environment, Host, Application info
- **Multiple sinks** - Console, file, Seq, OpenTelemetry integration

## Application Startup Configuration

### Serilog Setup in Startup.cs

Both ESB and API hosts follow the same configuration pattern in the `Startup` constructor:

```csharp
using Serilog;
using Serilog.Events;
using System.Net;
using System.Reflection;

public Startup(IWebHostEnvironment env, IConfiguration configuration)
{
    Configuration = configuration;

    var release = Assembly.GetEntryAssembly()!.GetName().Version!.ToString();

    // Set up logging
    var logBuilder = new LoggerConfiguration()
        .Enrich.FromLogContext()
        .Enrich.WithProperty("Environment", env.EnvironmentName)
        .Enrich.WithProperty("Release", release)
        .Enrich.WithProperty("HostId", Dns.GetHostName())
        .Enrich.WithProperty("AppId", env.ApplicationName)
        .WriteTo.Console(
            outputTemplate:
            "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] ({HostId}/{AppId}) {Message:lj} {NewLine}{Exception}")
        .AddOpenTelemetryLogging(env, config)
        .ReadFrom.Configuration(Configuration);

    Log.Logger = logBuilder.CreateLogger();

    // Log startup
    Log.Information("Starting {AppName}...", env.ApplicationName);
}
```

### Configuration Elements

#### Required Enrichers

1. **FromLogContext()** - Enables dynamic property enrichment from log context
2. **Environment** - Development/Staging/Production environment name
3. **Release** - Assembly version number for deployment tracking
4. **HostId** - Machine hostname for distributed system identification
5. **AppId** - Application name (Esb.Host, Api, etc.)

#### Console Sink Template

Standard output template for all services:

```
{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] ({HostId}/{AppId}) {Message:lj} {NewLine}{Exception}
```

**Format breakdown:**
- `Timestamp:yyyy-MM-dd HH:mm:ss.fff` - Timestamp with millisecond precision
- `Level:u3` - Log level as 3-character uppercase (INF, WRN, ERR)
- `({HostId}/{AppId})` - Host and application identification
- `Message:lj` - Message with literal JSON formatting (safe for structured parameters)
- `{NewLine}{Exception}` - Exception details on separate line if present

**Example output:**
```
2025-11-14 12:34:56.789 [INF] (prod-server-01/Verji.ItOps.Api) StartEnsureMetadataCommand/abc-123: Starting...
```

#### Additional Configuration

```csharp
.AddOpenTelemetryLogging(env, config)  // OpenTelemetry integration
.ReadFrom.Configuration(Configuration)  // Additional sinks from appsettings.json
```

The `ReadFrom.Configuration()` allows configuring additional sinks (Seq, file, etc.) and log level overrides in appsettings.json.

## Dependency Injection

### Autofac Registration

Register Serilog with Autofac in `ConfigureContainer`:

```csharp
using Autofac;
using AutofacSerilogIntegration;

public void ConfigureContainer(ContainerBuilder builder)
{
    builder.RegisterLogger();  // Registers ILogger for injection

    // ... other registrations
}
```

This uses the `AutofacSerilogIntegration` package to automatically inject `ILogger` into any class that requests it.

### Constructor Injection

**In Sagas:**

```csharp
using Serilog;

public class YourSaga : Saga<YourSagaData>,
    IAmStartedByMessages<YourCommand>
{
    public ILogger Logger { get; set; }  // Property for logger

    public YourSaga(
        IDocumentStore docStore,
        ILogger logger)  // Inject via constructor
    {
        _docStore = docStore;
        Logger = logger;
    }
}
```

**In Handlers:**

```csharp
using Serilog;

public class YourCommandHandler : IHandleMessages<YourCommand>
{
    private readonly ILogger _logger;  // Private readonly field

    public YourCommandHandler(
        IYourRepository repo,
        ILogger logger)  // Inject via constructor
    {
        _repo = repo;
        _logger = logger;
    }
}
```

**In Connectors/Services:**

```csharp
using Serilog;

public class YourConnector : IYourConnector
{
    private readonly ILogger _logger;  // Private readonly field

    public YourConnector(
        IMapper mapper,
        IDocumentStore docStore,
        ILogger logger)  // Inject via constructor
    {
        _mapper = mapper;
        _docStore = docStore;
        _logger = logger;
    }
}
```

## Usage Patterns

### Entry/Exit Logging (Required Pattern)

**All message handlers and saga handlers MUST log entry with "<prefix>: Starting ..." and exit points with "<prefix>: Done...".**

```csharp
public async Task Handle(YourCommand cmd, IMessageHandlerContext context)
{
    // Extract handler type (clean name without namespace)
    var handlerType = cmd.GetType().Name.Split(".").Last();

    // Log entry with HandlerType and CorrelationId
    Logger.Information("{HandlerType}/{CorrelationId}: Starting...",
        handlerType, cmd.CorrelationId);

    try
    {
        // Handler logic here

        // Log successful exit
        Logger.Information("{HandlerType}/{CorrelationId}: Done...",
            handlerType, cmd.CorrelationId);
    }
    catch (Exception ex)
    {
        Logger.Error(ex, "{HandlerType}/{CorrelationId}: Failed with error",
            handlerType, cmd.CorrelationId);
        throw;
    }
}
```

**For event handlers, strip implementation suffix:**

```csharp
public async Task Handle(IRoomCreated evt, IMessageHandlerContext context)
{
    // Events may have "__impl" suffix from NServiceBus
    var handlerType = evt.GetType().Name.Split(".").Last().Replace("__impl", "");

    Logger.Information("{HandlerType}/{CorrelationId}: Starting...",
        handlerType, evt.RoomId);

    // ... handler logic

    Logger.Information("{HandlerType}/{CorrelationId}: Done...",
        handlerType, evt.RoomId);
}
```

### Structured Logging (Correct Pattern)

**Always use named parameters in curly braces for structured data:**

```csharp
// CORRECT: Named parameters enable structured logging
Logger.Information(
    "{HandlerType}/{CorrelationId}: Processing {TenantCount} tenants",
    handlerType, correlationId, tenantCount);

Logger.Information(
    "Sent command {CommandTypeName} for JobId = {JobId}",
    cmd.GetType().Name, cmd.JobId);

Logger.Warning(
    "{HandlerType}/{CorrelationId}: TenantId missing for room {RoomId}. Skipping...",
    handlerType, correlationId, roomId);
```

This creates structured log entries where each parameter is a separate field that can be queried:
```json
{
  "HandlerType": "ProcessTenantsCommand",
  "CorrelationId": "abc-123",
  "TenantCount": 42,
  "Message": "ProcessTenantsCommand/abc-123: Processing 42 tenants"
}
```

### String Interpolation (Anti-Pattern)

**DO NOT use string interpolation - it prevents structured logging:**

```csharp
// WRONG: String interpolation creates unqueryable logs
_logger.Information($"Processing {tenantCount} tenants");  // ❌

// WRONG: Cannot query by specific TenantCount value
_logger.Information($"Found {deleteCount} rooms that can be deleted.");  // ❌

// CORRECT: Use named parameters
_logger.Information("Processing {TenantCount} tenants", tenantCount);  // ✓
_logger.Information("Found {DeleteCount} rooms that can be deleted", deleteCount);  // ✓
```

## Log Levels

### Information Level

Use for **normal operation flow:**

```csharp
// Starting/completing operations
Logger.Information("{HandlerType}/{CorrelationId}: Starting...", handlerType, correlationId);
Logger.Information("{HandlerType}/{CorrelationId}: Done...", handlerType, correlationId);

// Successful operations
Logger.Information("Sent command {CommandTypeName} for JobId = {JobId}",
    cmdType, jobId);

// Progress updates
Logger.Information(
    "{HandlerType}/{CorrelationId}: Scheduled adding {PersonCount} persons to target",
    handlerType, correlationId, personCount);

// Configuration values
Logger.Information("Allowed CORS origins: {CorsOrigins}", corsOrigins);
```

### Warning Level

Use for **issues that don't stop execution:**

```csharp
// Missing expected data (operation continues)
Logger.Warning(
    "{HandlerType}/{CorrelationId}: TenantSpaceId missing for tenant {TenantId}. Skipping sync...",
    handlerType, correlationId, tenantId);

// Validation issues
Logger.Warning(
    "{HandlerType}/{CorrelationId}: Found no persons in source. Aborting saga...",
    handlerType, correlationId);

// Data quality issues
Logger.Warning(
    "{HandlerType}/{CorrelationId}: cmd.TenantId is '{TenantId}' (unexpected value)",
    handlerType, cmdCorrelationId, tenantId);
```

### Error Level

Use for **failures and exceptions:**

```csharp
// Exceptions
Logger.Error(exc,
    "FAILED sending command {CommandTypeName} for JobId = {JobId}",
    cmdType, jobId);

// Critical validation failures
Logger.Error("FetchSourceInfo(): Data.SourceId is null");

// Operations that cause saga abort
Logger.Error(
    "{HandlerType}/{CorrelationId}: Status indicates failure for space {SpaceId}",
    handlerType, correlationId, spaceId);
Logger.Error(
    "{HandlerType}/{CorrelationId}: Aborting saga...",
    handlerType, correlationId);
```

### Debug Level

Use for **detailed data dumps:**

```csharp
// JSON serialization of complex objects
Logger.Debug(
    "{HandlerType}/{CorrelationId}: PersonsToAdd: {PersonsToAdd}",
    handlerType, correlationId,
    JsonConvert.SerializeObject(persons, Formatting.Indented));

// Detailed state information
Logger.Debug(
    "{HandlerType}/{CorrelationId}: SagaData state: {SagaDataJson}",
    handlerType, correlationId,
    JsonConvert.SerializeObject(Data));
```

## Standard Parameter Names

Use these standardized names for consistency across services:

| Parameter | Description | Example Value |
|-----------|-------------|---------------|
| `{HandlerType}` | Message handler type name | `"StartSyncCommand"` |
| `{CorrelationId}` | Saga/workflow correlation ID | `"abc-123-def"` |
| `{CommandCorrelationId}` | Command-specific correlation | `"cmd-456"` |
| `{SagaCorrelationId}` | Saga-specific correlation | `"saga-789"` |
| `{TenantId}` | Tenant identifier | `"tenant-001"` |
| `{JobId}` | Job identifier | `"job-123"` |
| `{PersonCount}` | Number of persons | `42` |
| `{TenantCount}` | Number of tenants | `10` |
| `{UserCount}` | Number of users | `150` |
| `{RoomId}` | Matrix room identifier | `"!abc:matrix.org"` |
| `{SpaceId}` | Space identifier | `"!space:matrix.org"` |
| `{CommandTypeName}` | Command type name | `"EnsureVsysInRoom"` |
| `{Path}` | File system path | `"/app/config.json"` |
| `{Status}` | Status value | `"Success"` |
| `{ReasonPhrase}` | Error reason/HTTP phrase | `"Not Found"` |

## Special Patterns

### Method Name Prefixes

When logging from utility or connector methods, prefix with the method name:

```csharp
public async Task StartSyncTenantSpacesForTenants()
{
    // Prefix with method name for context
    Log.Information(
        "StartSyncTenantSpacesForTenants(): Will process {TenantCount} tenants",
        tenants.Count);

    foreach (var tenant in tenants)
    {
        if (string.IsNullOrEmpty(tenant.TenantSpaceId))
        {
            Log.Warning(
                "StartSyncTenantSpacesForTenants(): TenantSpaceId missing for tenant {TenantId}. Skipping sync...",
                tenant.Id);
            continue;
        }

        // ... process tenant
    }
}
```

### CorrelationId Propagation

**CorrelationId must be propagated through all log messages** to enable distributed tracing:

```csharp
public async Task Handle(StartWorkflowCommand cmd, IMessageHandlerContext context)
{
    var handlerType = cmd.GetType().Name.Split(".").Last();

    // Log with CorrelationId from incoming message
    Logger.Information("{HandlerType}/{CorrelationId}: Starting...",
        handlerType, cmd.CorrelationId);

    // Call helper method - pass CorrelationId
    await ProcessWorkflowAsync(cmd.CorrelationId, context);

    // Log completion with same CorrelationId
    Logger.Information("{HandlerType}/{CorrelationId}: Done...",
        handlerType, cmd.CorrelationId);
}

private async Task ProcessWorkflowAsync(string correlationId, IMessageHandlerContext context)
{
    // Use CorrelationId in all log messages
    Logger.Information("ProcessWorkflowAsync/{CorrelationId}: Validating...", correlationId);

    // ... processing

    Logger.Information("ProcessWorkflowAsync/{CorrelationId}: Complete", correlationId);
}
```

### Exception Logging with Context

**Always include relevant context when logging exceptions:**

```csharp
try
{
    await _sdk.EnsureUserInRoom(roomId, userId);

    Log.Information(
        "Successfully added user {UserId} to room {RoomId}",
        userId, roomId);
}
catch (Exception exc)
{
    // Include context: what operation failed, what IDs were involved
    Log.Error(exc,
        "FAILED adding user {UserId} to room {RoomId}. JobId = {JobId}",
        userId, roomId, jobId);
    throw;
}
```

### Saga-Specific Multiple CorrelationIds

Some sagas track both command-level and saga-level correlation IDs:

```csharp
public async Task Handle(YourCommand cmd, IMessageHandlerContext context)
{
    var handlerType = cmd.GetType().Name.Split(".").Last();

    // Log both command and saga correlation IDs
    Logger.Information(
        "{HandlerType}/{CommandCorrelationId}/{SagaCorrelationId}: Starting...",
        handlerType, cmd.CorrelationId, Data.CorrelationId);

    // ... handler logic

    Logger.Information(
        "{HandlerType}/{CommandCorrelationId}/{SagaCorrelationId}: Done...",
        handlerType, cmd.CorrelationId, Data.CorrelationId);
}
```

## Complete Examples

### Example 1: Saga Handler with Full Logging

```csharp
public class TenantToTenantSpaceSyncSaga : Saga<TenantToTenantSpaceSyncSagaData>,
    IAmStartedByMessages<StartSyncCommand>,
    IHandleMessages<ContinueSyncCommand>
{
    public ILogger Logger { get; set; }
    private readonly IPersonIdProvider _personIdProvider;

    public TenantToTenantSpaceSyncSaga(
        IPersonIdProvider personIdProvider,
        ILogger logger)
    {
        _personIdProvider = personIdProvider;
        Logger = logger;
    }

    public async Task Handle(StartSyncCommand cmd, IMessageHandlerContext context)
    {
        var handlerType = cmd.GetType().Name.Split(".").Last();

        Logger.Information(
            "{HandlerType}/{CorrelationId}: Starting...",
            handlerType, cmd.CorrelationId);

        if (string.IsNullOrEmpty(cmd.TenantId))
        {
            Logger.Warning(
                "{HandlerType}/{CorrelationId}: cmd.TenantId is null or empty",
                handlerType, cmd.CorrelationId);
            MarkAsComplete();
            return;
        }

        // Initialize saga data
        Data.CorrelationId = cmd.CorrelationId;
        Data.TenantId = cmd.TenantId;

        // Fetch persons from source
        var loadResult = await _personIdProvider.GetPersonIds(cmd.TenantId);

        Logger.Information(
            "{HandlerType}/{CorrelationId}: Got {PersonCount} person-ids from source provider",
            handlerType, cmd.CorrelationId, loadResult.LoadCount);

        if (loadResult.LoadCount == 0)
        {
            Logger.Warning(
                "{HandlerType}/{CorrelationId}: Found no persons in source. Aborting saga...",
                handlerType, cmd.CorrelationId);
            MarkAsComplete();
            return;
        }

        Data.SourcePersonIds = loadResult.PersonIds;

        // Continue processing
        await context.SendLocal(new ContinueSyncCommand(cmd.CorrelationId));

        Logger.Information(
            "{HandlerType}/{CorrelationId}: Done...",
            handlerType, cmd.CorrelationId);
    }
}
```

### Example 2: Command Handler with Exception Handling

```csharp
public class SendNotificationCommandHandler : IHandleMessages<SendNotificationCommand>
{
    private readonly ILogger _logger;
    private readonly INotificationService _notificationService;

    public SendNotificationCommandHandler(
        ILogger logger,
        INotificationService notificationService)
    {
        _logger = logger;
        _notificationService = notificationService;
    }

    public async Task Handle(SendNotificationCommand cmd, IMessageHandlerContext context)
    {
        var handlerType = cmd.GetType().Name.Split(".").Last();

        _logger.Information(
            "{HandlerType}/{CorrelationId}: Starting...",
            handlerType, cmd.CorrelationId);

        try
        {
            await _notificationService.SendAsync(
                cmd.UserId,
                cmd.Message);

            _logger.Information(
                "{HandlerType}/{CorrelationId}: Notification sent to user {UserId}",
                handlerType, cmd.CorrelationId, cmd.UserId);

            // Publish success event
            await context.Publish<INotificationSent>(evt =>
            {
                evt.CorrelationId = cmd.CorrelationId;
                evt.UserId = cmd.UserId;
            });

            _logger.Information(
                "{HandlerType}/{CorrelationId}: Done...",
                handlerType, cmd.CorrelationId);
        }
        catch (Exception exc)
        {
            _logger.Error(exc,
                "{HandlerType}/{CorrelationId}: FAILED sending notification to user {UserId}",
                handlerType, cmd.CorrelationId, cmd.UserId);
            throw;
        }
    }
}
```

### Example 3: Connector with Method-Level Logging

```csharp
public class ItOpsConnector : IItOpsConnector
{
    private readonly ILogger _logger;
    private readonly IMessageSession _msgSession;

    public ItOpsConnector(
        ILogger logger,
        IMessageSession msgSession)
    {
        _logger = logger;
        _msgSession = msgSession;
    }

    public async Task StartSyncTenantSpacesForTenants()
    {
        var tenants = await GetTenantsAsync();

        _logger.Information(
            "StartSyncTenantSpacesForTenants(): Will process {TenantCount} tenants out of total {TenantCountTotal}",
            tenants.Count, tenants.TotalCount);

        foreach (var tenant in tenants)
        {
            if (string.IsNullOrEmpty(tenant.TenantSpaceId))
            {
                _logger.Warning(
                    "StartSyncTenantSpacesForTenants(): TenantSpaceId missing for tenant {TenantId}. Skipping sync...",
                    tenant.Id);
                continue;
            }

            try
            {
                var cmd = new StartTenantSpaceSyncCommand(
                    Guid.NewGuid().ToString(),
                    tenant.Id,
                    tenant.TenantSpaceId);

                await _msgSession.Send(cmd);

                Log.Information(
                    "Sent command {CommandTypeName} for tenant {TenantId}",
                    cmd.GetType().Name, tenant.Id);
            }
            catch (Exception exc)
            {
                Log.Error(exc,
                    "FAILED sending command for tenant {TenantId}",
                    tenant.Id);
                // Don't rethrow - continue processing other tenants
            }
        }

        _logger.Information(
            "StartSyncTenantSpacesForTenants(): Completed processing {TenantCount} tenants",
            tenants.Count);
    }
}
```

## API Request Logging

For API applications, enable Serilog request logging middleware in `Startup.Configure`:

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ... other middleware

    // Logs HTTP requests/responses automatically
    app.UseSerilogRequestLogging();

    // ... routing, endpoints, etc.
}
```

This automatically logs all HTTP requests with:
- Request method and path
- Response status code
- Response time
- Client IP (if configured)

## Configuration Checklist

Use this checklist when implementing logging in a new service:

### Startup Configuration
- [ ] Serilog configured in `Startup` constructor
- [ ] Enrichers added: FromLogContext, Environment, Release, HostId, AppId
- [ ] Console sink configured with standard template
- [ ] OpenTelemetry integration added
- [ ] `ReadFrom.Configuration()` called for additional sinks
- [ ] Static `Log.Logger` set to configured logger
- [ ] Initial startup logged with `Log.Information()`

### Dependency Injection
- [ ] `builder.RegisterLogger()` called in `ConfigureContainer`
- [ ] `ILogger` injected via constructor in all handlers/sagas/services
- [ ] Logger stored in private readonly field (or public property for sagas)

### Handler/Saga Logging
- [ ] Entry logging: `{HandlerType}/{CorrelationId}: Starting...`
- [ ] Exit logging: `{HandlerType}/{CorrelationId}: Done...`
- [ ] HandlerType extracted via `GetType().Name.Split(".").Last()`
- [ ] Event handlers strip `__impl` suffix
- [ ] CorrelationId included in all log messages
- [ ] Exception logging includes context (IDs, operation details)

### Structured Logging
- [ ] All log messages use named parameters (curly braces)
- [ ] No string interpolation used (no `$"..."`)
- [ ] Standard parameter names used consistently
- [ ] Appropriate log levels (Information, Warning, Error, Debug)

### API-Specific
- [ ] `app.UseSerilogRequestLogging()` added to middleware pipeline

## Common Mistakes to Avoid

1. **String interpolation instead of structured parameters:**
   ```csharp
   // WRONG
   _logger.Information($"Processing {count} items");

   // CORRECT
   _logger.Information("Processing {ItemCount} items", count);
   ```

2. **Missing CorrelationId in logs:**
   ```csharp
   // WRONG
   Logger.Information("Processing command");

   // CORRECT
   Logger.Information("{HandlerType}/{CorrelationId}: Processing command",
       handlerType, cmd.CorrelationId);
   ```

3. **Missing entry/exit logging:**
   ```csharp
   // WRONG - No logging
   public async Task Handle(YourCommand cmd, IMessageHandlerContext context)
   {
       // ... processing
   }

   // CORRECT - Entry and exit logged
   public async Task Handle(YourCommand cmd, IMessageHandlerContext context)
   {
       var handlerType = cmd.GetType().Name.Split(".").Last();
       Logger.Information("{HandlerType}/{CorrelationId}: Starting...",
           handlerType, cmd.CorrelationId);

       // ... processing

       Logger.Information("{HandlerType}/{CorrelationId}: Done...",
           handlerType, cmd.CorrelationId);
   }
   ```

4. **Exception without context:**
   ```csharp
   // WRONG
   catch (Exception exc)
   {
       _logger.Error(exc, "Operation failed");
       throw;
   }

   // CORRECT
   catch (Exception exc)
   {
       _logger.Error(exc,
           "{HandlerType}/{CorrelationId}: FAILED processing {TenantId}",
           handlerType, correlationId, tenantId);
       throw;
   }
   ```

5. **Creating session without logging context:**
   ```csharp
   // Not a logging mistake, but common related issue
   // WRONG - Session without context
   await using var session = _docStore.LightweightSession();

   // CORRECT - Session with NServiceBus context (enables OutBox)
   await using var session = _sessionResolver.GetSession(_docStore, context, _dbTenant);
   ```
