---
title: Breaking changes in EF Core 3.0 - EF Core
author: divega
ms.date: 02/19/2019
ms.assetid: EE2878C9-71F9-4FA5-9BC4-60517C7C9830
uid: core/what-is-new/ef-core-3.0/breaking-changes
---

# Breaking changes included in EF Core 3.0 (currently in preview)

> [!IMPORTANT]
> Please note that the feature sets and schedules of future releases are always subject to change, and although we will try to keep this page up to date, it may not reflect our latest plans at all times.

The following API and behavior changes have the potential to break applications developed for EF Core 2.2.x when upgrading them to 3.0.0.
Changes that we expect to only impact database providers are documented under [provider changes](../../providers/provider-log.md).
Breaks in new features introduced from one 3.0 preview to another 3.0 preview aren't documented here.

## LINQ queries aren't evaluated on the client

[Tracking Issue #12795](https://github.com/aspnet/EntityFrameworkCore/issues/12795)

> [!IMPORTANT]
> We are pre-announcing this break. It has not yet shipped in any 3.0 preview.

**Old behavior**

Prior to EF Core 3.0, any part of a query that could not be either captured as a parameter or translated to SQL, would instead be evaluated on the client.
Potentially expensive client evaluation of expressions only triggered a warning by default.

**New behavior**

Starting with EF Core 3.0, we only allow the top-level projection (the expression passed to the last `Select()` LINQ operator in the query) to be evaluated on the client.
If expressions in any other part of the query cannot be either captured as a parameter or translated to SQL, then an exception is thrown.

**Why**

The automatic client-evaluation of queries allows many queries to be executed even if important parts of them cannot not be translated. This can result in unexpected and potentially damaging behavior that may only become apparent in production. For example, a query containing an expression in the `Where()` LINQ operator that cannot be translated, can result in all rows from the table to be transferred from the database server, and the filter to be applied on the client. This can easily go undetected if the table contains only a few rows in development and staging, but hit hard when you move the application to production where the table contains millions of rows. Client evaluation warnings also proved too easy to ignore during development.

Besides this, automatic client-evaluation can lead to issues in which improving query translation for specific expressions caused breaking changes between releases.

**Mitigations**

If a query cannot be fully translated, then either rewrite the query in a form that can be translated or use `AsEnumerable()`, `ToList()`, or similar to explicitly bring data back to the client where it can then be further processed using LINQ-to-Objects.


## Entity Framework Core is not part of the ASP.NET Core shared framework

This change was introduced in ASP.NET Core 3.0 preview 1. 

**Old behavior**

Prior to ASP.NET Core 3.0, when you added a package reference to `Microsoft.AspNetCore.App` or `Microsoft.AspNetCore.All`, it would include EF Core and some of the EF Core data providers like the SQL Server provider.

**New behavior**

Starting in 3.0, the ASP.NET Core shared framework does not include EF Core or any EF Core providers. In order to use EF Core in an ASP.NET Core 3.0 application, you need to explicitly add package references to the EF Core database providers that your application will use.

**Why**
The removal from the ASP.NET Core shared framework enables a NuGet-based acquisition and servicing story for EF Core that works uniformly across all EF Core providers and on all the supported .NET implementations and application types. 

Prior to this change, if your application targetd ASP.NET Core and used SQL Server, acquiring EF Core required one less step. But if the application doesn't target ASP.NET Core or uses a different provider, then you needed to learn a different set of steps. Also, performing a minor version upgrade of ASP.NET Core on a machine or targeted by an application, would force the upgrade of the EF Core and SQL Server povider version, which isn't recommended without proper testing. 

Note, Entity Framework Core moving out of the shared framework has no impact on its status as a Microsoft developed, supported, and serviceable library, and it will continue to be covered by the [.NET Core support policy.](https://www.microsoft.com/net/platform/support-policy)


## Query execution is logged at Debug level

[Tracking Issue #14523](https://github.com/aspnet/EntityFrameworkCore/issues/14523)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, execution of queries and other commands was logged at the `Info` level.

**New behavior**

Starting with EF Core 3.0, logging of command/SQL execution is at the `Debug` level.

**Why**

This change was made to reduce the noise at the `Info` log level.

**Mitigations**

This logging event is defined by `RelationalEventId.CommandExecuting` with event ID 20100. To log SQL at the `Info` level again, switch on logging at the `Debug` level and filter to just this event.


## Temporary key values are not set onto entity instances

[Tracking Issue #12378](https://github.com/aspnet/EntityFrameworkCore/issues/12378)

This change was introduced in EF Core 3.0-preview 2.

**Old behavior**

Prior to EF Core 3.0, temporary values were assigned to all key properties that would later have a real value generated by the database. These values were usually large negative numbers.

**New behavior**

Starting with EF Core 3.0, the temporary key value is now stored with EF's tracking information while the value of the key property itself will not be changed.

**Why**

This change was made to allow entities that have been previously tracked by some context instance to be moved to another instance without the temporary key values erroneously becoming permanent.

**Mitigations**

Applications may be using the temporary key values to form associations between entities. For example, the temporary primary key value may have been used to set an FK value. This can be avoided by:
* Not using store-generated keys.
* Not using primary key/foreign key values to associate entities, but instead use navigation properties. This is a best practice anyway since it uses only the object model without dependency on the underlying keys.
* Obtain the temporary values from EF's tracking information. For example, `context.Entry(blog).Property(e => e.Id).CurrentValue` will return the temporary value even though `blog.Id` itself has not been set.

## DetectChanges honors store-generated key values

[Tracking Issue #14616](https://github.com/aspnet/EntityFrameworkCore/issues/14616)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, an untracked entity found by `DetectChanges` would be tracked in the `Added` state. This means that it would be inserted as a new row when `SaveChanges` is called.

**New behavior**

Starting with EF Core 3.0, if an entity is using generated key values and some key value is set, then the entity will be tracked in the `Modified` state. This means that a row for the entity is assumed to exist and it will be updated when `SaveChanges` is called. If the key value is not set, or if the entity type is not using generated keys, then the new entity will still be tacked as `Added` just as in previous versions.

**Why**

This change was made to make it easier and more consistent to work with disconnected entity graphs while using store-generated keys.

**Mitigations**

This can break an application if an entity type is configured to use generated keys but then explicit key values are being set for new instances. The fix is to explicitly configure the key properties to not use generated values.  For example, with the fluent API:

```C#
modelBuilder
    .Entity<Blog>()
    .Property(e => e.Id)
    .ValueGeneratedNever();
```

Or with data annotations:

```C#
[DatabaseGenerated(DatabaseGeneratedOption.None)]
public string Id { get; set; }
```

## Cascade deletions now happen immediately by default

[Tracking Issue #10114](https://github.com/aspnet/EntityFrameworkCore/issues/10114)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, cascading actions (deleting dependent entities when a required principal is deleted or when the relationship to a required principal is severed) did not happen until SaveChanges was called.

**New behavior**

Starting with EF Core 3.0, cascading actions happen immediately that the triggering condition is detected by EF. For example, calling `context.Remove()` to delete a principal entity will result in all tracked related required dependents also being set to `Deleted` immediately.

**Why**

This change was made to improve the experience for data binding and auditing scenarios where it is important to understand which entities will be deleted _before_ `SaveChanges` is called.

**Mitigations**

The previous behavior can be restored through settings on `context.ChangedTracker`. For example:

```C#
context.ChangeTracker.CascadeDeleteTiming = CascadeTiming.OnSaveChanges;
context.ChangeTracker.DeleteOrphansTiming = CascadeTiming.OnSaveChanges;
```

## Query types are consolidated with entity types

[Tracking Issue #14194](https://github.com/aspnet/EntityFrameworkCore/issues/14194)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, [Query types](xref:core/modeling/query-types) were a means to query data that doesn't contain a primary key in a structured way. That is, a query type was used for mapping entity types without keys (more likely from a view, but possibly from a table) while a regular entity type was used when a key was available (more likely from a table, but possibly from a view.)

**New behavior**

A query type now becomes just an entity type without a primary key. Keyless entity types have the same functionality as query types in previous versions.

**Why**

This change was made to reduce the confusion around the purpose of query types. Specifically, they are keyless entity types. They are inherently read-only because of this, but should not be used just because an entity type is read-only. Likewise, they are often mapped to views, but this is only because views often don't define keys.

**Mitigations**

The following parts of the API are now obsolete:
* **`ModelBuilder.Query<>()`** - Instead `ModelBuilder.Entity<>().HasNoKey()` needs to be called to mark an entity type as having no keys. This would still not be configured by convention to avoid misconfiguration when a primary key is expected, but doesn't match the convention.
* **`DbQuery<>`** - Instead `DbSet<>` should be used.
* **`DbContext.Query<>()`** - Instead `DbContext.Set<>()` should be used.

## Configuration API for owned type relationships has changed

[Tracking Issue #12444](https://github.com/aspnet/EntityFrameworkCore/issues/12444)
[Tracking Issue #9148](https://github.com/aspnet/EntityFrameworkCore/issues/9148)
[Tracking Issue #14153](https://github.com/aspnet/EntityFrameworkCore/issues/14153)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, configuration of the owned relationship was performed directly after the `OwnsOne` or `OwnsMany` call. 

**New behavior**

Starting with EF Core 3.0, there is now fluent API to configure a navigation to the owner using `WithOwner()`. For example:

```C#
modelBuilder.Entity<Order>.OwnsOne(e => e.Details).WithOwner(e => e.Order);
```

The configuration related to the relationship between owner and owned should now be chained after `WithOwner()` similarly to how other relationships are configured. While the configuration for the owned type itself would still be chained after `OwnsOne()/OwnsMany()`. For example:

```C#
modelBuilder.Entity<Order>.OwnsOne(e => e.Details, eb =>
    {
        eb.WithOwner()
            .HasForeignKey(e => e.AlternateId)
            .HasConstraintName("FK_OrderDetails");
            
        eb.ToTable("OrderDetails");
        eb.HasKey(e => e.AlternateId);
        eb.HasIndex(e => e.Id);

        eb.HasOne(e => e.Customer).WithOne();

        eb.HasData(
            new OrderDetails
            {
                AlternateId = 1,
                Id = -1
            });
    });
```

Additionally calling `Entity()`, `HasOne()` or `Set()` with an owned type target will now throw an exception.

**Why**

This change was made to make a cleaner separation between configuring the owned type itself and the _relationship to_ the owned type. This in turn removes ambiguity and confusion around methods like `HasForeignKey`.

**Mitigations**

Change configuration of owned type relationships to use the new API surface as shown in the example above.

## The foreign key property convention no longer matches same name as the principal property

[Tracking Issue #13274](https://github.com/aspnet/EntityFrameworkCore/issues/13274)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Consider the following model:
```C#
public class Customer
{
    public int CustomerId { get; set; }
    public ICollection<Order> Orders { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
}

```
Prior to EF Core 3.0, the `CustomerId` property would be used for the foreign key by convention. However, if `Order` is an owned type, then this would also make `CustomerId` the primary key and this is usually not the expectation.

**New behavior**

Starting with EF Core 3.0, EF will not try to use properties for foreign keys by convention if they have the same name as the principal property. Principal type name concatenated with principal property name and navigation name concatenated with principal property name patterns will still be matched. For example:

```C#
public class Customer
{
    public int Id { get; set; }
    public ICollection<Order> Orders { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
}
```

```C#
public class Customer
{
    public int Id { get; set; }
    public ICollection<Order> Orders { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int BuyerId { get; set; }
    public Customer Buyer { get; set; }
}
```

**Why**

This change was made to avoid erroneously defining a primary key property on the owned type.

**Mitigations**

If the property was intended to be the foreign key, and hence part of the primary key, then explicitly configure it as such.

## Each property uses independent in-memory integer key generation

[Tracking Issue #6872](https://github.com/aspnet/EntityFrameworkCore/issues/6872)

This change will be introduced in EF Core 3.0-preview 4.

**Old behavior**

Prior to EF Core 3.0, one shared value generator was used for all in-memory integer key properties.

**New behavior**

Starting with EF Core 3.0, each integer key property gets its own value generator when using the in-memory database. Also, if the database is deleted, then key generation is reset for all tables.

**Why**

This change was made to align in-memory key generation more closely to real database key generation and to improve the ability to isolate tests from each other when using the in-memory database.

**Mitigations**

This can break an application that is relying on specific in-memory key values to be set. Consider instead not relying on specific key values, or updating to match the new behavior.

## Backing fields are used by default

[Tracking Issue #12430](https://github.com/aspnet/EntityFrameworkCore/issues/12430)

This change was introduced in EF Core 3.0-preview 2.

**Old behavior**

Prior to EF Core 3.0, even if the backing field for a property was known, EF would still by default read and write the property value using the property getter and setter methods. The exception to this was query execution, where the backing field would be set directly if known.

**New behavior**

Starting with EF Core 3.0, if the backing field for a property is known, then will always read and write that property using the backing field. This could cause an application break if the application is relying on additional behavior coded into the getter or setter methods.

**Why**

This change was made to prevent EF from erroneously triggering business logic by default when performing database operations involving the entities.

**Mitigations**

The pre-3.0 behavior can be restored through configuration of the property access mode in the modelBuilder fluent API. For example:

```C#
modelBuilder.UsePropertyAccessMode(PropertyAccessMode.PreferFieldDuringConstruction);
```

## Throw if multiple compatible backing fields are found

[Tracking Issue #12523](https://github.com/aspnet/EntityFrameworkCore/issues/12523)

This change will be introduced in EF Core 3.0-preview 4.

**Old behavior**

Prior to EF Core 3.0, if multiple fields matched the rules for finding the backing field of a property, then one field would be chosen based on some precedence order. This could cause the wrong field to be used in ambiguous cases.

**New behavior**

Starting with EF Core 3.0, if multiple fields are matched to the same property, then an exception is thrown.

**Why**

This change was made to avoid silently using one field over another when only one can be correct.

**Mitigations**

Properties with ambiguous backing fields must have the field to use specified explicitly. For example, using the fluent API:

```C#
modelBuilder
    .Entity<Blog>()
    .Property(e => e.Id)
    .HasField("_id");
```

## DbContext.Entry now performs a local DetectChanges

[Tracking Issue #13552](https://github.com/aspnet/EntityFrameworkCore/issues/13552)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, calling `DbContext.Entry` would cause changes to be detected for all tracked entities. This ensured that the state exposed in the `EntityEntry` was up-to-date.

**New behavior**

Starting with EF Core 3.0, calling `DbContext.Entry` will now only attempt to detect changes in the given entity and any tracked principal entities related to it. This means that changes elsewhere may not have been detected by calling this method, which could have implications on application state.

Note that if `ChangeTracker.AutoDetectChangesEnabled` is set to `false` then even this local change detection will be disabled.

Other methods that cause change detection--for example `ChangeTracker.Entries` and `SaveChanges`--still cause a full `DetectChanges` of all tracked entities.

**Why**

This change was made to improve the default performance of using `context.Entry`.

**Mitigations**

Call `ChgangeTracker.DetectChanges()` explicitly before calling `Entry` to ensure the pre-3.0 behavior.

## String and byte array keys are not client-generated by default

[Tracking Issue #14617](https://github.com/aspnet/EntityFrameworkCore/issues/14617)

This change will be introduced in EF Core 3.0-preview 4.

**Old behavior**

Prior to EF Core 3.0, `string` and `byte[]` key properties could be used without explicitly setting a non-null value. In such a case, the key value would be generated on the client as a GUID, serialized to bytes for `byte[]`.

**New behavior**

Starting with EF Core 3.0 an exception will be thrown indicating that no key value has been set.

**Why**

This change was made because client-generated `string`/`byte[]` values are generally not useful and this inappropriate default configuration was causing issues with reasoning about generated key values in a common way.

**Mitigations**

The pre-3.0 behavior can be obtained by explicitly specifying that the key properties should use generated values if no other non-null value is set. For example, with the fluent API:

```C#
modelBuilder
    .Entity<Blog>()
    .Property(e => e.Id)
    .ValueGeneratedOnAdd();
```

Or with data annotations:

```C#
[DatabaseGenerated(DatabaseGeneratedOption.Identity)]
public string Id { get; set; }
```

## ILoggerFactory is now a scoped service

[Tracking Issue #14698](https://github.com/aspnet/EntityFrameworkCore/issues/14698)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, `ILoggerFactory` was registered as a singleton service.

**New behavior**

Starting with EF Core 3.0, `ILoggerFactory` is now registered as scoped.

**Why**

This change was made to allow association of a logger with a `DbContext` instance, which enables other functionality and removes some cases of pathological behavior such as an explosion of internal service providers.

**Mitigations**

This change should not impact application code unless it is registering and using custom services on the EF internal service provider. This is not common. In these cases, most things will still work, but any singleton service that was depending on `ILoggerFactory` will need to be changed to obtain the `ILoggerFactory` in a different way.

If you run into situations like this, please file an issue at on the [EF Core GitHub issue tracker](https://github.com/aspnet/EntityFrameworkCore/issues) to let us know how you are using `ILoggerFactory` such that we can better understand how not to break this again in the future.

## IDbContextOptionsExtensionWithDebugInfo merged into IDbContextOptionsExtension

[Tracking Issue #13552](https://github.com/aspnet/EntityFrameworkCore/issues/13552)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

`IDbContextOptionsExtensionWithDebugInfo` was an additional optional interface extended from `IDbContextOptionsExtension` to avoid making a breaking change to the interface during the 2.x release cycle.

**New behavior**

The interfaces are now merged together into `IDbContextOptionsExtension`.

**Why**

This change was made because the interfaces are conceptually one.

**Mitigations**

Any implementations of `IDbContextOptionsExtension` will need to be updated to support the new member.

## Lazy-loading proxies no longer assume navigation properties are fully loaded

[Tracking Issue #12780](https://github.com/aspnet/EntityFrameworkCore/issues/12780)

This change will be introduced in EF Core 3.0-preview 4.

**Old behavior**

Prior to EF Core 3.0, once a `DbContext` was disposed there was no way of knowing if a given navigation property on an entity obtained from that context was fully loaded or not. Proxies would instead assume that a reference navigation is loaded if it has a non-null value, and that a collection navigation is loaded if it is not empty. In these cases, attempting to lazy-load would be a no-op.

**New behavior**

Starting with EF Core 3.0, proxies keep track of whether or not a navigation is loaded. This means attempting to access a navigation property that is loaded after the context has been disposed will always be a no-op, even when the loaded navigation is empty or null. Conversely, attempting to access a navigation property that is not loaded will throw an exception if the context is disposed even if the navigation property is a non-empty collection. If this situation arises, it means the application code is attempting to use lazy-loading at an invalid time and the application should be changed to not do this.

**Why**

This change was made to make the behavior consistent and correct when attempting to lazy-load on a disposed `DbContext` instance.

**Mitigations**

Update application code to not attempt lazy-loading with a disposed context, or configure this to be a no-op as described in the exception message.

## Excessive creation of internal service providers is now an error by default

[Tracking Issue #10236](https://github.com/aspnet/EntityFrameworkCore/issues/10236)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, a warning would be logged for an application creating a pathological number of internal service providers.

**New behavior**

Starting with EF Core 3.0, this warning is now considered and error and an exception is thrown. 

**Why**

This change was made to drive better application code through exposing this pathological case more explicitly.

**Mitigations**

The most appropriate cause of action on encountering this error is to understand the root cause and stop creating so many internal service providers. However, the error can be converted back to a warning (or ignored) via configuration on the `DbContextOptionsBuilder`. For example:

```C#
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .ConfigureWarnings(w => w.Log(CoreEventId.ManyServiceProvidersCreatedWarning));
}
```

## The Relational:TypeMapping annotation is now just TypeMapping

[Tracking Issue #9913](https://github.com/aspnet/EntityFrameworkCore/issues/9913)

This change was introduced in EF Core 3.0-preview 2.

**Old behavior**

The annotation name for type mapping annotations was "Relational:TypeMapping".

**New behavior**

The annotation name for type mapping annotations is now "TypeMapping".

**Why**

Type mappings are now used for more than just relational database providers.

**Mitigations**

This will only break applications that access the type mapping directly as an annotation, which is not common. The most appropriate action to fix is to use API surface to access type mappings rather than using the annotation directly.

## ToTable on a derived type throws an exception 

[Tracking Issue #11811](https://github.com/aspnet/EntityFrameworkCore/issues/11811)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, `ToTable()` called on a derived type would be ignored since only inheritance mapping strategy was TPH where this is not valid. 

**New behavior**

Starting with EF Core 3.0 and in preparation for adding TPT and TPC support, `ToTable()` called on a derived type will now throw an exception to avoid an unexpected mapping change in the future.

**Why**

Currently it is not valid to map a derived type to a different table. This change avoids breaking in the future when it becomes a valid thing to do.

**Mitigations**

Remove any attempts to map derived types to other tables.

## ForSqlServerHasIndex replaced with HasIndex 

[Tracking Issue #12366](https://github.com/aspnet/EntityFrameworkCore/issues/12366)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, `ForSqlServerHasIndex().ForSqlServerInclude()` provided a way to configure columns used with `INCLUDE`.

**New behavior**

Starting with EF Core 3.0, using `Include` on an index is now supported at the relational level. Use `HasIndex().ForSqlServerInclude()`.

**Why**

This change was made to consolidate the API for indexes with `Includes` into one place for all database providers.

**Mitigations**

Use the new API, as shown above.

## EF Core no longer sends pragma for SQLite FK enforcement

[Tracking Issue #12151](https://github.com/aspnet/EntityFrameworkCore/issues/12151)

This change was introduced in EF Core 3.0-preview 3.

**Old behavior**

Prior to EF Core 3.0, EF Core would send `PRAGMA foreign_keys = 1` when a connection to SQLite is opened.

**New behavior**

Starting with EF Core 3.0, EF Core no longer sends `PRAGMA foreign_keys = 1` when a connection to SQLite is opened.

**Why**

This change was made because EF Core uses `SQLitePCLRaw.bundle_e_sqlite3` by default, which in turn means that FK enforcement is switched on by default and does not need to be explicitly enabled each time a connection is opened.

**Mitigations**

Foreign keys are enabled by default in SQLitePCLRaw.bundle_e_sqlite3, which is used by default for EF Core. For other cases, foreign keys can be enabled by specifying `Foreign Keys=True` in your connection string.

## Microsoft.EntityFrameworkCore.Sqlite now depends on SQLitePCLRaw.bundle_e_sqlite3

**Old behavior**

Prior to EF Core 3.0, EF Core used `SQLitePCLRaw.bundle_green`.

**New behavior**

Starting with EF Core 3.0, EF Core uses `SQLitePCLRaw.bundle_e_sqlite3`.

**Why**

This change was made so that the version of SQLite used on iOS consistent with other platforms.

**Mitigations**

To use the native SQLite version on iOS, configure `Microsoft.Data.Sqlite` to use a different `SQLitePCLRaw` bundle.
