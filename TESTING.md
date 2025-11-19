# Testing Patterns and Conventions

This document describes Verji's patterns and conventions for testing backend services using xUnit, NServiceBus.Testing, Moq, and NFluent.

## Overview

Verji backend services emphasize **integration testing** with real dependencies over isolated unit tests. Tests use real databases, real message handlers, and comprehensive state verification.

**Testing Frameworks:**
- **xUnit** - Test framework
- **NServiceBus.Testing** - Saga and handler testing
- **Moq** - Mocking framework
- **NFluent** - Assertion library (required for new tests)
- **Marten** - Document database for integration tests

**Testing Philosophy:**
- Integration tests over unit tests
- Real database over in-memory/fake
- Test complete workflows, not just individual methods
- Verify state changes and message flows
- Use method names for test isolation

## Assertion Library Policy

### For NEW Tests

**Use NFluent exclusively.**

Reason: FluentAssertions has licensing concerns. NFluent is licensing-friendly and provides similar fluent syntax.

```csharp
// CORRECT: Use NFluent for all new tests
Check.That(result).IsEqualTo(expected);
Check.That(messages).HasSize(1);
Check.That(saga.Data.HasCompleted).IsTrue();
```

### For EXISTING Tests

**FluentAssertions may exist in legacy tests.**

When modifying existing tests:
- Convert FluentAssertions to NFluent
- No mass migration required - incremental conversion only
- Update as you touch each test file

```csharp
// Legacy (FluentAssertions) - convert when touching this test
result.Should().BeEquivalentTo(expected);  // Old

// New standard (NFluent)
Check.That(result).IsEqualTo(expected);  // New
```

### Migration Examples

Common FluentAssertions → NFluent conversions:

```csharp
// Equality
result.Should().Be(expected);              // FluentAssertions
Check.That(result).IsEqualTo(expected);    // NFluent

// Collections
list.Should().BeEmpty();                   // FluentAssertions
Check.That(list).IsEmpty();                // NFluent

list.Should().HaveCount(3);                // FluentAssertions
Check.That(list).HasSize(3);               // NFluent

// Nullability
value.Should().BeNull();                   // FluentAssertions
Check.That(value).IsNull();                // NFluent

value.Should().NotBeNull();                // FluentAssertions
Check.That(value).IsNotNull();             // NFluent

// Type checks
obj.Should().BeOfType<MyType>();           // FluentAssertions
Assert.IsType<MyType>(obj);                // xUnit Assert (simpler for type checks)

// Exceptions
await action.Should().NotThrowAsync();     // FluentAssertions
// Use xUnit Assert.ThrowsAsync or verify manually with NFluent
```

## Test Structure Conventions

### Test Class Naming

**Pattern:** `Test{ClassUnderTest}`

Examples:
```csharp
public class TestOrganizationRepository  // Tests OrganizationRepository
public class TestVerjiRoomMetadataSdk    // Tests VerjiRoomMetadataSdk
public class TestEnsureMetadataForVerjiRoomSaga  // Tests EnsureMetadataForVerjiRoomSaga
```

### Test Class Structure

```csharp
public class TestYourClass
{
    private readonly IConfiguration _config;

    public TestYourClass(ITestOutputHelper output)
    {
        // Load configuration
        _config = new ConfigurationBuilder()
            .AddJsonFile("test-settings.json")
            .Build();

        // Configure Serilog
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .Enrich.FromLogContext()
            .WriteTo.Console(outputTemplate:
                "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] {Message}{NewLine}{Exception}")
            .WriteTo.Debug()
            .WriteTo.TestOutput(output)  // xUnit test output integration
            .CreateLogger();
    }

    [Fact]
    public async Task YourTest()
    {
        // Arrange
        // Act
        // Assert
    }
}
```

**Key Elements:**
- Constructor takes `ITestOutputHelper output` for xUnit integration
- Configuration loaded from `test-settings.json`
- Serilog configured with Console, Debug, and TestOutput sinks
- Private readonly fields for shared config

### Test Method Naming

**Pattern 1:** `{Action}_{Scenario}_{ExpectedOutcome}`

```csharp
[Fact]
public async Task CanGetOrganizationsByIdsNoAccessControl()

[Fact]
public async Task CanGetOrganizationsByIdsWithAccessControl()
```

**Pattern 2:** Descriptive sentence with underscores

```csharp
[Theory]
[InlineData("mxc://new/avatar/url")]
public async Task SyncSpaceProfileDataSaga_2LevelHierarchy_CanonicalSpace_Avatar_IdealFlowTest(
    string? newAvatarUrl)

[Theory]
[InlineData(DoWorkStepFlow.CannotResolve, true, null, null)]
public async Task StartOrContinueCommand_Step1_RoomHasTypeSpace(
    DoWorkStepFlow expectedFlow,
    bool startCommand,
    bool? roomHasTypeSpaceFromEvent)
```

### Arrange-Act-Assert Pattern

**All tests must follow strict AAA pattern with clear comments:**

```csharp
[Fact]
public async Task SagaWillBeStartedByCommand()
{
    // Arrange
    var logger = Log.Logger;
    var verjiSdk = new Mock<IVerjiSdk>();
    var saga = new YourSaga(logger, verjiSdk.Object)
    {
        Data = new YourSagaData { }
    };

    var command = new StartCommand(correlationId, roomId);
    var context = new TestableMessageHandlerContext();

    // Act
    await saga.Handle(command, context);

    // Assert
    Check.That(context.SentMessages).HasSize(1);
    Check.That(saga.Data.HasBeenTriggered).IsTrue();
    Check.That(saga.Completed).IsFalse();
}
```

## Saga Testing Patterns

### NServiceBus.Testing Framework

**TestableSaga Pattern:**

```csharp
using NServiceBus.Testing;

var saga = new TestableSaga<YourSaga, YourSagaData>(() =>
{
    return new YourSaga(logger, dependency.Object);
});
```

**Direct Instantiation (Alternative):**

```csharp
var saga = new YourSaga(logger, dependency.Object)
{
    Data = new YourSagaData
    {
        CorrelationId = correlationId,
        SomeProperty = initialValue
    }
};
```

### Testing Saga Handlers Step-by-Step

Sagas often involve multiple handlers and external command handlers. Test the complete flow:

