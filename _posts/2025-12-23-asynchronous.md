---
title: "Performing Asynchronous Work"
date: 2025-12-23 00:00:00 +0800
categories: [asp.net]
tags: [asp.net]
---


# Asynchronous

## What is asynchronous?
- If you do one task each time, that is synchronous. You won't start the next task before the current task ends.
- If you do multiple task at the same time,  that is asynchronous. You will start other tasks simulanteously.


## How this works on ASP.NET?

Let's say we have 3 clients. Your phone and the tablet and the browser on the laptop. Your phone sends a request to webserver. When tablet sends a request while the server is handling the reqeust from the phone. Then, the server also starts handling the tablet's request at the same time with the phone's request.


## What is benefit?
- Improved Performance: Avoids blocking callers, freeing then up for other tasks
- Improved Scalability: Allows your application to handle more requests and users simultaneously.
- Simplified Code: Asynchronous code is simple to write via task objects and the async and await keywords.

_Please click [this](https://github.com/shinjuno123/asp-net-tutorial/tree/b76346196c96d44a2ec128b9e4dedb37ffd0361b) to view the ASP.NET Core tutorial repo at this time_