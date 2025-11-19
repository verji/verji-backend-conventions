# Marten Repository Patterns and Conventions

This document describes Verji's patterns and conventions for implementing repositories using Marten document database.

## Overview

All Verji backend services use **Marten** as the document database with PostgreSQL. The repository pattern provides:

- **Abstraction** - Separates data access logic from business logic
- **Testability** - Repositories can be mocked for unit tests
- **Consistency** - Standardized CRUD operations and query patterns
- **Extensibility** - Base classes can be specialized for domain-specific needs

### Architecture

The dependency chain flows: **Connector → Repository → Domain**

- **Connectors** map DTOs to domain objects before passing to repositories, and map domain objects back to DTOs for API responses
- **Repositories** work exclusively with domain aggregates (not DTOs), handle session management, and execute queries
- **Domain** aggregates represent business entities with behavior and state

## Base Class Hierarchy

```
Aggregates:
IAggregate
    └── BaseAggregate (with ACL support)
            ├── OwnableAggregate (for aggregates with ownership concept)
            └── Your domain aggregates

Repositories:
IBaseMartenRepository
    └── BaseMartenRepository
            ├── Specialized base (optional, e.g. BaseLicenseRepository)
            └── Concrete repositories (YourDomainRepository)
```

**Key Interfaces:**

```csharp
public interface IAggregate
{
    string Id { get; set; }
    string[] Acl { get; }
    IAggregateMetadata Metadata { get; set; }
}

public interface IBaseMartenRepository
{
    Task<TAgg?> GetById<TAgg>(IDocumentSession session, string id) where TAgg : notnull;
    Task<TAgg[]> GetByIds<TAgg>(IDocumentSession session, string[] ids) where TAgg : notnull;
    void Store<TAgg>(IDocumentSession session, TAgg aggregate) where TAgg : BaseAggregate;
    void Delete(IDocumentSession session, string id);
    Task<IPagedList<TAgg>> FindByQuery<TAgg, TQuery>(
        IDocumentSession session, TQuery query, int pageNum = 1, int pageSize = 1000)
        where TAgg : BaseAggregate
        where TQuery : BaseQuery;
}
```

## Session Management Pattern

### Creating Sessions

Sessions are created per operation using the `LightweightSession` method:

```csharp
using (var session = DocStore.LightweightSession(dbTenant))
{
    // Perform operations
    await session.SaveChangesAsync(); // Commit if needed
}
```

**Key Points:**
- Sessions are **never** stored as fields - always create fresh per operation
- Use `using` statement to ensure proper disposal
- dbTenant parameter is currently static (e.g., `"*VMX-DEFAULT*"`) - reserved for future database-level multi-tenancy
- Multi-tenancy is currently implemented via access control rules (see Access Control Pattern section)

### Session Lifecycle

```csharp
// CORRECT: Session created, used, and disposed
public async Task<TDto> GetById<TAgg, TDto>(string id) where TAgg : notnull
{
    using (var session = DocStore.LightweightSession(dbTenant))
    {
        var aggregate = await Repo.GetById<TAgg>(session, id);
        var result = Mapper.Map<TDto>(aggregate);
        return result;
    } // Session disposed here
}

// WRONG: Never store session as field
private IDocumentSession _session; // ❌ Don't do this
```

### Passing Sessions to Repositories

Sessions are passed as **method parameters** to repository methods:

```csharp
public virtual async Task<TAgg?> GetById<TAgg>(IDocumentSession session, string id)
    where TAgg : notnull
{
    var result = await session.LoadAsync<TAgg>(id);
    return result;
}
```

This ensures:
- Repository doesn't manage session lifecycle
- Repository methods are stateless
- Clear ownership of session lifecycle (caller manages)

## Repository Base Class (BaseMartenRepository)

The `BaseMartenRepository` provides standard CRUD operations and query patterns.

**Location:** `Rosberg.Common.Base.BaseMartenRepository`

### CRUD Operations