```csharp
[Theory]
[InlineData("mxc://new/avatar")]
public async Task SagaFlow_CompleteWorkflow(string newAvatarUrl)
{
    // Arrange
    var logger = Log.Logger;
    var verjiSdk = SetupVerjiSdkMock();
    var roomId = "!room:server";
    var correlationId = Guid.NewGuid().ToString();

    // Step 1: Test dispatcher that starts the saga
    var dispatcher = new AvatarChangedDispatcher(logger, verjiSdk.Object);
    var dispatcherContext = new TestableMessageHandlerContext();
    var avatarChangedEvent = new TestRoomAvatarChanged(roomId, newAvatarUrl);

    await dispatcher.Handle(avatarChangedEvent, dispatcherContext);

    // Verify dispatcher sent command
    Check.That(dispatcherContext.SentMessages).HasSize(1);
    var startCommand = dispatcherContext.SentMessages[0].Message as StartSagaCommand;
    Assert.NotNull(startCommand);
    Check.That(startCommand.NewAvatarUrl).IsEqualTo(newAvatarUrl);

    // Step 2: Saga handles start command
    var saga = new TestableSaga<YourSaga, YourSagaData>(() =>
    {
        return new YourSaga(logger, verjiSdk.Object);
    });

    var startContext = new TestableMessageHandlerContext();
    var startResult = await saga.Handle(startCommand, startContext);

    // Verify saga state
    Check.That(startResult.Completed).IsFalse();
    Check.That(saga.Data.NewAvatarUrl).IsEqualTo(newAvatarUrl);
    Check.That(startContext.SentMessages).HasSize(1);

    // Step 3: External handler processes command
    var externalHandler = new ProcessAvatarCommandHandler(logger, verjiSdk.Object);
    var externalContext = new TestableMessageHandlerContext();
    var processCommand = startContext.SentMessages[0].Message as ProcessAvatarCommand;

    await externalHandler.Handle(processCommand, externalContext);

    // Verify external handler published event
    Check.That(externalContext.PublishedMessages).HasSize(1);
    var completedEvent = externalContext.PublishedMessages[0].Message as IAvatarProcessedEvent;
    Assert.NotNull(completedEvent);

    // Step 4: Saga handles completion event
    var finalContext = new TestableMessageHandlerContext();
    var finalResult = await saga.Handle(completedEvent, finalContext);

    // Verify saga completed
    Check.That(finalResult.Completed).IsTrue();
    Check.That(saga.Data.ProcessingCompleted).IsTrue();
}
```

### Testing Saga State

**Setup and Verify Saga State:**

```csharp
[Fact]
public async Task SagaStateTransition()
{
    // Arrange - Set up initial state
    var saga = new YourSaga(logger, sdk.Object)
    {
        Data = new YourSagaData
        {
            CorrelationId = correlationId,
            SagaHasBeenTriggered = false,
            HaveCheckedStep1 = false,
            Step1Value = null
        }
    };

    var command = new ContinueCommand(correlationId);
    var context = new TestableMessageHandlerContext();

    // Act
    await saga.Handle(command, context);

    // Assert - Verify state changed
    Check.That(saga.Data.SagaHasBeenTriggered).IsTrue();
    Check.That(saga.Data.HaveCheckedStep1).IsTrue();
    Check.That(saga.Data.Step1Value).IsEqualTo(expectedValue);
    Check.That(saga.Completed).IsFalse();
}
```

### Testing Sent/Published/Timeout Messages

**Verify Messages in TestableMessageHandlerContext:**

```csharp
// Assert sent messages
Check.That(context.SentMessages).HasSize(1);
Check.That(context.SentMessages[0].Message).IsInstanceOf<YourCommand>();

var sentCommand = context.SentMessages[0].Message as YourCommand;
Assert.NotNull(sentCommand);
Check.That(sentCommand.CorrelationId).IsEqualTo(correlationId);

// Assert published events
Check.That(context.PublishedMessages).HasSize(2);
var firstEvent = context.PublishedMessages[0].Message as IYourEvent;
Assert.NotNull(firstEvent);

// Assert timeout messages
Check.That(context.TimeoutMessages).HasSize(1);
var timeout = context.TimeoutMessages[0].Message as YourTimeout;
Assert.NotNull(timeout);
Check.That(saga.Data.SagaHasBeenPaused).IsTrue();

// Assert no messages
Check.That(context.SentMessages).IsEmpty();
Check.That(context.PublishedMessages).IsEmpty();
```

### Custom Test Event Classes

**Create test implementations for interface-based events:**

```csharp
#region Test classes for mocking events

private abstract class TestRoomChangedBase(string roomId) : IMxBaseClientRoomEvent
{
    public string RoomId { get; set; } = roomId;
    public string Period { get; set; } = string.Empty;
    public string EventId { get; set; } = Guid.NewGuid().ToString();
    public string SenderId { get; set; } = "@sender:server";
    public long Timestamp { get; set; } = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
}

private class TestRoomAvatarChanged(string roomId, string? newAvatarUrl)
    : TestRoomChangedBase(roomId), IRoomAvatarChanged
{
    public string? AvatarUrl { get; set; } = newAvatarUrl;
}

private class TestRoomNameChanged(string roomId, string? newName)
    : TestRoomChangedBase(roomId), IRoomNameChanged
{
    public string? Name { get; set; } = newName;
}

#endregion
```

### NServiceBus MessageMapper

**For creating interface-based event instances:**

```csharp
using NServiceBus;

var messageMapper = new MessageMapper();

var roomCreatedEvent = messageMapper.CreateInstance<IRoomCreated>(e =>
{
    e.RoomId = roomId;
    e.SenderId = personId;
    e.Content = [];
    e.RoomType = "m.room";
});

// Use in test
await saga.Handle(roomCreatedEvent, context);
```

## Repository and Database Testing

### Real Database Integration Tests

**Initialize Test Database:**

```csharp
[Fact]
public async Task CanStoreAndRetrieveOrganizations()
{
    // Arrange - Setup database
    var testContext = GetMyMethodName();  // Unique per test method
    var builder = GetBuilderForTenant(testContext);
    var container = builder.Build();

    await TestExtensions.InitTestDbAsync(_config, "unittest");
    TestExtensions.EnsureConnectionPossible(_config, container);

    var docStore = container.Resolve<IDocumentStore>();
    var repo = container.Resolve<IOrganizationRepository>();
    var tenantId = container.Resolve<IResolveTenants>().TenantId;

    // Create test data
    var organizations = await TestData.Organization.CreateManyOrganizations(5, "org_", 1, 1, 1);
    var orgIds = organizations.Select(o => o.Id).ToArray();

    // Act - Store data
    await using (var session = docStore.LightweightSession(tenantId))
    {
        foreach (var org in organizations)
        {
            await repo.StoreOrUpdate(session, org);
        }
        await session.SaveChangesAsync();
    }

    // Assert - Retrieve and verify
    await using (var session = docStore.LightweightSession(tenantId))
    {
        var retrieved = await repo.GetByIds<Organization>(session, orgIds);
        Check.That(retrieved).HasSize(5);
        Check.That(retrieved.Select(o => o.Id)).ContainsExactly(orgIds);
    }
}
```

