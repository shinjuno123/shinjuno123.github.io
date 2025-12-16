---
title: "Start project"
date: 2025-12-13 00:00:00 +0800
categories: [asp.net]
tags: [asp.net]
---

## List types of projects

I can display all list of dotnet project to create with the command below

```
dotnet new list
```

## Start .net project

In the beginning, it is better to start from the empty project so that I can better understand how each pieces are assembled together.

```
1. Ctrl + Shift + p
2. Start a project with ASP.Net (Empty)
```

## Each File explanation

### Program.cs: It bootstraps the application.
```cs
var builder = WebApplication.CreateBuilder(args);

// It is going to start listening HTTP requests
// build instance of  the web application
var app = builder.Build();

// root GET -> response will be "Hello World"
app.MapGet("/", () => "Hello World!");

app.Run();
```


### GameStore.Api.csproj
   - This is where we are going to add all dependencies.

```xml
<!-- What type of project? web project -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <!-- We have access to all apis that are available at net9.0 -->
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>

```

### appsettings.json/appsettings.Development.json

#### appsettings.json 
- This is used for configurations
- I need to put everything that shouldn't be harcoded on the code
- This file is used for production
```json

{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}

```

#### appsettings.Development.json
- This file is only used for development
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}

```

#### launchSettings.json
- It provides profiles
- It only provides configurations only for development (only for local development)
- I see http and https profiles for this project

```json
{
  "$schema": "https://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "http://localhost:5115",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "https://localhost:7081;http://localhost:5115",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}

```


## How to build the project


### On UI
- Click mouse right on the solution
- Click "build"

### On command

```
.net build
```

### Shortcut
- Ctrl + shift + b


## How to run the project

### Start with debugging
- f5 key
- Choose c#
- Select a profile


### Start without debugging 1
- Right click on the solution
- Go to Debug > "Start without debugging"


### Start without debugging 2

- Go to the project folder
```
dotnet run
```