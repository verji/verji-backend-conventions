# NServiceBus Patterns and Conventions

This document describes Verji's patterns and conventions for implementing NServiceBus message handlers and sagas across backend microservices.

## Overview

NServiceBus is the backbone of Verji's event-driven architecture, enabling reliable message-based communication between microservices. Our implementation emphasizes:

- **CQRS Pattern** - APIs handle queries, ESB handlers process commands
- **Event-Driven Architecture** - Services communicate via commands and events
- **Saga Orchestration** - Long-running workflows coordinate multiple services
- **Transactional Consistency** - OutBox pattern ensures message delivery with database changes

## Message Types

### Commands (Imperative)
Commands represent actions to be performed. They have exactly one handler.

**Public Commands** (in `*.Esb.Abstractions` assemblies):
- Shared across services
- Part of the public contract
- Used for inter-service communication
- Examples: `EnsureVsysIsInRoomCommand`, `AddTenantInfoToRoomCommand`, `ValidateTenantCommand`

**Internal Commands** (defined within handler/saga files):
- Private to a single saga or handler
- Used for saga flow control
- Not exposed in Abstractions assemblies
- Examples: `StartEnsureMetadataForVerjiRoomSagaCommand`, `ContinueEnsureMetadataForVerjiRoomSagaCommand`, `ValidateRoomAndFindParentSpaceCommand`

### Events (Past Tense)
Events represent things that have happened. They can have multiple subscribers.

**Public Events** (in `*.Esb.Abstractions` assemblies):
- Published for other services to react to
- Interface-based (prefix with `I`)
- Examples: `IRoomCreated`, `IVsysUserEnsuredToBeInRoom`, `ITenantInfoAddedToRoom`

## Message Handler Patterns

### Basic Handler Structure

All message handlers follow this standard pattern:

```csharp
public class YourCommandHandler : IHandleMessages<YourCommand>
{
    private readonly ILogger _logger;
    private readonly IYourRepository _repo;

    public YourCommandHandler(ILogger logger, IYourRepository repo)
    {
        _logger = logger;
        _repo = repo;
    }

    public async Task Handle(YourCommand message, IMessageHandlerContext context)
    {
        var handlerType = message.GetType().Name.Split(".").Last();
        _logger.Information("{HandlerType}/{CorrelationId}: Starting ...",
            handlerType, message.CorrelationId);

        // Handler logic here

        _logger.Information("{HandlerType}/{CorrelationId}: Done ...",
            handlerType, message.CorrelationId);
    }
}
```

### Required Patterns

1. **Entry/Exit Logging** - Always log at start and end with `{HandlerType}/{CorrelationId}` format
2. **Extract HandlerType** - Use `GetType().Name.Split(".").Last()` to get clean type name
3. **CorrelationId Propagation** - Extract from incoming message, create if missing, add to outgoing messages
4. **Structured Logging** - Use Serilog with structured parameters

## Saga Orchestration Patterns

Sagas coordinate long-running business processes that span multiple message handlers and services.

### Core Principle: Sagas Orchestrate, Handlers Execute

**Sagas should:**
- Control workflow and decision logic
- Dispatch commands to specialized handlers
- Track orchestration state
- Handle timeouts and retries
- Maintain business-meaningful state in domain objects

**Sagas should NOT:**
- Perform business operations directly
- Make direct API/SDK calls for business logic (delegate to handlers instead)
- Contain complex business rules (move to domain services)

### Dual State Management Pattern

This is a **critical pattern** in Verji sagas. Sagas maintain two distinct types of state:

#### 1. SagaData - Private Orchestration State

SagaData is private to the saga and used for flow control:

```csharp
public class EnsureMetadataForVerjiRoomSagaData : ContainSagaData
{
    public string CorrelationId { get; set; } = null!;
    public string RoomId { get; set; } = null!;

    // Orchestration flags - track workflow progress
    public bool SagaHasBeenTriggered { get; set; }
    public bool SagaHasBeenPaused { get; set; }
    public bool HaveCheckedRoomIsNotSpace { get; set; }
    public bool HaveCheckedRoomIsNotDmRoom { get; set; }
    public bool HaveCheckedIfVsysIsInRoom { get; set; }
    public bool EnsureVsysIsInRoomCommandSent { get; set; }

    // Resolved values from multiple sources
    public bool? RoomHasTypeSpace { get; set; }
    public bool? RoomHasTypeSpaceFromRoomCreateEvent { get; set; }
    public bool? RoomHasTypeSpaceFromSdkLookupInTimeout { get; set; }

    // Temporary state for current execution
    public int CurrentRawRoomIndex { get; set; }
    public int TotalRawRoomsToAnalyze { get; set; }
}
```

