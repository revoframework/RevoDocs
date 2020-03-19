---
description: >-
  Database migrations in Revo are a simple feature how to version database
  schema.
---

# Database migrations

## Overview

Migrations are simple SQL scripts that are applied to database anytime an upgrade is needed. Revo automatically tracks the latest version \(using semantic versioning with dots, e.g. 1.2.3\) of every migration module you define and if it finds a migration to newer version, it applies it \(and all subsequential migrations, in order\).

{% hint style="success" %}
To deal with changes in domain event definitions for events stored in an event store, you should rather look into [event upgrades](events.md#event-upgrades). Revo database migrations described in this section are better suited to upgrade your read models.
{% endhint %}

By default, migrations are only applied automatically in debug mode \(for safety reasons\) and framework tries to upgrade all modules for which it finds new migrations upon the startup of the application. This can be overriden with [configuration](database-migrations.md#configuration). Alternative way to run the migrations \(e.g. in production, using CI/CD\) is to use the [CLI](database-migrations.md#cli-revo-dbmigrate).

{% hint style="info" %}
All migrations during an upgrade are always applied transactionally.
{% endhint %}

## Using migrations

### Register migrations

To automatically discover migrations in your project, add them to a folder in your project \(e.g. _Sql_\) and compile them as **embedded resources**. In a `NinjectModule` in your project, you can register the migrations in an assembly then like this:

```csharp
Bind<ResourceDatabaseMigrationDiscoveryAssembly>()
  .ToConstant(new ResourceDatabaseMigrationDiscoveryAssembly(GetType().Assembly, "Sql"))
  .InSingletonScope();
```

You can also specify custom migration file name regex \(by default it finds all _.sql_ files with names adhering to a convention\) or configure auto-discovery in a local file system directory using `FileDatabaseMigrationDiscoveryPath`.

Default migration filename format is as follows:

```csharp
module-name_1.0.0_pgsql.sql
```

* Module name may contain letter, numbers and hyphen and defines the scope for versioning.
* Version \(here 1.0.0\) must be specified using the semantic versioning notation \(i.e. both _1_ or _1.2.3.4.5_ are also fine\). Version can be omitted if specified inside the file or if migration is repeatable.
* _pgsql_ \(optional\) here denotes a migration tag - using tags, you can define the same migration version differently for different database systems, environments \(debug/production\), etc.

### Migration file headers

Besides naming the migrations files using a conventions, many migration attributes can also be specified inside the migration file itself using SQL comments at the beginnin of the SQL script.

Example:

{% code title="app-main.sql" %}
```sql
-- this is an example SQL migration
-- version: 1.1.0
-- description: Description is only informative.
-- dependency: vendor-xyz
-- dependency: app-commons@1.2.0

-- SQL migration script itself...

CREATE TABLE app_mytable (
	app_mta_mytable_id uuid PRIMARY KEY
	-- etc.
);
```
{% endcode %}

The example shows all supported header options:

* **version**: You can specify the version in the file instead of specifying it in the file name, which may be useful if you want to prevent renaming the migration file often.
* **description:** Description is only informative and gets stored in the database migration history table.
* **dependency**: A migration script can specify any number of dependency script that must be installed first before proceeding with its own execution. Dependency can also be specified with specific version \(_app-commons@1.2.0_\) or to the latest module version \(example of _vendor-xyz_ dependency\).

### Baseline migrations

Baseline migrations are optionally defined migrations that define a simplified script for creating a new schema from scratch and will only be used on empty  databases \(or more precisely, if the module has no previous migration history in used database\). For such cases, baseline migrations will always be preferred over regular migrations \(if available\). For example, this means that if we define these migrations

* _app-1.0.0.sql_
* _app-1.1.0.sql_
* _app\_baseline\_1.1.0.sql_,

and run them on an empty database, the baseline script will be used. However, if the module has previously been created using the _app-1.0.0.sql_ migration script, app-1.1.0.sql will be run now \(not the baseline\).

Migration can be marked as baseline by including _\_baseline_ after its module name in its filename, just before the version \(if specified in the filename\).

### Repeatable migrations

Repeatable migrations are a special kind of migrations that is not explicitly versioned and is only versioned using the checksum of its contents. Simply put, repeatable migrations are re-applied everytime their contents change.

Migration can be marked as repeatable by including _\_repeatable_ after its module name in its filename and they cannot have an version explicitly specified.

### Dependencies

As shown in [migration file headers example](database-migrations.md#migration-file-headers), migrations can also specify dependencies to other modules. A dependency to a module means that said module must be installed first before running the original migration itself. Dependencies can also be specified with specific version \(e.g. _app-commons@1.2.0_\) or to the latest module version \(when no version is specified\).

 Dependencies may have multiple levels, Revo always tries to resolve a correct dependency tree for corresponding migration path.

## CLI \(revo-dbmigrate\)

Database migrations CLI allow you to run the migrations outside of the execution scope of your project. This is useful when you need to run the migration manually from a script or from CI/CD, for exaple.

You can install the CLI it as a global .NET Core tool:

```bash
dotnet tool install -g Revo.Tools.DatabaseMigrator
```

This installs the latest CLI version from NuGet. Afterwards, you can use the tool using the _revo-dbmigrate_ command from anywhere. Example:

```csharp
revo-dbmigrate install -c "Server=localhost;Port=5432;Database=mydb_test;User Id=postgres;Password=" -p Npgsql -a "C:\my\project.dll" -m "app-*" "*"
```

You can either _preview_ \(which only shows what will be done\) or _install_ \(which actually executes the migrations on the database\). You also either need to specify the path to your project DLLs \(so it can load the migrations from its embedded resources\) or specify the path on disk where the SQL scripts physically reside.

You can see the full description for all available CLI commands using the --help switch:

```csharp
revo-dbmigrate --help
```

## Configuration

Database migrations can be configured when setting-up Revo application \(e.g. in you Startup class\).  
Besides overriding wheter to apply migrations upon application startup \(on only in debug mode by default\), you can also affect which modules to migrate \(which can also use module name wildcards\) or the target versions to upgrade to.

```csharp
return new RevoConfiguration()
    ...
    .ConfigureInfrastructure(
      cfg =>
      {
        //automatic migrations always on
        cfg.DatabaseMigrations.ApplyMigrationsUponStartup = true;
        
        //first migrate modules with name beginnin 'app-', then migrate everything
        cfg.DatabaseMigrations.MigrateOnlySpecifiedModules = new List<DatabaseMigrationSearchSpecifier>()
        {
          new DatabaseMigrationSearchSpecifier("app-*", null),
          new DatabaseMigrationSearchSpecifier("*", null)
        };
      });
```