### Test Isolation with Method Names

**Use method name as unique test context/tenant:**

```csharp
[MethodImpl(MethodImplOptions.NoInlining)]
public static string GetMyMethodName()
{
    var st = new StackTrace();
    var sf = st.GetFrame(4);  // Adjust frame number as needed
    return sf?.GetMethod()?.Name ?? "UnknownTest";
}

// Usage in test
var testContext = GetMyMethodName();  // Returns "CanStoreAndRetrieveOrganizations"
var builder = GetBuilderForTenant(testContext, _config);
```

**Benefits:**
- Each test runs in isolated tenant/database partition
- No need for explicit cleanup between tests
- Parallel test execution without conflicts
- Clear test context in logs

### Autofac Container Setup

```csharp
public ContainerBuilder GetBuilderForTenant(string tenantId, IConfiguration config)
{
    var builder = new ContainerBuilder();

    // Register modules
    builder.RegisterModule(new MartenTestIoCModule(config));
    builder.RegisterModule(new YourTestModule(config));

    // Register tenant resolver
    builder.RegisterInstance(new LiteralTenancyResolver(tenantId))
        .As<IResolveTenants>();

    // Register other test dependencies
    builder.RegisterLogger();

    return builder;
}
```

### Database Session Management

**Pattern:**

```csharp
// Write data
await using (var session = docStore.LightweightSession(tenantId))
{
    repo.Store(session, aggregate);
    await session.SaveChangesAsync();  // Commit
}

// Read data (separate session)
await using (var session = docStore.LightweightSession(tenantId))
{
    var result = await repo.GetById<MyAggregate>(session, id);
    Check.That(result).IsNotNull();
    Check.That(result.SomeProperty).IsEqualTo(expectedValue);
}
```

### Database Cleanup

**Option 1: DeleteWhere with session action**

```csharp
await TestExtensions.RunSessionActionForTenantAsync(_config, container, session =>
{
    session.DeleteWhere<Organization>(org => true);  // Delete all
});
```

**Option 2: Test isolation (preferred)**

Use unique tenant ID per test via `GetMyMethodName()` - no cleanup needed.

### Test Data Creation

```csharp
// Use test data helpers
var organizations = await TestData.Organization.CreateManyOrganizations(
    count: 5,
    prefix: "test_org_",
    param1: 1,
    param2: 1,
    param3: 1);

// Create specific test aggregates
var testOrg = new Organization("test-org-1")
{
    Name = "Test Organization",
    Status = OrganizationStatus.Active,
    Metadata = new AggregateMetadata
    {
        Created = DateTimeOffset.UtcNow
    }
};
```

## Mocking Patterns

### Moq Basics

**Setup method returns:**

```csharp
using Moq;

var verjiSdk = new Mock<IVerjiSdk>();

// Simple return value
verjiSdk.Setup(sdk => sdk.GetRoomId())
    .Returns("!room:server");

// Async return
verjiSdk.Setup(sdk => sdk.GetRoomAsync(It.IsAny<string>()))
    .ReturnsAsync(new Room { Id = roomId });

// Property getter
verjiSdk.SetupGet(sdk => sdk.V2)
    .Returns(verjiSdkV2.Object);

// Conditional returns
verjiSdk.Setup(sdk => sdk.IsVsysUser(It.IsAny<string>()))
    .Returns((string userId) => userId.StartsWith("@vsys:"));
```

### It.IsAny and Argument Matchers

```csharp
// Any value of type
csClient.Setup(c => c.SetRoomAvatar(
    It.IsAny<SetRoomAvatarRequest>(),  // Any request
    roomId,                             // Specific room ID
    It.IsAny<string>(),                 // Any access token
    It.IsAny<IAsyncPolicy>(),           // Any policy
    null))
    .ReturnsAsync(new RestResultWrapper<SetRoomAvatarResponse>
    {
        Outcome = RestOutcome.Success,
        Result = new SetRoomAvatarResponse { Url = newAvatarUrl }
    });

// Not null matcher
adminApi.Setup(a => a.GetRoomState(
    roomId,
    It.IsNotNull<string>(),  // Access token must not be null
    It.IsNotNull<string>(),  // Policy must not be null
    null))
    .ReturnsAsync(roomStateResponse);
```

### Nested Mocking with IIndex<string, T>

**Mocking Autofac IIndex for keyed services:**

```csharp
// Create mock for indexed service
var csApiMock = new Mock<ICsApiClient>();
csApiMock.Setup(c => c.ApplicationServiceLogin(
    It.IsNotNull<ApplicationServiceLoginRequest>(),
    It.IsNotNull<string>(),
    It.IsNotNull<string>(),
    null))
    .ReturnsAsync(loginResponse);

// Create IIndex mock
var csApiClientIndexMock = new Mock<IIndex<string, ICsApiClient>>();
csApiClientIndexMock.SetupGet(idx => idx[It.IsAny<string>()])
    .Returns(csApiMock.Object);

// Multiple indexed services
var adminApiIndexMock = new Mock<IIndex<string, IAdminApiClient>>();
adminApiIndexMock.SetupGet(idx => idx[It.IsAny<string>()])
    .Returns(adminApiMock.Object);

var vmxServicesApiIndexMock = new Mock<IIndex<string, IVmxServicesApiClient>>();
vmxServicesApiIndexMock.SetupGet(idx => idx[It.IsAny<string>()])
    .Returns(new Mock<IVmxServicesApiClient>().Object);

// Use in constructor
var sdk = new VerjiSdk(
    config,
    logger,
    mapper,
    appServiceConfig,
    adminApiIndexMock.Object,
    csApiClientIndexMock.Object,
    vmxServicesApiIndexMock.Object);
```

### Partial Mocking with CallBase

**Mock some methods while preserving real implementation:**

