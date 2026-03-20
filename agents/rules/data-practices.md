# Data Best Practices

Good data management reduces risk, lowers cost, and makes systems easier to reason about.
These practices apply across all services and storage technologies.

---

## Consistent Naming and Conventions

All database objects follow a single consistent naming convention across every service and
environment. Inconsistency compounds over time and makes cross-system queries, reporting,
and onboarding significantly harder.

- **Tables**: `snake_case`, plural (`customers`, `order_line_items`)
- **Columns**: `snake_case` (`created_at`, `customer_id`, `unit_price`)
- **Primary keys**: always named `id`, always a `uuid`
- **Foreign keys**: `{referenced_table_singular}_id` (`customer_id`, `order_id`)
- **Boolean columns**: prefix with `is_` or `has_` (`is_active`, `has_been_verified`)
- **Timestamp columns**: suffix with `_at` (`created_at`, `shipped_at`, `deleted_at`)
- **Indexes**: `ix_{table}_{columns}` (`ix_orders_customer_id`)
- **Unique constraints**: `uq_{table}_{columns}` (`uq_customers_email`)
- **Foreign key constraints**: `fk_{table}_{referenced_table}` (`fk_orders_customers`)

Deviations require explicit justification in the PR. Naming is documented and kept in sync
with the Mermaid schema diagram (see `database-schema` rules).

---

## Audit Columns

Tables that store business-significant records should carry audit columns tracking who
created and last modified the record and when. This supports debugging, compliance, and
incident investigation without requiring separate audit log queries.

### Standard audit columns

```sql
created_at   timestamptz  NOT NULL DEFAULT now()
created_by   uuid         NOT NULL  -- FK to users.id or service identity
updated_at   timestamptz  NOT NULL DEFAULT now()
updated_by   uuid         NOT NULL
```

### Implement with a shared base entity in EF Core

```csharp
// Shared infrastructure — apply to entities that need auditing
public abstract class AuditableEntity
{
    public DateTimeOffset CreatedAt  { get; private set; }
    public Guid           CreatedBy  { get; private set; }
    public DateTimeOffset UpdatedAt  { get; private set; }
    public Guid           UpdatedBy  { get; private set; }
}

// Automatically set via SaveChanges interceptor — never set manually in application code
public sealed class AuditInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUserService _currentUser;

    public AuditInterceptor(ICurrentUserService currentUser)
        => _currentUser = currentUser;

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct)
    {
        var now    = DateTimeOffset.UtcNow;
        var userId = _currentUser.UserId;

        foreach (var entry in eventData.Context!.ChangeTracker.Entries<AuditableEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Property(nameof(AuditableEntity.CreatedAt)).CurrentValue = now;
                entry.Property(nameof(AuditableEntity.CreatedBy)).CurrentValue = userId;
            }

            if (entry.State is EntityState.Added or EntityState.Modified)
            {
                entry.Property(nameof(AuditableEntity.UpdatedAt)).CurrentValue = now;
                entry.Property(nameof(AuditableEntity.UpdatedBy)).CurrentValue = userId;
            }
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken: ct);
    }
}
```

### When audit columns are not required

Not every table needs audit columns. Omit them when:

- The table is a pure join/association table with no independent lifecycle (e.g. `user_roles`)
- The data is derived or computed and has no meaningful "owner"
- The client has explicitly confirmed auditing is out of scope for this data type

When omitting, document the decision in the schema notes.

---

## Data Retention

Data must not be kept longer than it serves a purpose. Retention periods vary by client,
data type, and jurisdiction. Define and implement a retention policy before a system goes
to production — retrofitting it is significantly more expensive.

### Define retention per data type

Document retention requirements in `docs/data-retention.md` for each project:

| Data type | Retention period | Basis | Action on expiry |
|---|---|---|---|
| Customer PII | Duration of account + 2 years | Contract / GDPR | Anonymise or delete |
| Order records | 7 years | Tax / legal | Archive to cold storage |
| Application logs | 90 days | Operational | Delete |
| Audit logs | 5 years | Compliance | Archive |
| Session tokens | Until expiry | Security | Delete |
| Test / synthetic data | Duration of test run | Hygiene | Delete immediately |

### Implement retention as automated jobs, not manual processes

```csharp
// Background service that runs retention cleanup on a schedule
public sealed class DataRetentionWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromDays(1));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await _retentionService.DeleteExpiredSessionsAsync(stoppingToken);
            await _retentionService.AnonymiseExpiredCustomerDataAsync(stoppingToken);
            await _retentionService.ArchiveOldOrdersAsync(stoppingToken);
        }
    }
}
```

Retention jobs are idempotent — safe to run multiple times without unintended side effects.

---

## Minimise Data Movement and Access

Every time data moves, the risk of exposure, corruption, and latency increases. Design
systems to keep data close to where it is used and to access it as few times as necessary.

### Fetch only what you need

