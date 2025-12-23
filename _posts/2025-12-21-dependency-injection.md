---
title: "Dependency Injection (DI) & Dependency Inversion"
date: 2025-12-21 00:00:00 +0800
categories: [asp.net]
tags: [asp.net]
---

# Dependency Injection (DI)

## What is a Dependency Injection?

### Description by Example without DI
- Let's say there are 2 classes called "MyService" and "MyLogger"
- MyService uses LogThis("foo) of "MyLogger"
- "MyService" is using some functionalities of "MyLogger".
- In the "MyService", it creates the instance of "MyLogger". After that, "MyService" can use "MyLogger"'s methods in it.


### What is the problem here?
- The author of "MyLogger" decides to slighly modify it. So he creates "MyFileWriter" and inject it into MyLogger whenever creating logger instance as below.

```csharp

public MyService()
{
    var writer = new MyFileWritter("output.log");
    var logger = new MyLogger(writer);
    logger.LogThis("I'm Ready!");
}
```

- "MyService" is **tightly coupled** to teh Logger dependency. **Any changes to MyLogger require chanes to MyService**
- MyService **needs to know** how to _construct and configure MyLogger dependency_.
- It's hard to **test** MyService since the **MyLogger dependency cannot be mocked or stubbed**.

### How to solve this problem?
- MyLogger is passed in as a constructor parameter.

```csharp

public MyService(MyLogger logger) {
    logger.LogThis("I'm Ready!")
}

```

- Now, the service doesn't need to know how to construct or configure the logger.


### Who does create the instance then?
- ASP.NET Core provides IServiceProvider, which is known as Service Container.
- Your app can register any dependencies with IServiceProvider.
- When HTTP request arrives and your service needs the instance of MyService, the service container will notice its dependencies and it will resolve, construct and inject dependencies.

### what is good?
- MyService won't be affected by changes to its dependencies
- MyService doesn't need to know how to construct its dependences
- Dependencies can also be injected as parameter to minimal API endpoints.
- It opens the door to using Dependency Inversion.


# Dependency Inversion


## What is Dependency Inversion?
- Principle is the following
   - "Code should depend on abstractions as opposed to concrete implementations"

### Example

Let's say MyService is using MyLogger. We want to change MyLogger to CloudLogger. We could modify MyService to receive and use CloudLogger instance.

Instead, we are going to make MyService to use ILogger interface instead, which provides all required logging functionality. Then, we can have both Mylogger and CloudLogger implement this new interface. 

In this way, we are decoupling MyService from the Logger dependencies.


```csharp

public MyService(ILogger logger) {
    logger.LogThis("I'm Ready!")
}

```


### Benefits
- The logger dependency can be swapped out for a different implementation without modifying MyService.
- It's easier to test MyService since the logger dependency can be mocked or stubbed.
- Code is cleaner, easier to modify and easier to understand. 



### When should instances be created?
- MyLogger --- (register) ---> IServiceProvider 
- When HTTP Request comes in, IServiceProvider ---- (Resolve, construct, inject) ----> MyService (is using MyLogger)
- new request comes in, you can configure
   - IServiceProvider to create a new MyLogger instance.
   - IServiceProvider to re-use the same MyLogger instance.

### Life Times
- Transient: Create a new insance whenever any class needs it. 
   - Whenever new HTTP request comes in, IServiceProvider will resolve, construct, and inject MyLogger
   - If there is any other service using MyLogger, it will also create a brand-new instance of it.
- Scoped: Keeps track of some state and shared through multiple services (created when new HTTP comes in and shared within the request)
   - When a new HTTP request comes in, IServiceProvider will resolve, construct, and inject MyLogger
   - If there is any other service using MyLogger, it will use the same dependency.
   - However, If another new HTTP arrives, the service container will create a new instance that is unrelated with the previous instance.
- Singleton: The instance is shared through all classes (re-used when new HTTP comes in and shared through all the services)
   - When a new HTTP request comes in, IServiceProvider will resolve, construct, and inject MyLogger
   - If there is any other service using MyLogger, it will use the same dependency.
   - However, If another new HTTP arrives, the service container will also provide the same instance.


#### Reference
_Please click [here](https://github.com/shinjuno123/asp-net-tutorial/tree/280a3526ccac987fc235382b3886594067a79376) to see the entire code base_