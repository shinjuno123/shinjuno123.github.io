---
title: "Handling Valid Inputs"
date: 2025-12-20 00:00:00 +0800
categories: [asp.net]
tags: [asp.net]
---

## Data Annotation Package
- This works on ASP.NET MVC. However, it doesn't work on ASP.NET Core API

```csharp
using System.ComponentModel.DataAnnotations;

namespace GameStore.Api.Dtos;

public record class CreateGameDto(
    [Required][StringLength(50)] string Name, 
    [Required][StringLength(20)] string Genre, 
    [Range(1, 100)] decimal Price,
    DateOnly ReleaseDate
);

```

## Download the MinimalApis.Extensions

```
dotnet add package MinimalApis.Extensions --version 0.11.0
```

## Add .WithParameterValidation() to group or app

```csharp
var group = app.MapGroup("games")
    .WithParameterValidation();
```


## Response
- Now, the validation works perfectly. It gives nice error messages for each field

```json
HTTP/1.1 400 Bad Request
Connection: close
Content-Type: application/problem+json
Date: Sat, 20 Dec 2025 23:53:52 GMT
Server: Kestrel
Transfer-Encoding: chunked

{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Price": [
      "The field Price must be between 1 and 100."
    ]
  }
}

```

_You can view the entire code in github for this step of the project [by clicking here](https://github.com/shinjuno123/asp-net-tutorial/tree/e6524fc708faf8bbdd9e757d6f00a1dfe93d47fe)_