```csharp
var verjiSdkMock = new Mock<VerjiSdk>(
    new VerjiSdkV2(
        "api-variant",
        config,
        logger,
        mapper,
        appServiceConfig,
        adminApiIndexMock.Object,
        csApiClientIndexMock.Object,
        vmxServicesApiIndexMock.Object
    ));

// Use real implementation for V2 property
verjiSdkMock.SetupGet(m => m.V2).CallBase();

// Mock specific methods
verjiSdkMock.Setup(m => m.CustomMethod())
    .Returns("mocked value");

var verjiSdk = verjiSdkMock.Object;
```

### RestResultWrapper Pattern

**Helper for creating API response wrappers:**

```csharp
private RestResultWrapper<TResult> GetResultWrapperWithResult<TResult>(
    HttpStatusCode resultStatusCode,
    Func<TResult> getResultFunc)
{
    return new RestResultWrapper<TResult>
    {
        Details = new OutcomeDetails
        {
            StatusCode = resultStatusCode,
            ReasonPhrase = resultStatusCode.ToString()
        },
        Outcome = resultStatusCode == HttpStatusCode.OK
            ? RestOutcome.Success
            : RestOutcome.Fail,
        Result = getResultFunc.Invoke(),
    };
}

// Usage
var roomStateResponse = GetResultWrapperWithResult(
    HttpStatusCode.OK,
    () => TestData.GetMockRoomStateWithSpaceParent(parentRoomId));

adminApiMock.Setup(a => a.GetRoomState(roomId, token, policy, null))
    .ReturnsAsync(roomStateResponse);
```

### AutoMapper Stub

**Create mapper for tests:**

```csharp
private IMapper GetAutoMapperStub()
{
    var mapperConfig = new MapperConfiguration(cfg =>
    {
        cfg.AddProfile(new YourMappingProfile());
        cfg.AddProfile(new AnotherMappingProfile());
    });

    return mapperConfig.CreateMapper();
}

// Use in test
var mapper = GetAutoMapperStub();
var dto = mapper.Map<YourDto>(aggregate);
```

### Mock Verification

**Verify methods were called:**

```csharp
// Verify called exactly once
adminApiMock.Verify(
    api => api.GetRoomState(roomId, It.IsAny<string>(), It.IsAny<string>(), null),
    Times.Once);

// Verify called specific number of times
csApiMock.Verify(
    c => c.ApplicationServiceLogin(
        It.IsNotNull<ApplicationServiceLoginRequest>(),
        It.IsNotNull<string>(),
        It.IsNotNull<string>(),
        null),
    Times.Exactly(2));

// Verify never called
sdkMock.Verify(
    s => s.DeleteRoom(It.IsAny<string>()),
    Times.Never);

// Verify called at least once
repoMock.Verify(
    r => r.Store(It.IsAny<IDocumentSession>(), It.IsAny<MyAggregate>()),
    Times.AtLeastOnce);
```

## Assertion Patterns

### NFluent (Required for New Tests)

**Collection Assertions:**

```csharp
Check.That(context.SentMessages).HasSize(1);
Check.That(context.PublishedMessages).IsEmpty();
Check.That(list).Contains(expectedItem);
Check.That(list).ContainsExactly(item1, item2, item3);
Check.That(ids).ContainsExactly(expectedIds);
```

**Equality Assertions:**

```csharp
Check.That(result).IsEqualTo(expected);
Check.That(saga.Data.CorrelationId).Equals(correlationId);
Check.That(command.NewValue).IsEqualTo("expected value");
```

**Boolean Assertions:**

```csharp
Check.That(saga.Data.HasBeenTriggered).IsTrue();
Check.That(saga.Completed).IsFalse();
Check.That(isValid).IsTrue();
```

**Null Assertions:**

```csharp
Check.That(result).IsNull();
Check.That(aggregate).IsNotNull();
```

**Type Assertions:**

```csharp
Check.That(message).IsInstanceOf<YourCommand>();
```

**Numeric Comparisons:**

```csharp
Check.That(count).IsStrictlyGreaterThan(0);
Check.That(value).IsLessThan(100);
Check.That(percentage).IsStrictlyPositive();
```

### xUnit Assert (Secondary)

**Use for simple type checks and null checks:**

```csharp
// Type checks
Assert.IsType<YourCommand>(message);
Assert.IsAssignableFrom<IYourInterface>(result);

// Null checks
Assert.NotNull(sentCommand);
Assert.Null(optional Value);

// Collections
Assert.Single(context.SentMessages);
Assert.Empty(context.PublishedMessages);

// Equality (when NFluent not available)
Assert.Equal(expected, actual);
Assert.NotEqual(unwanted, actual);
```

**Assert.Multiple for Grouped Assertions:**

```csharp
Assert.Multiple(() =>
{
    Assert.Equal(newAvatarUrl, command.NewAvatarUrl);
    Assert.Equal(originatingRoomId, command.OriginatingRoomId);
    Assert.Equal(SyncProperty.Avatar, command.ProfileDataProperty);
});
```

### FluentAssertions (Legacy Only)

**Do NOT use in new tests. Convert to NFluent when modifying existing tests.**

```csharp
// Legacy patterns (for understanding existing tests only)
result.Should().BeEquivalentTo(expected);  // Convert to Check.That(result).IsEqualTo(expected)
list.Should().HaveCount(3);                // Convert to Check.That(list).HasSize(3)
value.Should().BeNull();                   // Convert to Check.That(value).IsNull()
await action.Should().NotThrowAsync();     // Verify manually or use xUnit Assert
```

## Theory and Parameterized Testing

### Simple Theory with InlineData

```csharp
[Theory]
[InlineData("@vsys:verji.local", true)]
[InlineData("@user:verji.local", false)]
[InlineData("", false)]
[InlineData(null, false)]
public void TestIsVsysUser(string userId, bool expectedResult)
{
    // Arrange
    var verjiSdk = new VerjiSdk(config, logger, mapper);

    // Act
    var actualResult = verjiSdk.IsVsysUser(userId);

    // Assert
    Check.That(actualResult).IsEqualTo(expectedResult);
}
```

### Complex Theory with Enums

**Use enums to represent test scenarios:**