```csharp
public abstract class BaseMartenRepository : IBaseMartenRepository
{
    // READ - Single entity
    public virtual async Task<TAgg?> GetById<TAgg>(IDocumentSession session, string id)
        where TAgg : notnull
    {
        var result = await session.LoadAsync<TAgg>(id);
        return result;
    }

    // READ - Multiple entities
    public virtual async Task<TAgg[]> GetByIds<TAgg>(IDocumentSession session, string[] ids)
        where TAgg : notnull
    {
        var result = await session.LoadManyAsync<TAgg>(ids);
        return result.ToArray();
    }

    // CREATE/UPDATE - Store entity
    public virtual void Store<TAgg>(IDocumentSession session, TAgg aggregate)
        where TAgg : BaseAggregate
    {
        session.Store(aggregate);
        // Note: Caller must call session.SaveChangesAsync()
    }

    // DELETE
    public void Delete(IDocumentSession session, string id)
    {
        session.Delete(id);
        // Note: Caller must call session.SaveChangesAsync()
    }
}
```

**Important:** `Store` and `Delete` stage changes; caller must call `session.SaveChangesAsync()` to commit.

### Query Pattern

```csharp
public virtual async Task<IPagedList<TAgg>> FindByQuery<TAgg, TQuery>(
    IDocumentSession session, TQuery query, int pageNum = 1, int pageSize = 1000)
    where TAgg : BaseAggregate
    where TQuery : BaseQuery
{
    // Start with base query
    IQueryable<TAgg> dbQuery = session.Query<TAgg>();

    // Apply filters via extensible method
    dbQuery = AddQueryTerms(dbQuery, query);

    // Execute with pagination
    var results = await dbQuery.ToPagedListAsync(pageNum, pageSize);

    return results;
}
```

### Extensible Query Builder

The `AddQueryTerms` method uses the **chain of responsibility pattern** for composable filters:

```csharp
public virtual IQueryable<TAgg> AddQueryTerms<TAgg>(IQueryable<TAgg> dbQuery, BaseQuery qry)
    where TAgg : BaseAggregate
{
    // Base implementation: filter by Id if provided
    if (!string.IsNullOrWhiteSpace(qry.Id))
        dbQuery = dbQuery.Where(agg => agg.Id == qry.Id);

    return dbQuery;
}
```

**Design Principles:**
- Method is `virtual` - subclasses can override
- Returns `IQueryable` - maintains composability
- Subclasses call `base.AddQueryTerms()` first, then add their filters

## Creating New Repositories

### Option 1: Direct Inheritance from BaseMartenRepository

For simple repositories with minimal custom logic:

**Step 1: Define Query Object**

```csharp
public class TenantsSummaryReportQuery : BaseQuery
{
    public string? TenantId { get; set; }
    public DateTime? CreatedAfter { get; set; }
}
```

**Step 2: Create Repository Interface**

```csharp
public interface ITenantsSummaryReportRepository
{
    Task<IPagedList<TAgg>> FindByQuery<TAgg>(
        IDocumentSession session,
        TenantsSummaryReportQuery query,
        int pageNum = 1,
        int pageSize = 1000)
        where TAgg : TenantsSummaryReport;
}
```

**Step 3: Implement Repository**

```csharp
public class TenantsSummaryReportRepository : BaseMartenRepository,
    ITenantsSummaryReportRepository
{
    public async Task<IPagedList<TAgg>> FindByQuery<TAgg>(
        IDocumentSession session,
        TenantsSummaryReportQuery query,
        int pageNum = 1,
        int pageSize = 1000)
        where TAgg : TenantsSummaryReport
    {
        // Start query
        IQueryable<TAgg> dbQuery = session.Query<TAgg>();

        // Apply filters
        dbQuery = AddQueryTerms(dbQuery, query);

        // Apply ordering
        dbQuery = dbQuery.OrderByDescending(c => c.Id);

        // Execute with pagination
        var results = await dbQuery.ToPagedListAsync(pageNum, pageSize);

        return results;
    }

    // Override AddQueryTerms to add domain-specific filters
    public IQueryable<TTenantSummary> AddQueryTerms<TTenantSummary>(
        IQueryable<TTenantSummary> dbQuery,
        TenantsSummaryReportQuery query)
        where TTenantSummary : TenantsSummaryReport
    {
        // Chain to base class first
        var qry = base.AddQueryTerms(dbQuery, query);

        // Add domain-specific filters
        if (!string.IsNullOrWhiteSpace(query.TenantId))
            qry = qry.Where(t => t.TenantId == query.TenantId);

        if (query.CreatedAfter.HasValue)
            qry = qry.Where(t => t.Metadata.Created >= query.CreatedAfter);

        return qry;
    }
}
```