**Characteristics:**
- Automatically persisted by NServiceBus
- Not queryable by other services
- Contains workflow control flags
- Tracks what steps have been completed
- Stores temporary execution state
- May aggregate values from multiple sources

#### 2. Domain Objects - Shared Business State

Domain objects persist business-meaningful data queryable by other services:

```csharp
public async Task Handle(AnalyzeRawRoomCommand cmd, IMessageHandlerContext context)
{
    var handlerType = cmd.GetType().Name.Split(".").Last();
    _logger.Information("{HandlerType}/{CorrelationId}: Starting ...", handlerType, cmd.CorrelationId);

    // Get Marten session with OutBox participation
    await using var session = _sessionResolver.GetSession(_docStore, context, _dbTenant);

    // Load domain object
    var hierarchyAnalysis = await _repo.GetById(session, Data.HierarchyAnalysisId);

    // Perform business logic
    var roomAnalysis = new HierarchyAnalysisAnalyzedRoom
    {
        RoomId = roomId,
        RoomName = roomName,
        IsDirectMessage = isDirect,
        RoomMembers = roomMembers.ToArray(),
        DomainMemberMap = domainMemberMap
    };

    // Update domain object using Marten patch operations
    session.Patch<HierarchyAnalysis>(Data.HierarchyAnalysisId)
        .InsertIfNotExists(x => x.AnalyzedRooms, roomAnalysis)
        .Append(h => h.Log, "Analyzed room " + roomId);

    // Save changes - participates in OutBox transaction
    await session.SaveChangesAsync(context.CancellationToken);

    // Update saga orchestration state
    Data.CurrentRawRoomIndex++;

    _logger.Information("{HandlerType}/{CorrelationId}: Done ...", handlerType, cmd.CorrelationId);
}
```

**Characteristics:**
- Persisted via Marten repositories
- Queryable by other services via APIs
- Contains business-meaningful data
- Updated using Marten patch operations
- Participates in OutBox transaction

**Decision Guide: SagaData vs Domain Object**

| Data Type | Store In | Example |
|-----------|----------|---------|
| "Have I checked X?" flags | SagaData | `HaveCheckedRoomIsNotSpace` |
| "Which step am I on?" | SagaData | `CurrentRawRoomIndex` |
| "Did I already send this command?" | SagaData | `EnsureVsysIsInRoomCommandSent` |
| Business results | Domain Object | `AnalyzedRooms`, `PersonCacheList` |
| Audit logs | Domain Object | `Log` entries |
| Queryable metrics | Domain Object | `DmRoomCount`, `NormalRoomCount` |

### NServiceBus OutBox with Marten Sessions

**CRITICAL PATTERN**: Always obtain Marten sessions via `IResolveSessions` and pass the NServiceBus `context`.

#### Why This Matters

The OutBox pattern ensures transactional consistency between:
1. Database changes (domain objects)
2. Saga state updates (SagaData)
3. Outgoing message dispatch (commands/events)

All three happen atomically or not at all.

#### Correct Pattern

```csharp
public class YourSaga : Saga<YourSagaData>
{
    private readonly IDocumentStore _docStore;
    private readonly IResolveSessions _sessionResolver;
    private readonly string _dbTenant;

    public YourSaga(
        IDocumentStore docStore,
        IResolveSessions sessionResolver,
        IResolveTenants tenantResolver)
    {
        _docStore = docStore;
        _sessionResolver = sessionResolver;
        _dbTenant = tenantResolver.TenantId;
    }

    public async Task Handle(YourCommand cmd, IMessageHandlerContext context)
    {
        var handlerType = cmd.GetType().Name.Split(".").Last();
        _logger.Information("{HandlerType}/{CorrelationId}: Starting ...", handlerType, cmd.CorrelationId);

        // CORRECT: Get session with OutBox participation
        await using var session = _sessionResolver.GetSession(_docStore, context, _dbTenant);

        // Perform database operations
        var entity = await _repo.GetById(session, Data.EntityId);
        _repo.Upsert(session, entity);
        await session.SaveChangesAsync(context.CancellationToken);

        // Send messages - will be dispatched atomically with DB changes
        await context.Send(new NextCommand(Data.CorrelationId));

        _logger.Information("{HandlerType}/{CorrelationId}: Done ...", handlerType, cmd.CorrelationId);
    }
}
```