```csharp
public enum DoWorkStepFlow
{
    CanResolveWithNeededData = 0,        // Got data, continue
    CanResolveDataImpliesExit = 1,       // Got data, exit saga
    CanResolveNeedsToFireCommand = 2,    // Need to fire command
    CannotResolve = 3,                   // Cannot resolve, timeout
}

[Theory]
[InlineData(DoWorkStepFlow.CannotResolve, true, null, null)]
[InlineData(DoWorkStepFlow.CanResolveDataImpliesExit, true, true, null)]
[InlineData(DoWorkStepFlow.CanResolveWithNeededData, true, false, null)]
[InlineData(DoWorkStepFlow.CanResolveDataImpliesExit, false, null, true)]
public async Task SagaStep_Handles_VariousScenarios(
    DoWorkStepFlow expectedFlow,
    bool startCommand,
    bool? dataFromEvent,
    bool? dataFromTimeout)
{
    // Arrange based on expectedFlow
    Mock<IVerjiSdk> verjiSdk;
    switch (expectedFlow)
    {
        case DoWorkStepFlow.CanResolveDataImpliesExit:
            verjiSdk = CreateMockForExitScenario();
            break;
        case DoWorkStepFlow.CanResolveWithNeededData:
            verjiSdk = CreateMockForContinueScenario();
            break;
        case DoWorkStepFlow.CannotResolve:
            verjiSdk = CreateMockForTimeoutScenario();
            break;
        default:
            throw new ArgumentException($"Unexpected flow: {expectedFlow}");
    }

    var saga = new YourSaga(logger, verjiSdk.Object)
    {
        Data = new YourSagaData
        {
            DataFromEvent = dataFromEvent,
            DataFromTimeout = dataFromTimeout,
            SagaHasBeenTriggered = !startCommand
        }
    };

    var command = startCommand
        ? new StartCommand(correlationId)
        : new ContinueCommand(correlationId);
    var context = new TestableMessageHandlerContext();

    // Act
    await saga.Handle(command, context);

    // Assert based on expectedFlow
    switch (expectedFlow)
    {
        case DoWorkStepFlow.CanResolveWithNeededData:
            Check.That(saga.Data.ResolvedData).IsEqualTo(false);
            Check.That(saga.Data.HaveCheckedStep).IsTrue();
            Check.That(saga.Completed).IsFalse();
            break;

        case DoWorkStepFlow.CanResolveDataImpliesExit:
            Check.That(saga.Data.ResolvedData).IsEqualTo(true);
            Check.That(saga.Completed).IsTrue();
            break;

        case DoWorkStepFlow.CannotResolve:
            Check.That(saga.Data.ResolvedData).IsNull();
            Check.That(context.TimeoutMessages).HasSize(1);
            Check.That(saga.Data.SagaHasBeenPaused).IsTrue();
            break;
    }
}
```

### Theory with Nullable Parameters

```csharp
[Theory]
[InlineData("", "Created")]
[InlineData(null, "Created")]
[InlineData("tenant-id", "")]
[InlineData("tenant-id", null)]
[InlineData("tenant-id", "InvalidStatus")]
public async Task TestMethod_HandlesInvalidInput(
    string tenantId,
    string status)
{
    // Arrange
    var sdk = CreateMockSdk(tenantId, status);

    // Act & Assert - Should not throw
    await sdk.Invoking(s => s.GetTenantInfoAsync(tenantId))
        .Should().NotThrowAsync();  // Legacy FluentAssertions
    // Convert to: var result = await sdk.GetTenantInfoAsync(tenantId); // Manual check
}
```

### When to Use Theory vs Fact

**Use `[Theory]` when:**
- Testing same logic with multiple input/output combinations
- Testing edge cases (null, empty, invalid values)
- Testing state transitions with different starting states
- Reducing test code duplication

**Use `[Fact]` when:**
- Testing a single, specific scenario
- Complex setup that varies significantly between cases
- Test readability suffers from parameterization
- Testing complete workflows (arrange is too different per case)

## Complete Examples

### Example 1: Saga Flow Test (Step-by-Step)

```csharp
[Theory]
[InlineData("mxc://new/avatar/url")]
[InlineData(null)]
public async Task SyncSpaceAvatarSaga_IdealFlow(string? newAvatarUrl)
{
    // Arrange
    var logger = Log.Logger;
    var mapper = GetAutoMapperStub();
    var originatingRoomId = "!originating:server";
    var canonicalSpaceId = "!canonical:server";
    var personalSpaceId = "!personal:server";
    var correlationId = Guid.NewGuid().ToString();

    var verjiSdk = SetupVerjiSdkMock(
        mapper,
        logger,
        originatingRoomId,
        canonicalSpaceId,
        personalSpaceId);

    // Step 1: Dispatcher receives event and sends command
    var dispatcher = new AvatarChangedDispatcher(logger, verjiSdk.Object);
    var dispatcherContext = new TestableMessageHandlerContext();
    var roomAvatarChangedEvent = new TestRoomAvatarChanged(originatingRoomId, newAvatarUrl);

    await dispatcher.Handle(roomAvatarChangedEvent, dispatcherContext);

    Check.That(dispatcherContext.SentMessages).HasSize(1);
    var startCommand = dispatcherContext.SentMessages[0].Message as StartSyncAvatarCommand;
    Assert.NotNull(startCommand);
    Check.That(startCommand.NewAvatarUrl).IsEqualTo(newAvatarUrl);
    Check.That(startCommand.OriginatingRoomId).IsEqualTo(originatingRoomId);

    // Step 2: Saga handles start command
    var saga = new TestableSaga<SyncSpaceAvatarSaga, SyncSpaceAvatarSagaData>(() =>
    {
        return new SyncSpaceAvatarSaga(logger, verjiSdk.Object);
    });

    var startContext = new TestableMessageHandlerContext();
    var startResult = await saga.Handle(startCommand, startContext);

    Check.That(startResult.Completed).IsFalse();
    Check.That(saga.Data.NewAvatarUrl).IsEqualTo(newAvatarUrl);
    Check.That(startContext.SentMessages).HasSize(1);

    var syncCanonicalCommand = startContext.SentMessages[0].Message as SyncCanonicalSpaceCommand;
    Assert.NotNull(syncCanonicalCommand);

    // Step 3: External handler processes canonical space
    var canonicalHandler = new SyncCanonicalSpaceHandler(logger, verjiSdk.Object);
    var canonicalContext = new TestableMessageHandlerContext();

    await canonicalHandler.Handle(syncCanonicalCommand, canonicalContext);

    Check.That(canonicalContext.PublishedMessages).HasSize(1);
    var canonicalSyncedEvent = canonicalContext.PublishedMessages[0].Message as ICanonicalSpaceSynced;
    Assert.NotNull(canonicalSyncedEvent);

    // Step 4: Saga handles canonical synced event
    var handleCanonicalContext = new TestableMessageHandlerContext();
    var handleCanonicalResult = await saga.Handle(canonicalSyncedEvent, handleCanonicalContext);

    Check.That(handleCanonicalResult.Completed).IsFalse();
    Check.That(saga.Data.CanonicalSpaceSynced).IsTrue();
    Check.That(handleCanonicalContext.SentMessages).HasSize(1);

    var syncPersonalCommand = handleCanonicalContext.SentMessages[0].Message as SyncPersonalSpaceCommand;
    Assert.NotNull(syncPersonalCommand);

    // Step 5: External handler processes personal space
    var personalHandler = new SyncPersonalSpaceHandler(logger, verjiSdk.Object);
    var personalContext = new TestableMessageHandlerContext();

    await personalHandler.Handle(syncPersonalCommand, personalContext);

    Check.That(personalContext.PublishedMessages).HasSize(1);
    var personalSyncedEvent = personalContext.PublishedMessages[0].Message as IPersonalSpaceSynced;
    Assert.NotNull(personalSyncedEvent);

    // Step 6: Saga handles personal synced event and completes
    var finalContext = new TestableMessageHandlerContext();
    var finalResult = await saga.Handle(personalSyncedEvent, finalContext);

    Check.That(finalResult.Completed).IsTrue();
    Check.That(saga.Data.PersonalSpaceSynced).IsTrue();
    Check.That(saga.Data.CanonicalSpaceSynced).IsTrue();
}
```