### Option 2: Create Specialized Base Class

When multiple repositories share common domain logic or filters, create an intermediate specialized base class.

**Why Create Specialized Base Classes:**
- **Reusability** - Share common query filters across related repositories
- **Consistency** - Ensure related repositories handle queries the same way
- **Maintainability** - Change shared logic in one place
- **Type Safety** - Strongly-typed query objects for domain area

**Example: BaseLicenseRepository**

**Step 1: Define Base Query for Domain**

```csharp
public class BaseFeatureLicenseQuery : BaseQuery
{
    public bool? Enabled { get; set; }
    public string? TenantId { get; set; }
    public string? PersonId { get; set; }
    public string? EntitlementGroup { get; set; }
    public string? ModuleName { get; set; }
    // ... common licensing filters
}
```

**Step 2: Create Specialized Base Repository**

```csharp
public class BaseLicenseRepository : BaseMartenRepository
{
    // Override AddQueryTerms with licensing-specific filters
    public virtual IQueryable<TLic> AddQueryTerms<TLic>(
        IQueryable<TLic> dbQuery,
        BaseFeatureLicenseQuery query)
        where TLic : BaseLicense
    {
        // Chain to parent first
        dbQuery = base.AddQueryTerms(dbQuery, query);

        // Add licensing-specific filters
        if (!string.IsNullOrWhiteSpace(query.TenantId))
            dbQuery = dbQuery.Where(lm => lm.TenantId == query.TenantId);

        if (query.Enabled.HasValue)
            dbQuery = dbQuery.Where(l => l.Enabled == query.Enabled);

        if (!string.IsNullOrWhiteSpace(query.PersonId))
            dbQuery = dbQuery.Where(lm =>
                lm.UnfilteredMembers.Any(m => m.PersonId.Equals(query.PersonId)));

        if (!string.IsNullOrWhiteSpace(query.EntitlementGroup))
            dbQuery = dbQuery.Where(lm => lm.EntitlementGroup == query.EntitlementGroup);

        if (!string.IsNullOrWhiteSpace(query.ModuleName))
            dbQuery = dbQuery.Where(lm => lm.ModuleName == query.ModuleName);

        return dbQuery;
    }
}
```

**Step 3: Create Concrete Repository from Specialized Base**

```csharp
public class SigningLicenseQuery : BaseFeatureLicenseQuery
{
    // Add signing-specific query fields
    public int? EntitledUserCount { get; set; }
}

public class SigningLicenseRepository : BaseLicenseRepository, ISigningLicenseRepository
{
    public async Task<IPagedList<TLic>> FindByQuery<TLic>(
        IDocumentSession session,
        SigningLicenseQuery query,
        int pageNum = 1,
        int pageSize = 1000)
        where TLic : SigningLicense
    {
        IQueryable<TLic> dbQuery = session.Query<TLic>();

        // Apply filters (chains through BaseLicenseRepository → BaseMartenRepository)
        dbQuery = AddQueryTerms(dbQuery, query);

        // Apply ordering
        dbQuery = dbQuery.OrderByDescending(c => c.ModuleName);

        var results = await dbQuery.ToPagedListAsync(pageNum, pageSize);
        return results;
    }

    // Add signing-specific filters
    public IQueryable<TLicense> AddQueryTerms<TLicense>(
        IQueryable<TLicense> dbQuery,
        SigningLicenseQuery query)
        where TLicense : SigningLicense
    {
        // Chain to BaseLicenseRepository (which chains to BaseMartenRepository)
        var qry = base.AddQueryTerms(dbQuery, query);

        // Add signing-specific filters
        if (query.EntitledUserCount.HasValue)
        {
            qry = qry.Where(lm => lm.EntitledUserCount == query.EntitledUserCount);
        }

        return qry;
    }
}
```

**Filter Chain Flow:**