#### Incorrect Pattern (DO NOT USE)

```csharp
// WRONG: Creating session without context - NO OutBox participation!
await using var session = _docStore.LightweightSession(_dbTenant);

// This is dangerous because:
// 1. DB changes might succeed but message dispatch fails
// 2. Messages might be sent but DB changes roll back
// 3. Saga state might update but domain object changes fail
```

#### Key Points

1. **Always pass `context`** - This enrolls the session in the OutBox transaction
2. **Use `IResolveSessions`** - Don't create sessions directly
3. **Multi-tenancy support** - Pass `_dbTenant` from `IResolveTenants`
4. **Session lifecycle** - Use `await using` for proper disposal

### Internal vs Public Commands

#### Internal Commands (SendLocal)

Internal commands control saga flow and are not exposed to other services:

```csharp
// Internal command - defined in saga file or internal namespace
public class StartEnsureMetadataForVerjiRoomSagaCommand
{
    public string CorrelationId { get; }
    public string RoomId { get; }
    public string SenderId { get; }

    public StartEnsureMetadataForVerjiRoomSagaCommand(
        string correlationId, string roomId, string senderId)
    {
        CorrelationId = correlationId;
        RoomId = roomId;
        SenderId = senderId;
    }
}

// Usage in saga
await context.SendLocal(new StartEnsureMetadataForVerjiRoomSagaCommand(
    correlationId, evt.RoomId, evt.SenderId));
```

**Use `SendLocal()` when:**
- Command is only handled by this saga
- Command controls internal workflow
- Command is not in Abstractions assembly

#### Public Commands (Send)

Public commands delegate work to external handlers in other services:

```csharp
// Public command - defined in *.Esb.Abstractions assembly
// In Verji.ItOps.Matrix.Esb.Abstractions.Commands.RoomMembership
public class EnsureVsysIsInRoomCommand
{
    public string CorrelationId { get; }
    public string RoomId { get; }
    public string VsysUserId { get; }
    public string RoomUserId { get; }

    // ... constructor
}

// Usage in saga
await context.Send(new EnsureVsysIsInRoomCommand(
    Data.CorrelationId, Data.RoomId, Data.VsysUserId, Data.RoomUserId));
```

**Use `Send()` when:**
- Command is handled by another service/handler
- Command is in Abstractions assembly
- Command represents public contract

### Saga Structure

```csharp
public class YourSaga : Saga<YourSagaData>,
    IAmStartedByMessages<TriggerEvent>,      // Creates new saga instance
    IHandleMessages<YourCommand>,            // Handles subsequent messages
    IHandleMessages<CompletionEvent>,        // Handles completion
    IHandleTimeouts<YourTimeout>             // Handles timeout
{
    private readonly ILogger _logger;
    private readonly IDocumentStore _docStore;
    private readonly IResolveSessions _sessionResolver;
    private readonly string _dbTenant;

    public YourSaga(
        ILogger logger,
        IDocumentStore docStore,
        IResolveSessions sessionResolver,
        IResolveTenants tenantResolver)
    {
        _logger = logger;
        _docStore = docStore;
        _sessionResolver = sessionResolver;
        _dbTenant = tenantResolver.TenantId;
    }

    protected override void ConfigureHowToFindSaga(
        SagaPropertyMapper<YourSagaData> mapper)
    {
        mapper.MapSaga(saga => saga.CorrelationId)
            .ToMessage<TriggerEvent>(msg => msg.CorrelationId)
            .ToMessage<YourCommand>(msg => msg.CorrelationId)
            .ToMessage<CompletionEvent>(msg => msg.CorrelationId);
    }
}
```

### Multiple Trigger Events