### Example 2: Saga State Test with Theory

```csharp
public enum DoWorkStepFlow
{
    CanResolveWithNeededData = 0,
    CanResolveDataImpliesExit = 1,
    CannotResolve = 3,
}

[Theory]
[InlineData(DoWorkStepFlow.CannotResolve, true, null)]
[InlineData(DoWorkStepFlow.CanResolveDataImpliesExit, true, true)]
[InlineData(DoWorkStepFlow.CanResolveWithNeededData, true, false)]
public async Task SagaStep1_RoomHasTypeSpace_VariousScenarios(
    DoWorkStepFlow expectedFlow,
    bool startCommand,
    bool? roomHasTypeSpaceFromEvent)
{
    // Arrange
    var testContext = GetMyMethodName();
    var builder = GetBuilderForTenant(testContext, _config);
    var container = builder.Build();
    var mapper = container.Resolve<IMapper>();
    var logger = container.Resolve<ILogger>();

    var roomId = "!room:server";
    var personId = "@person:server";
    var correlationId = Guid.NewGuid().ToString();

    Mock<IVerjiSdk> verjiSdk;
    switch (expectedFlow)
    {
        case DoWorkStepFlow.CanResolveDataImpliesExit:
            verjiSdk = CreateVerjiSdkMockForSpaceRoom(mapper, logger, roomId, personId);
            break;
        case DoWorkStepFlow.CanResolveWithNeededData:
        case DoWorkStepFlow.CannotResolve:
            verjiSdk = CreateVerjiSdkMockForNormalRoom(mapper, logger, roomId, personId);
            break;
        default:
            throw new ArgumentException($"Unexpected flow: {expectedFlow}");
    }

    var saga = new EnsureMetadataForRoomSaga(logger, verjiSdk.Object)
    {
        Data = new EnsureMetadataForRoomSagaData
        {
            CorrelationId = correlationId,
            RoomId = roomId,
            PersonId = personId,
            SagaHasBeenTriggered = !startCommand,
            HaveCheckedRoomIsNotSpace = false,
            RoomHasTypeSpace = null,
            RoomHasTypeSpaceFromRoomCreateEvent = roomHasTypeSpaceFromEvent
        }
    };

    var command = startCommand
        ? new StartCommand(correlationId, roomId, personId)
        : new ContinueCommand(correlationId);

    var context = new TestableMessageHandlerContext();

    // Act
    await saga.Handle(command, context);

    // Assert based on expectedFlow
    switch (expectedFlow)
    {
        case DoWorkStepFlow.CanResolveWithNeededData:
            Check.That(saga.Data.RoomHasTypeSpace).IsEqualTo(false);
            Check.That(saga.Data.HaveCheckedRoomIsNotSpace).IsTrue();
            Check.That(saga.Completed).IsFalse();
            Check.That(context.SentMessages).HasSize(1);
            break;

        case DoWorkStepFlow.CanResolveDataImpliesExit:
            Check.That(saga.Data.RoomHasTypeSpace).IsEqualTo(true);
            Check.That(saga.Data.HaveCheckedRoomIsNotSpace).IsTrue();
            Check.That(saga.Completed).IsTrue();
            break;

        case DoWorkStepFlow.CannotResolve:
            Check.That(saga.Data.RoomHasTypeSpace).IsNull();
            Check.That(saga.Data.HaveCheckedRoomIsNotSpace).IsFalse();
            Check.That(context.TimeoutMessages).HasSize(1);
            Check.That(saga.Data.SagaHasBeenPaused).IsTrue();
            var timeout = context.TimeoutMessages[0].Message as WaitForRoomTypeTimeout;
            Assert.NotNull(timeout);
            break;
    }
}
```

### Example 3: Repository Integration Test

```csharp
[Fact]
public async Task CanGetOrganizationsByIds_WithAccessControl()
{
    // Arrange - Setup database
    var testContext = GetMyMethodName();
    var builder = GetBuilderForTenant(testContext);
    var container = builder.Build();

    await TestExtensions.InitTestDbAsync(_config, "unittest");
    TestExtensions.EnsureConnectionPossible(_config, container);

    var docStore = container.Resolve<IDocumentStore>();
    var repo = container.Resolve<IOrganizationRepository>();
    var tenantId = container.Resolve<IResolveTenants>().TenantId;

    // Create test data
    var organizations = await TestData.Organization.CreateManyOrganizations(10, "org_", 1, 1, 1);
    var allIds = organizations.Select(o => o.Id).ToArray();
    var oddIds = allIds.Where((id, idx) => idx % 2 == 0).ToArray();
    var evenIds = allIds.Where((id, idx) => idx % 2 == 1).ToArray();

    // Store all organizations
    await using (var session = docStore.LightweightSession(tenantId))
    {
        foreach (var org in organizations)
        {
            await repo.StoreOrUpdate(session, org);
        }
        await session.SaveChangesAsync();
    }

    // Act - Retrieve with access control
    var acContext = new AcContext
    {
        UserId = "test-user",
        ActiveAclDomain = "test-domain",
        ActiveRoles = oddIds  // User has access to odd IDs only
    };

    await using (var session = docStore.LightweightSession(tenantId))
    {
        var accessibleOrgs = await repo.GetByIds<Organization>(
            acContext,
            session,
            allIds);  // Request all, but ACL filters

        // Assert - Only odd IDs returned
        Check.That(accessibleOrgs).HasSize(oddIds.Length);
        Check.That(accessibleOrgs.Select(o => o.Id)).ContainsExactly(oddIds);
    }

    // Cleanup
    await TestExtensions.RunSessionActionForTenantAsync(_config, container, session =>
    {
        session.DeleteWhere<Organization>(org => true);
    });
}
```