```
SigningLicenseRepository.AddQueryTerms
    ↓ calls base
BaseLicenseRepository.AddQueryTerms (adds licensing filters)
    ↓ calls base
BaseMartenRepository.AddQueryTerms (adds Id filter)
    ↓ returns
Composed IQueryable with all filters applied
```

### When to Use Specialized Base Classes

**Use specialized base class when:**
- Multiple repositories share common domain logic
- 3+ repositories need the same filters
- Domain area has consistent query patterns
- You want strongly-typed query objects

**Use direct inheritance when:**
- Repository is unique with no shared logic
- Simple CRUD with minimal querying
- Prototyping or early development

## Query Pattern

### Query Object Hierarchy

Query objects inherit to provide type-safe, reusable filter definitions:

```csharp
// Base query - available to all repositories
public class BaseQuery
{
    public string? Id { get; set; }
    public IAggregateMetadata? Metadata { get; set; }
}

// Specialized domain query
public class BaseFeatureLicenseQuery : BaseQuery
{
    public bool? Enabled { get; set; }
    public string? TenantId { get; set; }
    public string? PersonId { get; set; }
    // ... domain-specific fields
}

// Concrete implementation query
public class SigningLicenseQuery : BaseFeatureLicenseQuery
{
    public int? EntitledUserCount { get; set; }
    // ... signing-specific fields
}
```

### Composable Filters with AddQueryTerms

Each level adds its filters and chains to parent:

```csharp
// Level 1: BaseMartenRepository
public virtual IQueryable<TAgg> AddQueryTerms<TAgg>(IQueryable<TAgg> dbQuery, BaseQuery qry)
{
    if (!string.IsNullOrWhiteSpace(qry.Id))
        dbQuery = dbQuery.Where(agg => agg.Id == qry.Id);
    return dbQuery;
}

// Level 2: Specialized Base
public virtual IQueryable<TLic> AddQueryTerms<TLic>(
    IQueryable<TLic> dbQuery, BaseFeatureLicenseQuery query)
{
    dbQuery = base.AddQueryTerms(dbQuery, query); // Chain to parent

    if (query.Enabled.HasValue)
        dbQuery = dbQuery.Where(l => l.Enabled == query.Enabled);

    return dbQuery;
}

// Level 3: Concrete Repository
public IQueryable<TLicense> AddQueryTerms<TLicense>(
    IQueryable<TLicense> dbQuery, SigningLicenseQuery query)
{
    var qry = base.AddQueryTerms(dbQuery, query); // Chain to parent

    if (query.EntitledUserCount.HasValue)
        qry = qry.Where(lm => lm.EntitledUserCount == query.EntitledUserCount);

    return qry;
}
```

### Pagination

Queries commonly use `ToPagedListAsync` for pagination:

```csharp
var results = await dbQuery.ToPagedListAsync(pageNum, pageSize);

// IPagedList provides:
// - results.Items (current page items)
// - results.TotalItemCount
// - results.PageCount
// - results.PageNumber
// - results.PageSize
```

## Access Control Pattern

### Overview

Multi-tenancy and data isolation are implemented via **access control rules**, not database-level separation. Each aggregate has an ACL (Access Control List) that defines who can access it.

**Current Implementation:**
- dbTenant in Marten sessions is static (reserved for future database-level multi-tenancy)
- Multi-tenancy via ACL arrays and AcContext filtering
- Defense in depth: query-level filtering based on tenant membership

### OwnableAggregate and ACL

Domain aggregates typically inherit from `BaseAggregate`. Use `OwnableAggregate` (which inherits from `BaseAggregate`) when the aggregate has an ownership concept - where the owner gets additional privileges or visibility for the object:

```csharp
public abstract class OwnableAggregate : BaseAggregate, IOwnable
{
    protected OwnableAggregate(string id) : base(id)
    {
        OwnerId = string.Empty;
    }

    protected OwnableAggregate(string id, string ownerId) : base(id)
    {
        OwnerId = ownerId;
    }

    public string OwnerId { get; private set; }

    // Default ACL implementation (can be overridden)
    public override string[] Acl => new[] { Id, OwnerId };
}
```

**Concrete Implementation:**

