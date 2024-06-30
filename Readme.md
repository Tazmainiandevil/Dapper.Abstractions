# Dapper.Abstractions

[![Build status](https://ci.appveyor.com/api/projects/status/28kh570vv7wdmunk?svg=true)](https://ci.appveyor.com/project/Tazmainiandevil/dapper-abstractions)

<a href="https://badge.fury.io/nu/Dapper.Abstractions"><img src="https://badge.fury.io/nu/Dapper.Abstractions.svg" alt="NuGet version" height="18"></a>

Support for .NET Standard 2.0, .NET 6.0 and .NET 8.0

### **Breaking Change Notice**

We have introduced a breaking change in the release 4.x to improve the modularity and flexibility of our codebase. As part of this update, we have separated the SQL Client from `Dapper.Abstractions` into its own package, `Dapper.Abstractions.Sql`.

#### Impact:
- Users will need to update their code to use the new `Dapper.Abstractions.Sql` package instead of the `Dapper.Abstractions` package to continue utilizing the SQL Client functionality that was previously part of `Dapper.Abstractions`.


# Introduction

Dapper.Abstractions is a fork of DapperWrapper and is a library that wraps the [Dapper](https://github.com/StackExchange/dapper-dot-net) extension methods on `IDbConnection` to make unit testing easier.

Why bother? Because stubbing the extension methods used in a method-under-unit-test is not simple. For instance, you can't just use a library like [Moq](https://github.com/moq/moq4) or [NSubstitute](http://nsubstitute.github.io/) to stub the `.Query` extension method on a fake `IDbConnection`. To work around this, this library introduces a new abstraction, `IDbExecutor`.

## The `IDbExecutor` Interface

The `IDbExectuor` interface has many methods, each corresponding to a Dapper extension method: `Execute`, `Query`, `Query<T>`, `QueryMultiple`, `QueryMultiple<T>`, etc.. Wherever you would previously inject an `IDbConnection` to use with Dapper, you instead inject an `IDbExecutor`. There is a single implementation of `IDbExecutor` included in Dapper.Abstractions, `SqlExecutor`, that uses the Dapper extension methods against `SqlConnection`. Adding your own `IDbExecutor` against other implementations of `IDbConnection` is easy.

### Example use of `IDbExecutor`:

```C#
public IEnumerable<SemanticVersion> GetAllPackageVersions(string packageId, IDbExecutor dbExecutor)
{
  return dbExecutor.Query<string>("SELECT p.version FROM packages p WHERE p.id = @packageId", new { packageId })
    .Select(version => new SemanticVersion(version));
}
```

### Example use of `IDbExecutorFactory`:

```C#
  private IDbExecutorFactory _dbExecutorFactory;
  public UserAccess(IDbExecutorFactory dbExecutorFactory)
  {
      _dbExecutorFactory = dbExecutorFactory
  }

  public IEnumerable<User> GetUsers()
  {
      using var db = _dbExecutorFactory.CreateExecutor();
      var data = db.Query<User>("SELECT ID, Name FROM Users");
      return data;
  }
```

## Injecting `IDbExecutor`

You probably already have an approach to injecting `IDbConnection` into your app that you're happy with. That same approach will probably work just as well with `IDbExecutor` or `IDbExecutorFactory`.

### Example ASP.NET .NET 6

In Program.cs

```C#
using Dapper.Abstractions;
using Dapper.Abstractions.Sql;

var builder = WebApplication.CreateBuilder(args);
...
var connectionString = builder.Configuration["database:connectionstring"];

var dbExecutorFactory = new SqlExecutorFactory(connectionString);

builder.Services.AddSingleton<IDbExecutorFactory>(dbExecutorFactory);
...
```

### Example .NET 6 Console

```C#
using Dapper.Abstractions;
using Dapper.Abstractions.Sql;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

using IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((builder, services) =>
        services.AddSingleton<IDbExecutorFactory>(new SqlExecutorFactory(builder.Configuration["DatabaseConnectionString"])))
    .Build();

```
or
```C#
using Dapper.Abstractions;
using Dapper.Abstractions.Sql;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

using IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((builder, services) =>
        services.AddSingleton<IDbExecutor>(new SqlExecutor(new SqlConnection(builder.Configuration["DatabaseConnectionString"]))))
    .Build();
```

## Additional Extensions

There are also times when the data coming from the database is not trimmed and so Dapper.Abstractions includes `QueryAndTrimResults<T>` for this purpose.

### Example usage

```C#
  private IDbExecutorFactory _dbExecutorFactory;
  public UserAccess(IDbExecutorFactory dbExecutorFactory)
  {
      _dbExecutorFactory = dbExecutorFactory
  }

  public IEnumerable<User> GetUsers()
  {
      using var db = _dbExecutorFactory.CreateExecutor();
      var data = db.QueryAndTrimResults<User>("SELECT ID, Name FROM Users");
      return data;
  }
```

## Transactions

Sometimes there is a need to assert whether a method-under-unit-test completes a transaction via `TransactionScope`. To make this easier, Dapper.Abstractions also has an `ITransactionScope` interface (and `TransactionScopeAbstraction` implementation) that makes it easy to create a fake transaction, and stub (and assert on) the `Complete` method. As with `IDbExecutor`, you can bind it directly, via `Func<ITransactionScope>`.
