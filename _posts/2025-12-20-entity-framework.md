---
title: "Entity Framework (ORM)"
date: 2025-12-21 00:00:00 +0800
categories: [asp.net]
tags: [asp.net, orm]
---

# Object Relational Mapping

## Why do we need ORM?

### Code to Write in PURE REST API - SQL
- Translate Web API request to SQL query
- Send SQL query to DB Server
- Read resulting database rows
- Translate database rows to Web API response

### Problems
- We need to learn new language (Not a problem for me, it could be still time-consuming)
- We need a lot of additional data-access code
- This is error pron.
- We need to manually keep C# models in sync with DB tables;

### What is ORM?

- Let's say if there are Songs, Artists, and Playlists Objects in my Object Oriented Program
- Most likely, there will be Songs, Artists, and Playlists tables in the relational DB
- ORM sets a map between the program and the relational DB so that ORM takes care of transformaing objects to tables and vice versa.

### What is Entity Framework Core?
- A lightweight, extensible, open source and cross-platform **object-relational mapper for .NET**

#### Benefits of using EFC
- No need to learn a new language
- Minimal data-access code (LINQ)
- Change tracking
- Multiple database providers

### How to set up EFC (SQLite)
- I have used appsettings.json below. I recommend using "Secret Manager" for the development purpose
- For cloud-based application in production, it is recommended to use service like AWS secret Manager, Azure Key Vault Or Use User-Secrets.

```csharp
var builder = WebApplication.CreateBuilder(args);

var connString = builder.Configuration.GetConnectionString("GameStore");
```

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "GameStore": "Data Source=GameStore.db"
  }
}


```

#### DB Migrations
```
dotnet tool install --global dotnet-ef --version 9.0.11
```

```
dotnet add package Microsoft.EntityFrameworkCore.Design --version 9.0.11
```

```
dotnet ef migrations add InitialCreate --output-dir Data/Migrations
```

```
dotnet ef database update
```

#### Start DB Migrations when the program starts up


```csharp

using Microsoft.EntityFrameworkCore;

namespace GameStore.Api.Data;

public static class DataExtensions
{
    public static void MigrateDb(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<GameStoreContext>();
        dbContext.Database.Migrate();
    }
}
```


#### Data Feeding

```csharp

using GameStore.Api.Entities;
using Microsoft.EntityFrameworkCore;

namespace GameStore.Api.Data;

public class GameStoreContext(DbContextOptions<GameStoreContext> options): DbContext(options)
{
    public DbSet<Game> games => Set<Game>();

    public DbSet<Genre> Genres => Set<Genre>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Genre>().HasData(
            new { Id = 1, Name = "Action" },
            new { Id = 2, Name = "Adventure" },
            new { Id = 3, Name = "Role-Playing" },
            new { Id = 4, Name = "Strategy" },
            new { Id = 5, Name = "Sports" },
            new { Id = 6, Name = "Simulation" }
        );
    }
}

```

_You can see the entire code [by clicking here](https://github.com/shinjuno123/asp-net-tutorial/tree/e24b79d91cdf2b2c4074fd7eddaa34db33eeb8d0)_