```csharp
public class SigningOrder : OwnableAggregate
{
    public string TenantId { get; private set; }
    public string OwnerId { get; private set; }

    // Define who can access this signing order
    public override string[] Acl
    {
        get
        {
            return new[]
            {
                Id,          // The signing order ID itself
                TenantId,    // The tenant/customer ID
                OwnerId      // The owner/creator ID
            };
        }
    }
}
```

### AcContext (Access Control Context)

The `AcContext` contains user identity and role information for access control filtering.

**Note:** AcContext creation, population, and AccessControlTerms implementation details are encapsulated in RosbergCommon and will be covered in dedicated AccessControl documentation.

### AccessControlTerms Extension

The `AccessControlTerms` extension method (from RosbergCommon) filters queries based on the aggregate's ACL array and the user's access rights:

```csharp
// Apply access control filtering
query = query.AccessControlTerms(context);
```

This extension method filters the query to return only aggregates where the user has access based on the ACL matching rules defined in RosbergCommon.

### Repository Methods with Access Control

Repositories provide **two variants** of methods:

```csharp
public interface ISigningOrderRepository
{
    // Without access control (for internal/admin use)
    Task<SigningOrder?> GetById(
        IDocumentSession session,
        string signingOrderId);

    // With access control (for user-facing queries)
    Task<SigningOrder?> GetById(
        AcContext context,
        IDocumentSession session,
        string signingOrderId);

    Task<IPagedList<SigningOrder>> FindByQuery(
        IDocumentSession session,
        SigningOrderQuery query,
        int pageNum = 1,
        int pageSize = 1000);

    Task<IPagedList<SigningOrder>> FindByQuery(
        AcContext context,
        IDocumentSession session,
        SigningOrderQuery query,
        int pageNum = 1,
        int pageSize = 1000);
}
```

### Implementation Example

```csharp
public class MartenSigningOrderRepository : ISigningOrderRepository
{
    // With access control
    public async Task<SigningOrder?> GetById(
        AcContext context,
        IDocumentSession session,
        string signingOrderId)
    {
        IQueryable<SigningOrder> query = session.Query<SigningOrder>();

        // Apply access control filter
        query = query.AccessControlTerms(context);

        // Filter by ID
        query = query.Where(s => s.Id == signingOrderId);

        var result = await query.SingleOrDefaultAsync();
        return result;
    }

    // Without access control
    public async Task<SigningOrder?> GetById(
        IDocumentSession session,
        string signingOrderId)
    {
        var result = await session.LoadAsync<SigningOrder>(signingOrderId);
        return result;
    }

    // Query with optional access control
    public async Task<IPagedList<SigningOrder>> FindByQuery(
        AcContext? context,
        IDocumentSession session,
        SigningOrderQuery query,
        int pageNum = 1,
        int pageSize = 1000)
    {
        IQueryable<SigningOrder> signingOrderQuery = session.Query<SigningOrder>();

        // Apply access control if context provided
        if (context != null)
        {
            signingOrderQuery = signingOrderQuery.AccessControlTerms(context);
        }

        // Apply business filters
        signingOrderQuery = signingOrderQuery.AddQueryTerms(query);

        // Execute with pagination
        var results = await signingOrderQuery.ToPagedListAsync(pageNum, pageSize);
        return results;
    }
}
```

### Query Filter Order

Always apply filters in this order:

```csharp
IQueryable<TAggregate> query = session.Query<TAggregate>();

// 1. Access control (filters by tenant membership)
if (context != null)
{
    query = query.AccessControlTerms(context);
}

// 2. Business filters (query-specific criteria)
query = query.AddQueryTerms(businessQuery);

// 3. Ordering
query = query.OrderBy(x => x.SomeField);

// 4. Execute with pagination
var results = await query.ToPagedListAsync(pageNum, pageSize);
```

### When to Use Access Control

**Use access control (AcContext) when:**
- User-facing API queries
- Multi-tenant data access
- Queries from authenticated users
- Need to enforce tenant isolation

**Skip access control when:**
- Internal admin operations
- Background jobs with system-level access
- Cross-tenant reporting (with proper authorization)
- Data migration or maintenance tasks

## Complete Examples

### Example 1: Simple Repository (Direct Inheritance)