Sagas can be started by different events, useful when multiple paths lead to the same workflow, or there may be races between events related to the same saga:

```csharp
public class EnsureMetadataForVerjiRoomSaga : Saga<EnsureMetadataForVerjiRoomSagaData>,
    IAmStartedByMessages<IRoomCreated>,         // Started when room is created
    IAmStartedByMessages<ISpaceChildChanged>,   // Started when space relationship changes
    IAmStartedByMessages<ISpaceParentChanged>,  // Started when parent changes
    IHandleMessages<YourCommand>
{
    protected override void ConfigureHowToFindSaga(
        SagaPropertyMapper<EnsureMetadataForVerjiRoomSagaData> mapper)
    {
        mapper.MapSaga(saga => saga.CorrelationId)
            .ToMessage<IRoomCreated>(msg => msg.RoomId)           // Use RoomId as correlation
            .ToMessage<ISpaceChildChanged>(msg => msg.ChildRoomId) // Child room is correlation
            .ToMessage<ISpaceParentChanged>(msg => msg.RoomId);    // Room ID is correlation
    }

    public async Task Handle(IRoomCreated evt, IMessageHandlerContext context)
    {
        var handlerType = evt.GetType().Name.Split(".").Last().Replace("__impl", "");
        var correlationId = evt.RoomId; // Room ID becomes saga correlation ID
        _logger.Information("{HandlerType}/{CorrelationId}: Starting ...", handlerType, correlationId);

        Data.CorrelationId = correlationId;
        Data.RoomId = evt.RoomId;

        // All trigger events converge to same internal command
        await context.SendLocal(new StartYourSagaCommand(correlationId, evt.RoomId));

        _logger.Information("{HandlerType}/{CorrelationId}: Done ...", handlerType, correlationId);
    }

    // Similar handlers for ISpaceChildChanged and ISpaceParentChanged
}
```

**Pattern**: Multiple triggers converge to a common internal command that starts the actual workflow.

### Pause/Resume Pattern for Async Workflows

When sagas need to wait for events that may not arrive, use the pause/resume pattern:

```csharp
public class YourSagaData : ContainSagaData
{
    public bool SagaHasBeenTriggered { get; set; }
    public bool SagaHasBeenPaused { get; set; }
    public bool HaveCheckedStep1 { get; set; }
    public bool HaveCheckedStep2 { get; set; }
}

public async Task Handle(StartYourSagaCommand cmd, IMessageHandlerContext context)
{
    if (!Data.SagaHasBeenTriggered)
    {
        Data.SagaHasBeenTriggered = true;
        Data.CorrelationId = cmd.CorrelationId;

        await DoSagaWorkAsync(handlerType, context);
    }
}

public async Task Handle(ContinueYourSagaCommand cmd, IMessageHandlerContext context)
{
    if (Data.SagaHasBeenPaused)
    {
        Data.SagaHasBeenPaused = false;
        await DoSagaWorkAsync(handlerType, context);
    }
}

private async Task DoSagaWorkAsync(string handlerType, IMessageHandlerContext context)
{
    // Step 1: Check if we can proceed
    if (!Data.HaveCheckedStep1)
    {
        if (!CanResolveStep1())
        {
            // Pause saga, wait for event or timeout
            await RequestTimeout(context, TimeSpan.FromSeconds(30), new WaitForStep1Timeout());
            Data.SagaHasBeenPaused = true;
            return; // Exit and wait
        }
        Data.HaveCheckedStep1 = true;
    }

    // Step 2: Continue only if Step 1 completed
    if (!Data.HaveCheckedStep2)
    {
        if (!CanResolveStep2())
        {
            await RequestTimeout(context, TimeSpan.FromSeconds(30), new WaitForStep2Timeout());
            Data.SagaHasBeenPaused = true;
            return;
        }
        Data.HaveCheckedStep2 = true;
    }

    // All steps complete
    _logger.Information("{HandlerType}/{CorrelationId}: All steps complete", handlerType, Data.CorrelationId);
    MarkAsComplete();
}

// Event handler that resumes saga
public async Task Handle(IRequiredEventArrived evt, IMessageHandlerContext context)
{
    Data.SomeValue = evt.Value;
    await context.SendLocal(new ContinueYourSagaCommand(Data.CorrelationId));
}

// Timeout handler with fallback
public async Task Timeout(WaitForStep1Timeout timeout, IMessageHandlerContext context)
{
    // Check if event arrived while waiting
    if (CanResolveStep1())
    {
        await context.SendLocal(new ContinueYourSagaCommand(Data.CorrelationId));
        return;
    }

    // Try SDK fallback
    var result = await _sdk.TryGetValueAsync();
    if (result != null)
    {
        Data.SomeValue = result.Value;
        await context.SendLocal(new ContinueYourSagaCommand(Data.CorrelationId));
    }
    else
    {
        _logger.Error("{CorrelationId}: Could not resolve step 1. Aborting saga.", Data.CorrelationId);
        MarkAsComplete();
    }
}
```