### Example 4: SDK Test with Advanced Mocking

```csharp
[Theory]
[InlineData("!room:server", "tenant-123")]
public async Task TestGetTenantInfo_Success(string roomId, string tenantId)
{
    // Arrange - Setup complex mocks
    var logger = Log.Logger;
    var mapper = GetAutoMapperStub();
    var config = _config;

    // Mock AdminApi
    var adminApiMock = new Mock<IAdminApiClient>();
    var roomStateResponse = GetResultWrapperWithResult(
        HttpStatusCode.OK,
        () => TestData.GetMockRoomStateWithTenantInfo(tenantId));

    adminApiMock.Setup(a => a.GetRoomState(
        roomId,
        It.IsNotNull<string>(),
        It.IsNotNull<string>(),
        null))
        .ReturnsAsync(roomStateResponse);

    // Mock CsApi
    var csApiMock = new Mock<ICsApiClient>();
    csApiMock.Setup(c => c.ApplicationServiceLogin(
        It.IsNotNull<ApplicationServiceLoginRequest>(),
        It.IsNotNull<string>(),
        It.IsNotNull<string>(),
        null))
        .ReturnsAsync(new RestResultWrapper<ApplicationServiceLoginResponse>
        {
            Outcome = RestOutcome.Success,
            Result = new ApplicationServiceLoginResponse
            {
                AccessToken = "test-token",
                UserId = "@vsys:server"
            }
        });

    // Create IIndex mocks
    var adminApiIndexMock = new Mock<IIndex<string, IAdminApiClient>>();
    adminApiIndexMock.SetupGet(idx => idx[It.IsAny<string>()])
        .Returns(adminApiMock.Object);

    var csApiClientIndexMock = new Mock<IIndex<string, ICsApiClient>>();
    csApiClientIndexMock.SetupGet(idx => idx[It.IsAny<string>()])
        .Returns(csApiMock.Object);

    var vmxServicesApiIndexMock = new Mock<IIndex<string, IVmxServicesApiClient>>();
    vmxServicesApiIndexMock.SetupGet(idx => idx[It.IsAny<string>()])
        .Returns(new Mock<IVmxServicesApiClient>().Object);

    // Create SDK with partial mocking
    var verjiSdkMock = new Mock<VerjiSdk>(
        new VerjiSdkV2(
            "test-variant",
            config,
            logger,
            mapper,
            new MxApplicationServiceConfig { Id = "APP", LocalPart = "vsys", AccessToken = "token" },
            adminApiIndexMock.Object,
            csApiClientIndexMock.Object,
            vmxServicesApiIndexMock.Object
        ));

    verjiSdkMock.SetupGet(m => m.V2).CallBase();
    var verjiSdk = verjiSdkMock.Object;

    // Act
    var tenantInfo = await verjiSdk.V2.GetTenantInfoAsync(roomId);

    // Assert
    Check.That(tenantInfo).IsNotNull();
    Check.That(tenantInfo.TenantId).IsEqualTo(tenantId);

    // Verify mocks were called
    adminApiMock.Verify(
        a => a.GetRoomState(roomId, It.IsAny<string>(), It.IsAny<string>(), null),
        Times.Once);

    csApiMock.Verify(
        c => c.ApplicationServiceLogin(
            It.IsNotNull<ApplicationServiceLoginRequest>(),
            It.IsNotNull<string>(),
            It.IsNotNull<string>(),
            null),
        Times.AtLeastOnce);
}
```

## Testing Utilities and Helpers

### TestExtensions

**Database initialization and session actions:**

```csharp
// Initialize test database
await TestExtensions.InitTestDbAsync(_config, "unittest");

// Ensure connection works
TestExtensions.EnsureConnectionPossible(_config, container);

// Run session action for tenant
await TestExtensions.RunSessionActionForTenantAsync(_config, container, session =>
{
    session.DeleteWhere<Organization>(org => true);
    session.DeleteWhere<Person>(p => p.TenantId == testTenantId);
});
```

### TestData Helpers

**Generate test data:**

```csharp
// Create multiple organizations
var organizations = await TestData.Organization.CreateManyOrganizations(
    count: 5,
    prefix: "test_org_",
    param1: 1,
    param2: 1,
    param3: 1);

// Room state test data
var spaceRoomState = TestData.AdminApi.GetRoomStateForSpaceRoom(tenantId);
var normalRoomState = TestData.AdminApi.GetRoomStateForNormalRoom(tenantId);
var mockRoomState = TestData.AdminApi.GetMockRoomStateWithSpaceParent(parentRoomId);
```

### Configuration Pattern

```csharp
private readonly IConfiguration _config;

public TestYourClass(ITestOutputHelper output)
{
    _config = new ConfigurationBuilder()
        .AddJsonFile("test-settings.json")
        .Build();

    // Use config
    var connectionString = _config["DatabaseConnectionString"];
    var apiUrl = _config["ApiUrl"];
}
```

### Method Name Extraction

```csharp
using System.Diagnostics;
using System.Runtime.CompilerServices;

[MethodImpl(MethodImplOptions.NoInlining)]
public static string GetMyMethodName()
{
    var st = new StackTrace();
    var sf = st.GetFrame(4);  // Adjust based on call depth
    return sf?.GetMethod()?.Name ?? "UnknownTest";
}

// Usage
var testContext = GetMyMethodName();  // Returns calling test method name
var builder = GetBuilderForTenant(testContext, _config);
```

## Best Practices Checklist

### Test Structure
- [ ] Test class named `Test{ClassUnderTest}`
- [ ] Constructor takes `ITestOutputHelper output`
- [ ] Serilog configured with Console, Debug, and TestOutput sinks
- [ ] Configuration loaded from `test-settings.json`
- [ ] Arrange-Act-Assert pattern with clear comments
- [ ] Async tests return `Task`