```csharp
// Domain Aggregate
public class TenantsSummaryReport : BaseAggregate
{
    public string TenantId { get; set; }
    public int TotalUsers { get; set; }
    public DateTime GeneratedAt { get; set; }
}

// Query Object
public class TenantsSummaryReportQuery : BaseQuery
{
    public string? TenantId { get; set; }
}

// Repository Interface
public interface ITenantsSummaryReportRepository
{
    Task<IPagedList<TAgg>> FindByQuery<TAgg>(
        IDocumentSession session,
        TenantsSummaryReportQuery query,
        int pageNum = 1,
        int pageSize = 1000)
        where TAgg : TenantsSummaryReport;
}

// Repository Implementation
public class TenantsSummaryReportRepository : BaseMartenRepository,
    ITenantsSummaryReportRepository
{
    public async Task<IPagedList<TAgg>> FindByQuery<TAgg>(
        IDocumentSession session,
        TenantsSummaryReportQuery query,
        int pageNum = 1,
        int pageSize = 1000)
        where TAgg : TenantsSummaryReport
    {
        IQueryable<TAgg> dbQuery = session.Query<TAgg>();

        dbQuery = AddQueryTerms(dbQuery, query);
        dbQuery = dbQuery.OrderByDescending(c => c.Id);

        var results = await dbQuery.ToPagedListAsync(pageNum, pageSize);
        return results;
    }

    public IQueryable<TTenantSummary> AddQueryTerms<TTenantSummary>(
        IQueryable<TTenantSummary> dbQuery,
        TenantsSummaryReportQuery query)
        where TTenantSummary : TenantsSummaryReport
    {
        var qry = base.AddQueryTerms(dbQuery, query);

        if (!string.IsNullOrWhiteSpace(query.TenantId))
            qry = qry.Where(t => t.TenantId == query.TenantId);

        return qry;
    }
}
```

### Example 2: Repository with Access Control

```csharp
// Domain Aggregate with ACL
public class SigningOrder : OwnableAggregate
{
    public string TenantId { get; private set; }
    public string OwnerId { get; private set; }
    public string RoomId { get; private set; }
    public SigningOrderStatus Status { get; private set; }

    public override string[] Acl
    {
        get
        {
            return new[] { Id, TenantId, OwnerId };
        }
    }
}

// Query Object
public class SigningOrderQuery : BaseQuery
{
    public string? SigningOrderId { get; set; }
    public string? RoomId { get; set; }
    public string? OwnerId { get; set; }
    public string? TenantId { get; set; }
    public string[]? TenantIds { get; set; }
}

// Repository Interface
public interface ISigningOrderRepository
{
    Task<SigningOrder?> GetById(
        AcContext context,
        IDocumentSession session,
        string signingOrderId);

    Task<IPagedList<SigningOrder>> FindByQuery(
        AcContext? context,
        IDocumentSession session,
        SigningOrderQuery query,
        int pageNum = 1,
        int pageSize = 1000);
}

// Repository Implementation
public class MartenSigningOrderRepository : ISigningOrderRepository
{
    public async Task<SigningOrder?> GetById(
        AcContext context,
        IDocumentSession session,
        string signingOrderId)
    {
        IQueryable<SigningOrder> signingOrderQuery = session.Query<SigningOrder>();

        // Apply access control
        signingOrderQuery = signingOrderQuery.AccessControlTerms(context);

        signingOrderQuery = signingOrderQuery.Where(s => s.Id == signingOrderId);

        var result = await signingOrderQuery.SingleOrDefaultAsync();
        return result;
    }

    public async Task<IPagedList<SigningOrder>> FindByQuery(
        AcContext? context,
        IDocumentSession session,
        SigningOrderQuery query,
        int pageNum = 1,
        int pageSize = 1000)
    {
        IQueryable<SigningOrder> signingOrderQuery = session.Query<SigningOrder>();

        if (context != null)
        {
            signingOrderQuery = signingOrderQuery.AccessControlTerms(context);
        }

        signingOrderQuery = signingOrderQuery.AddQueryTerms(query);

        var results = await signingOrderQuery.ToPagedListAsync(pageNum, pageSize);
        return results;
    }
}

// Extension Methods for Query Filters
public static class MartenSigningOrderRepositoryExtensions
{
    public static IQueryable<SigningOrder> AddQueryTerms(
        this IQueryable<SigningOrder> query,
        SigningOrderQuery signingOrderQuery)
    {
        if (!string.IsNullOrWhiteSpace(signingOrderQuery.SigningOrderId))
            query = query.Where(s => s.Id == signingOrderQuery.SigningOrderId);

        if (!string.IsNullOrWhiteSpace(signingOrderQuery.RoomId))
            query = query.Where(s => s.RoomId == signingOrderQuery.RoomId);

        if (!string.IsNullOrWhiteSpace(signingOrderQuery.OwnerId))
            query = query.Where(s => s.OwnerId == signingOrderQuery.OwnerId);

        if (!string.IsNullOrWhiteSpace(signingOrderQuery.TenantId))
            query = query.Where(s => s.TenantId == signingOrderQuery.TenantId);

        if (signingOrderQuery.TenantIds != null && signingOrderQuery.TenantIds.Any())
            query = query.Where(s => s.TenantId.IsOneOf(signingOrderQuery.TenantIds));

        return query;
    }
}
```