**Key Points:**
1. Use `SagaHasBeenPaused` flag to prevent multiple resume attempts
2. Exit `DoSagaWorkAsync` early when paused
3. Resume via internal `ContinueYourSagaCommand`
4. Timeouts provide fallback mechanism

### Timeout Handling with Fallback Strategies

Sagas should handle missing events gracefully with multiple fallback strategies:

```csharp
public async Task Timeout(WaitForPersonalSpaceEventTimeout timeout, IMessageHandlerContext context)
{
    var handlerType = timeout.GetType().Name.Split(".").Last();
    _logger.Information("{HandlerType}/{CorrelationId}: Starting ...", handlerType, Data.CorrelationId);

    // Strategy 1: Check if event arrived while we were waiting
    if (await CanResolvePersonalSpace(handlerType, context))
    {
        _logger.Information("{HandlerType}/{CorrelationId}: Event arrived, continuing saga",
            handlerType, Data.CorrelationId);
        await context.SendLocal(new ContinueYourSagaCommand(Data.CorrelationId));
        return;
    }

    // Strategy 2: Try manual SDK lookup / hierarchy traversal
    var parentSpaceInfo = await _verjiSdk.V2.ResolveParentSpaceIdByHierarchyTraversal(
        Data.RoomUserId, Data.RoomId, _retryPolicy);

    if (parentSpaceInfo is { RoomHadParentLink: true })
    {
        Data.PersonalSpaceIdFromHierarchyTraversalInTimeout = parentSpaceInfo.ParentSpaceId!;
        _logger.Information("{HandlerType}/{CorrelationId}: Resolved via SDK fallback",
            handlerType, Data.CorrelationId);
        await context.SendLocal(new ContinueYourSagaCommand(Data.CorrelationId));
    }
    else
    {
        // Strategy 3: Graceful failure
        _logger.Error("{HandlerType}/{CorrelationId}: All strategies failed. Marking saga complete",
            handlerType, Data.CorrelationId);
        MarkAsComplete();
    }

    _logger.Information("{HandlerType}/{CorrelationId}: Done ...", handlerType, Data.CorrelationId);
}
```

**Fallback Strategies:**
1. Check if required event arrived during timeout
2. Perform SDK lookup or direct query
3. Retry with different parameters
4. Gracefully abort if all strategies fail

### Multi-Step State Machine Pattern

Complex sagas implement state machines with progress tracking:

```csharp
private async Task DoSagaWorkAsync(string handlerType, IMessageHandlerContext context)
{
    // Step 1: Verify room is not a space
    if (!Data.HaveCheckedRoomIsNotSpace)
    {
        if (!CanResolveIfRoomIsSpace(handlerType, context))
        {
            await RequestTimeout(context, TimeSpan.FromSeconds(5), new WaitForRoomTypeTimeout());
            Data.SagaHasBeenPaused = true;
            return;
        }

        if (Data.RoomHasTypeSpace == true)
        {
            _logger.Information("{HandlerType}/{CorrelationId}: Room is a space, completing saga",
                handlerType, Data.CorrelationId);
            MarkAsComplete();
            return;
        }

        Data.HaveCheckedRoomIsNotSpace = true;
    }

    // Step 2: Verify room is not a DM
    if (!Data.HaveCheckedRoomIsNotDmRoom)
    {
        if (!await CanResolveIfRoomIsDmRoom(handlerType, context))
        {
            await RequestTimeout(context, TimeSpan.FromSeconds(5), new WaitForDmInfoTimeout());
            Data.SagaHasBeenPaused = true;
            return;
        }

        if (Data.IsDmRoom == true)
        {
            _logger.Information("{HandlerType}/{CorrelationId}: Room is DM, completing saga",
                handlerType, Data.CorrelationId);
            MarkAsComplete();
            return;
        }

        Data.HaveCheckedRoomIsNotDmRoom = true;
    }

    // Step 3: Ensure Vsys user is in room
    if (!Data.HaveCheckedIfVsysIsInRoom)
    {
        if (!CanResolveIfVsysIsInRoom(handlerType, context))
        {
            await RequestTimeout(context, TimeSpan.FromSeconds(5), new WaitForVsysJoinTimeout());
            Data.SagaHasBeenPaused = true;
            return;
        }

        // If Vsys not in room, send command to invite
        if (Data.VsysIsInRoom == false && !Data.EnsureVsysIsInRoomCommandSent)
        {
            await context.Send(new EnsureVsysIsInRoomCommand(
                Data.CorrelationId, Data.RoomId, Data.VsysUserId, Data.RoomUserId));
            await RequestTimeout(context, TimeSpan.FromMinutes(1), new EnsureVsysInRoomTimeout());

            Data.EnsureVsysIsInRoomCommandSent = true;
            Data.SagaHasBeenPaused = true;

            // Reset flags so we re-check after invite
            Data.HaveCheckedIfVsysIsInRoom = false;
            Data.VsysIsInRoom = null;
            return;
        }

        Data.HaveCheckedIfVsysIsInRoom = true;
    }

    // Continue with remaining steps...

    // Final Step: Mark complete when all conditions met
    if (Data is {
        RoomHasTypeSpace: false,
        IsDmRoom: false,
        VsysIsInRoom: true,
        TenantInfoEnsured: true
    })
    {
        await context.Publish<IYourSagaCompleted>(e => {
            e.CorrelationId = Data.CorrelationId;
            e.RoomId = Data.RoomId;
        });

        _logger.Information("{HandlerType}/{CorrelationId}: All steps complete, marking saga complete",
            handlerType, Data.CorrelationId);
        MarkAsComplete();
    }
}
```

**Pattern Features:**
- Each step has a "HaveChecked" flag in SagaData
- Steps execute sequentially
- Steps can pause and resume
- Steps can trigger commands and wait for responses
- Early exit conditions at each step
- Final validation before completion

## Complete Example: Delegation Pattern

Here's a complete example showing how a saga delegates to external handlers:

```csharp
public class EnsureMetadataForVerjiRoomSaga : Saga<EnsureMetadataForVerjiRoomSagaData>,
    IAmStartedByMessages<IRoomCreated>,
    IHandleMessages<StartEnsureMetadataForVerjiRoomSagaCommand>,
    IHandleMessages<IVsysUserEnsuredToBeInRoom>,
    IHandleMessages<ITenantInfoAddedToRoom>,
    IHandleTimeouts<EnsureVsysInRoomTimeout>
{
    private readonly ILogger _logger;
    private readonly IVerjiSdk _verjiSdk;

    // Saga is started by external event
    public async Task Handle(IRoomCreated evt, IMessageHandlerContext context)
    {
        var correlationId = evt.RoomId;
        Data.CorrelationId = correlationId;
        Data.RoomId = evt.RoomId;

        // Delegate to internal command to start workflow
        await context.SendLocal(new StartEnsureMetadataForVerjiRoomSagaCommand(
            correlationId, evt.RoomId, evt.SenderId));
    }

    private async Task DoSagaWorkAsync(string handlerType, IMessageHandlerContext context)
    {
        // ... previous steps ...

        // Saga determines Vsys needs to be invited
        if (!Data.EnsureVsysIsInRoomCommandSent && Data.VsysIsInRoom == false)
        {
            _logger.Information("{HandlerType}/{CorrelationId}: Delegating to EnsureVsysIsInRoomCommand",
                handlerType, Data.CorrelationId);

            // DELEGATION: Send public command to external handler
            await context.Send(new EnsureVsysIsInRoomCommand(
                Data.CorrelationId,
                Data.RoomId,
                Data.VsysUserId,
                Data.RoomUserId));

            // Set timeout in case handler never responds
            await RequestTimeout(context, TimeSpan.FromMinutes(1), new EnsureVsysInRoomTimeout());

            Data.EnsureVsysIsInRoomCommandSent = true;
            Data.SagaHasBeenPaused = true;
            return; // Wait for response
        }

        // ... continue with next steps ...
    }

    // Handle successful completion event from delegated handler
    public async Task Handle(IVsysUserEnsuredToBeInRoom evt, IMessageHandlerContext context)
    {
        var handlerType = evt.GetType().Name.Split(".").Last().Replace("__impl", "");
        _logger.Information("{HandlerType}/{CorrelationId}: Starting ...", handlerType, Data.CorrelationId);

        // Update saga state based on handler result
        Data.VsysIsInRoom = true;

        // Resume saga workflow
        await context.SendLocal(new ContinueEnsureMetadataForVerjiRoomSagaCommand(Data.CorrelationId));

        _logger.Information("{HandlerType}/{CorrelationId}: Done ...", handlerType, Data.CorrelationId);
    }

    // Handle timeout if delegated handler doesn't respond
    public async Task Timeout(EnsureVsysInRoomTimeout timeout, IMessageHandlerContext context)
    {
        // Check if event arrived while waiting
        if (Data.VsysIsInRoom == true)
        {
            await context.SendLocal(new ContinueEnsureMetadataForVerjiRoomSagaCommand(Data.CorrelationId));
            return;
        }

        _logger.Error("{CorrelationId}: Timeout waiting for Vsys to join room. Aborting saga.",
            Data.CorrelationId);
        MarkAsComplete();
    }
}
```

