# Installation

Install TickerQ.EntityFrameworkCore to enable database persistence for TickerQ jobs.

## Package Installation

### Package Manager Console

```powershell
Install-Package TickerQ.EntityFrameworkCore
```

### .NET CLI

```bash
dotnet add package TickerQ.EntityFrameworkCore
```

### PackageReference

```xml
<PackageReference Include="TickerQ.EntityFrameworkCore" Version="8.0.0" />
```

## Prerequisites

- **.NET 8.0** or later
- **TickerQ** package (core library)
- **Entity Framework Core** database provider for your chosen database

### Database Providers

Install the appropriate EF Core provider for your database:

**PostgreSQL:**
```bash
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

**SQL Server:**
```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

**MySQL:**
```bash
dotnet add package Pomelo.EntityFrameworkCore.MySql
```

**SQLite:**
```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
```

## Version Management

::: warning Important
All TickerQ packages are versioned together. Always update all packages to the same version.
:::

```bash
dotnet add package TickerQ --version 8.0.0
dotnet add package TickerQ.EntityFrameworkCore --version 8.0.0
```

## Next Steps

- [Setup Guide](./setup/index) - Configure Entity Framework Core integration
- [Configuration Reference](/api-reference/configuration/entity-framework-configuration) - Complete API documentation