### Example 3: Specialized Base Class Pattern

```csharp
// Specialized Query Base
public class BaseFeatureLicenseQuery : BaseQuery
{
    public bool? Enabled { get; set; }
    public string? TenantId { get; set; }
    public string? PersonId { get; set; }
    public string? EntitlementGroup { get; set; }
}

// Specialized Repository Base
public class BaseLicenseRepository : BaseMartenRepository
{
    public virtual IQueryable<TLic> AddQueryTerms<TLic>(
        IQueryable<TLic> dbQuery,
        BaseFeatureLicenseQuery query)
        where TLic : BaseLicense
    {
        dbQuery = base.AddQueryTerms(dbQuery, query);

        if (!string.IsNullOrWhiteSpace(query.TenantId))
            dbQuery = dbQuery.Where(lm => lm.TenantId == query.TenantId);

        if (query.Enabled.HasValue)
            dbQuery = dbQuery.Where(l => l.Enabled == query.Enabled);

        if (!string.IsNullOrWhiteSpace(query.PersonId))
            dbQuery = dbQuery.Where(lm =>
                lm.UnfilteredMembers.Any(m => m.PersonId.Equals(query.PersonId)));

        if (!string.IsNullOrWhiteSpace(query.EntitlementGroup))
            dbQuery = dbQuery.Where(lm => lm.EntitlementGroup == query.EntitlementGroup);

        return dbQuery;
    }
}

// Concrete Query
public class SigningLicenseQuery : BaseFeatureLicenseQuery
{
    public int? EntitledUserCount { get; set; }
}

// Concrete Repository
public class SigningLicenseRepository : BaseLicenseRepository,
    ISigningLicenseRepository
{
    public async Task<IPagedList<TLic>> FindByQuery<TLic>(
        IDocumentSession session,
        SigningLicenseQuery query,
        int pageNum = 1,
        int pageSize = 1000)
        where TLic : SigningLicense
    {
        IQueryable<TLic> dbQuery = session.Query<TLic>();

        dbQuery = AddQueryTerms(dbQuery, query);
        dbQuery = dbQuery.OrderByDescending(c => c.ModuleName);

        var results = await dbQuery.ToPagedListAsync(pageNum, pageSize);
        return results;
    }

    public IQueryable<TLicense> AddQueryTerms<TLicense>(
        IQueryable<TLicense> dbQuery,
        SigningLicenseQuery query)
        where TLicense : SigningLicense
    {
        var qry = base.AddQueryTerms(dbQuery, query);

        if (query.EntitledUserCount.HasValue)
        {
            qry = qry.Where(lm => lm.EntitledUserCount == query.EntitledUserCount);
        }

        return qry;
    }
}
```

## Best Practices Checklist

### Session Management
- [ ] Sessions created with `using` statement for automatic disposal
- [ ] Sessions never stored as fields - always local variables
- [ ] Sessions passed as method parameters to repositories
- [ ] Caller calls `SaveChangesAsync()` after `Store()` or `Delete()` operations
- [ ] dbTenant parameter recognized as static (not used for data isolation)