**Key Delegation Points:**
1. Saga identifies work to be done
2. Saga sends public command to specialized handler
3. Saga sets timeout for handler response
4. Saga pauses until response arrives
5. Handler publishes completion event
6. Saga resumes and continues workflow
7. Timeout provides fallback if handler fails

## Common Patterns Summary

### Saga Orchestration Checklist

- [ ] SagaData contains only orchestration flags and temporary state
- [ ] Domain objects store business-meaningful, queryable data
- [ ] Sessions obtained via `IResolveSessions` with `context` parameter
- [ ] Internal commands use `SendLocal()`, public commands use `Send()`
- [ ] Entry/exit logging with `{HandlerType}/{CorrelationId}` format
- [ ] CorrelationId propagated through all messages
- [ ] Timeouts set for all external delegations
- [ ] Fallback strategies for missing events
- [ ] Graceful saga completion on errors
- [ ] Progress flags prevent duplicate work
- [ ] Business results persisted to domain objects via Marten, or via REST/API calls to external systems

### Transaction Boundaries

**Within a single handler:**
```
┌─────────────────────────────────────┐
│   NServiceBus OutBox Transaction    │
│                                      │
│  ┌────────────────────────────────┐ │
│  │  1. Update SagaData            │ │
│  │  2. Update Domain Objects      │ │
│  │  3. Dispatch Messages          │ │
│  └────────────────────────────────┘ │
│                                      │
│  All succeed or all roll back       │
└─────────────────────────────────────┘
```

**Key Principle**: Never create Marten sessions without passing `context`. This breaks the OutBox transaction and can cause inconsistent state.

## Reference Implementations

Study these sagas for complete examples:

- **TenantToTenantSpaceSyncSaga** (`Verji.ItOps.Esb.Handlers/Spaces/TenantToTenantSpaceSyncSaga.cs`)
  - Complex person sync workflow
  - Conditional command dispatching
  - Integration with PersonIdProvider pattern
  - Whitelisting and filtering logic

- **HierarchyAnalysisSaga** (`Verji.ItOps.Esb.Handlers/HierarchyAnalysis/HierarchyAnalysisSaga.cs`)
  - Iterative processing (room by room)
  - Domain object updates with Marten patches
  - Person caching pattern
  - Progress tracking and logging

- **EnsureMetadataForVerjiRoomSaga** (`Verji.ItOps.Matrix.Esb.Handlers/EnsureMetadataForVerjiRoom/EnsureMetadataForVerjiRoomSaga.cs`)
  - Multiple trigger events
  - Multi-step state machine
  - Pause/resume pattern
  - Extensive timeout handling with fallbacks
  - Delegation to multiple external handlers