```csharp
// Good — project to the shape needed; do not fetch the full entity
var summaries = await _context.Orders
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderSummary(o.Id, o.Status, o.Total, o.PlacedAt))
    .ToListAsync(ct);

// Bad — fetches the full entity graph and discards most of it
var orders = await _context.Orders
    .Include(o => o.LineItems)
    .Include(o => o.Customer)
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(ct);
var summaries = orders.Select(o => new OrderSummary(...));
```

### Avoid passing data through intermediate services unnecessarily

If Service A needs data from the database that Service B already has, prefer giving
Service A direct (scoped) access over routing the data through Service B. Each hop is
a latency cost, a serialisation round-trip, and a potential exposure point.

### Use read replicas or read-optimised projections for reporting

Do not run analytical or reporting queries against the operational database. Use:

- A read replica for near-real-time reporting
- A dedicated reporting database or data warehouse for historical analysis
- Materialised views or pre-computed projections for complex aggregations

---

## Use Cloud Infrastructure for Data Storage

Prefer managed cloud data services over self-hosted infrastructure. Managed services
provide built-in redundancy, automated backups, transparent scaling, compliance
certifications, and lower operational overhead.

### Prefer managed services

| Need | Preferred approach |
|---|---|
| Relational data | Azure SQL / Azure Database for PostgreSQL / Amazon RDS |
| Document / flexible schema | Azure Cosmos DB / Amazon DynamoDB |
| Blob / file storage | Azure Blob Storage / Amazon S3 |
| Data lake (multi-format access) | Azure Data Lake Storage Gen2 / Amazon S3 + Athena |
| Message streaming | Azure Event Hubs / Amazon Kinesis |
| Cache | Azure Cache for Redis / Amazon ElastiCache |

Cloud-hosted data lakes expose structured, semi-structured, and unstructured data through
a standard interface (e.g. ADLS Gen2 + Synapse, or S3 + Athena), enabling analytics,
ML pipelines, and operational reporting without duplicating data into separate stores.

### Infrastructure as code for all data resources

All cloud data resources are provisioned and managed through infrastructure as code
(Bicep, Terraform, or Pulumi) — never created manually in the portal. This ensures
environments are reproducible and configuration is auditable.

---

## PII Classification and Protection

PII requirements vary by client and jurisdiction. What constitutes restricted PII for a
healthcare client differs from what it means for a retail client. Clarify this at the
start of every engagement.

### Determine PII scope with the client

Before any data model is finalised, establish with the client:

1. Which fields are considered PII in their context
2. Which PII is **restricted** (requires encryption, strict access control, or cannot be
   stored at all)
3. Which regulatory frameworks apply (GDPR, HIPAA, CCPA, etc.)
4. Residency requirements — where data can and cannot be stored geographically

Document the outcome in `docs/data-classification.md` and treat it as a living document
reviewed at each significant feature or schema change.

### Restricted PII must not appear in logs or query results

See the `security` rules for implementation detail on log redaction and sensitive data
handling. The principle here is: **if a field is classified as restricted PII, assume it
will appear somewhere it should not unless you have explicitly prevented it.**

---

## Test Data Hygiene

### Clean up test data after every test run

Tests that write to a database must clean up after themselves. Accumulating test data
degrades test reliability, distorts metrics, and can expose synthetic-but-realistic data
to unintended audiences.

```csharp
// Use EF Core transactions that are rolled back after each test
public sealed class DatabaseFixture : IAsyncLifetime
{
    private IDbContextTransaction _transaction = null!;

    public async Task InitializeAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task DisposeAsync()
    {
        await _transaction.RollbackAsync(); // test data never committed
        await _transaction.DisposeAsync();
    }
}
```

```typescript
// Vitest / Jest — wrap each test in a transaction
beforeEach(async () => { await db.query('BEGIN') });
afterEach(async ()  => { await db.query('ROLLBACK') });
```

### Never use production data as test data

Use generated synthetic data (factories, builders, fakers) for all test fixtures.
If production-like volume or variety is needed for performance tests, generate it
programmatically or use a scrubbed and anonymised snapshot (see `security` rules).

```csharp
// Use a builder pattern to generate realistic test data
public sealed class CustomerBuilder
{
    private string _email = $"{Guid.NewGuid()}@example.com";
    private string _name  = "Test Customer";

    public CustomerBuilder WithEmail(string email) { _email = email; return this; }
    public CustomerBuilder WithName(string name)   { _name  = name;  return this; }

    public Customer Build() => Customer.Create(_email, _name);
}
```

---

## Data Practices Checklist

For any PR that introduces or modifies data storage, retention, or access:

- [ ] Table and column names follow the project naming convention
- [ ] Audit columns (`created_at`, `created_by`, `updated_at`, `updated_by`) added where appropriate
- [ ] Retention period defined for any new data type and documented in `docs/data-retention.md`
- [ ] Queries fetch only the fields required — no unnecessary full-entity loads
- [ ] PII fields identified, classified, and confirmed with client scope
- [ ] Restricted PII is not logged, not returned in API responses where not needed
- [ ] Cloud managed service used in preference to self-hosted
- [ ] Infrastructure provisioned via IaC — no manual portal changes
- [ ] Test data cleaned up within the test — no persistent test records left behind
- [ ] No production data used in test or development environments without scrubbing