### Saga Testing
- [ ] Use `NServiceBus.Testing` framework
- [ ] Create saga with `TestableSaga<TSaga, TSagaData>` or direct instantiation
- [ ] Use `TestableMessageHandlerContext` for message verification
- [ ] Test complete workflows (multi-step sagas)
- [ ] Verify saga state changes
- [ ] Verify sent/published/timeout messages
- [ ] Use `MessageMapper` for interface-based events
- [ ] Test pause/resume patterns with timeouts

### Database Testing
- [ ] Use real Marten database (not in-memory fake)
- [ ] Initialize database with `TestExtensions.InitTestDbAsync`
- [ ] Use method name for test isolation (`GetMyMethodName()`)
- [ ] Autofac container configured with test modules
- [ ] Sessions created with `LightweightSession(tenantId)`
- [ ] Sessions use `await using` for proper disposal
- [ ] Call `SaveChangesAsync()` after mutations
- [ ] Cleanup via test isolation (preferred) or `DeleteWhere`

### Mocking
- [ ] Use Moq for mocking dependencies
- [ ] Setup returns with `Returns()` or `ReturnsAsync()`
- [ ] Use `It.IsAny<T>()` for flexible argument matching
- [ ] Use `It.IsNotNull<T>()` when null not allowed
- [ ] Mock IIndex<string, T> for Autofac keyed services
- [ ] Use `CallBase()` for partial mocking
- [ ] Verify mocks called with `Verify()`
- [ ] Use `RestResultWrapper` pattern for API responses

### Assertions
- [ ] Use NFluent for all new tests (`Check.That()`)
- [ ] Convert FluentAssertions to NFluent when modifying tests
- [ ] Use xUnit Assert for simple type/null checks
- [ ] Use `Assert.Multiple()` for grouped assertions
- [ ] Verify collections with `HasSize()`, `IsEmpty()`, `Contains()`
- [ ] Verify booleans with `IsTrue()`, `IsFalse()`
- [ ] Verify equality with `IsEqualTo()`, `Equals()`

### Theory Testing
- [ ] Use `[Theory]` for parameterized tests
- [ ] Use `[InlineData]` for simple parameter sets
- [ ] Use enums for complex scenario identification
- [ ] Test edge cases (null, empty, invalid)
- [ ] Switch on scenario enum in Arrange and Assert
- [ ] Descriptive parameter names

### Utilities
- [ ] Use `GetMyMethodName()` for test isolation
- [ ] Use `TestData` helpers for generating test data
- [ ] Use `TestExtensions` for database operations
- [ ] Create helper methods for common mock setups
- [ ] Use `GetAutoMapperStub()` for mapper configuration

## Common Mistakes to Avoid

1. **Using FluentAssertions in new tests:**
   ```csharp
   // WRONG - FluentAssertions (licensing concerns)
   result.Should().BeEquivalentTo(expected);

   // CORRECT - NFluent
   Check.That(result).IsEqualTo(expected);
   ```

2. **Not disposing sessions:**
   ```csharp
   // WRONG
   var session = docStore.LightweightSession(tenantId);
   // ... use session but never dispose

   // CORRECT
   await using (var session = docStore.LightweightSession(tenantId))
   {
       // ... use session
   }  // Automatically disposed
   ```

3. **Forgetting SaveChangesAsync:**
   ```csharp
   // WRONG
   repo.Store(session, aggregate);
   // Changes not committed!

   // CORRECT
   repo.Store(session, aggregate);
   await session.SaveChangesAsync();
   ```

4. **Not verifying saga state:**
   ```csharp
   // WRONG - Only verify messages, not state
   await saga.Handle(command, context);
   Check.That(context.SentMessages).HasSize(1);

   // CORRECT - Verify state AND messages
   await saga.Handle(command, context);
   Check.That(saga.Data.HasBeenProcessed).IsTrue();
   Check.That(saga.Data.CorrelationId).Equals(correlationId);
   Check.That(context.SentMessages).HasSize(1);
   ```

5. **Not using test isolation:**
   ```csharp
   // WRONG - All tests share same tenant, conflicts possible
   var builder = GetBuilderForTenant("shared-test-tenant");

   // CORRECT - Each test has unique tenant
   var testContext = GetMyMethodName();
   var builder = GetBuilderForTenant(testContext);
   ```

6. **Missing AAA comments:**
   ```csharp
   // WRONG - No structure
   var saga = new YourSaga(logger);
   await saga.Handle(command, context);
   Check.That(saga.Completed).IsTrue();

   // CORRECT - Clear AAA structure
   // Arrange
   var saga = new YourSaga(logger);
   var command = new YourCommand(correlationId);
   var context = new TestableMessageHandlerContext();

   // Act
   await saga.Handle(command, context);

   // Assert
   Check.That(saga.Completed).IsTrue();
   ```

7. **Not extracting sent messages properly:**
   ```csharp
   // WRONG - Type assertion without null check
   var sentCommand = context.SentMessages[0].Message as YourCommand;
   Check.That(sentCommand.CorrelationId).Equals(correlationId);  // NullReferenceException!

   // CORRECT - Verify before using
   Check.That(context.SentMessages).HasSize(1);
   var sentCommand = context.SentMessages[0].Message as YourCommand;
   Assert.NotNull(sentCommand);
   Check.That(sentCommand.CorrelationId).Equals(correlationId);
   ```

## Reference Implementations

Study these test classes for complete examples:

- **TestSyncSpaceProfileDataSagaFlow** (`Verji.ItOps.Matrix.Esb.Test/Spaces/`)
  - Multi-step saga flow testing
  - Complete workflow from dispatcher → saga → handlers → completion
  - Custom test event classes
  - Comprehensive message verification

- **TestEnsureMetadataForVerjiRoomSaga** (`Verji.ItOps.Matrix.Esb.Test/EnsureRoomVerjiMetadata/`)
  - Theory-based saga state testing
  - DoWorkStepFlow enum pattern
  - Pause/resume with timeouts
  - Complex scenario parameterization

- **TestOrganizationRepository** (`Rosberg.Common.Test/`)
  - Real database integration testing
  - Autofac container setup
  - Test isolation with method names
  - Access control testing

- **TestVerjiRoomMetadataSdk** (`Verji.ItOps.Matrix.Sdk.Test/VerjiSdk/`)
  - Advanced mocking with IIndex<string, T>
  - Partial mocking with CallBase
  - RestResultWrapper pattern
  - Multiple API client mocking