### Repository Design
- [ ] Repository inherits from `BaseMartenRepository` (or specialized base)
- [ ] All methods are `virtual` for extensibility
- [ ] Query methods return `IPagedList<T>` for pagination
- [ ] `AddQueryTerms` chains to `base.AddQueryTerms()` first
- [ ] Repository works with domain aggregates, not DTOs

### Query Patterns
- [ ] Query objects inherit from `BaseQuery` (or specialized query base)
- [ ] `AddQueryTerms` checks for null/empty before filtering
- [ ] Filters composed via chain of responsibility pattern
- [ ] Ordering applied before pagination
- [ ] `ToPagedListAsync` used for all paginated queries

### Access Control
- [ ] Domain aggregates inherit from `BaseAggregate` (all aggregates have ACL support)
- [ ] Domain aggregates inherit from `OwnableAggregate` when ownership concept needed
- [ ] `Acl` property returns relevant IDs (tenant, owner, etc.)
- [ ] Repository provides overloads with/without `AcContext`
- [ ] `AccessControlTerms` applied before business filters
- [ ] User-facing queries always use `AcContext`
- [ ] Internal/admin operations can skip access control

### Specialized Base Classes
- [ ] Created when 3+ repositories share common logic
- [ ] Specialized query inherits from `BaseQuery` (or parent query)
- [ ] Specialized repository inherits from `BaseMartenRepository`
- [ ] Methods are `virtual` for further extension
- [ ] Clear naming convention (e.g., `BaseLicenseRepository`)

## Common Mistakes to Avoid

1. **Storing sessions as fields:**
   ```csharp
   // WRONG
   private IDocumentSession _session;

   // CORRECT
   using (var session = DocStore.LightweightSession(TenantId))
   {
       // Use session
   }
   ```

2. **Forgetting to call SaveChangesAsync:**
   ```csharp
   // WRONG
   Repo.Store(session, aggregate);
   // Changes not committed!

   // CORRECT
   Repo.Store(session, aggregate);
   await session.SaveChangesAsync();
   ```

3. **Not chaining to base.AddQueryTerms:**
   ```csharp
   // WRONG - Loses base filters
   public IQueryable<T> AddQueryTerms<T>(IQueryable<T> query, MyQuery q)
   {
       return query.Where(x => x.Foo == q.Foo);
   }

   // CORRECT - Chains to parent
   public IQueryable<T> AddQueryTerms<T>(IQueryable<T> query, MyQuery q)
   {
       var result = base.AddQueryTerms(query, q);
       return result.Where(x => x.Foo == q.Foo);
   }
   ```

4. **Applying business filters before access control:**
   ```csharp
   // WRONG - Business filters first
   query = query.AddQueryTerms(businessQuery);
   query = query.AccessControlTerms(context);

   // CORRECT - Access control first
   query = query.AccessControlTerms(context);
   query = query.AddQueryTerms(businessQuery);
   ```

5. **Using TenantId for data isolation:**
   ```csharp
   // WRONG - TenantId is static, not for isolation
   session.Query<Order>().Where(o => o.TenantId == userTenantId);

   // CORRECT - Use AccessControlTerms
   session.Query<Order>()
       .AccessControlTerms(context)
       .Where(o => /* business filters */);
   ```

## Reference Implementations

Study these repositories for complete examples:

- **BaseMartenRepository** (`Rosberg.Common/Base/BaseMartenRepository.cs`)
  - Foundation for all repositories
  - CRUD operations
  - Query pattern with AddQueryTerms

- **BaseLicenseRepository** (`Rosberg.Common/Licensing/BaseLicenseRepository.cs`)
  - Example of specialized base class
  - Licensing-specific filters
  - Member and entitlement queries

- **TenantsSummaryReportRepository** (`Verji.ItOps.Api.Service/DomainAccess/TenantSummaryReport/`)
  - Simple direct inheritance example
  - Basic query pattern

- **SigningLicenseRepository** (`Verji.Signing.Api.Service/DomainAccess/Licensing/`)
  - Concrete implementation from specialized base
  - Query chain through multiple inheritance levels

- **MartenSigningOrderRepository** (`Verji.Signing.Api.Service/DomainAccess/`)
  - Access control implementation
  - Overloaded methods with/without AcContext
  - Extension methods for query filters
