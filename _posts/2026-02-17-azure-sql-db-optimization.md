---
title: "From 5–Minute Timeouts to 1–Second Responses: Optimizing Azure SQL on a Budget"
data: 2026-02-21 23:00:00 +0800
categories: [Projects, Fulfillment Server]
tags: ["Azure SQL", "Query Optimization", "Cloud Architecture"]
---
I want to leave the post what problems I have encountered to enhance the speed of search queries while developing company's internal fulfillment API server.

I have migrated the following amount of data from 3rd party API and spread sheets to S0(10 DTUs) SQL DB in Azure. My company's fulfillment app is composed of the following tables:

- Orders: 440k rows
- OrderItems: 450k rows
- Items: 380k rows
- Packages: 450k rows
- OrphanItems: X rows
- Warehouses: X rows
- User: X rows (only internal company users)

Please see the endpoints below. The app has 2 very important endpoints that are related with this post.

- **/orders?location=US&from=2026-01-01&to=2026-02-21&page=1&limit=50&fulfillmentStatus=unfulfilled&sort=shipDate&order=desc**
- **/orders/summary?location=CA&from=2026-01-01&to=2026-02-21**

In this post, we are going to see /orders route in details with the optimization process

# Optimization Strategy

## Selectively Join

### The Problem: The "Data Explosion"

When you join tables with millions of rows without filtering early, the Database Engine tries to build a massive intermediate table in memory before it even gets to your `where` clause.

In our case, the theoretical scale was terrifying:
- **Orders**: 440k rows
- **OrderItems**: 450k rows
- **Items**: 380k rows

If the DB isn't optimized, it treats this like a massive multiplication problem. Trying to fetch 100k orders through a raw triple–join was taking **more than 5 minutes** and eventually crashing the DB. The server simply ran out of RAM trying to hold that many potential combinations.

#### The solution: Filtering Early with CTEs or Subqueries

The "Aha!" moment was realzing that we don't need to join the entire universe of items just to see 100k orders. We need to shrink the dataset first.

By using a **Common Table Expression (CTE)** or **Subquery**, I changed the execution order:
1. **Phase 1 (The Filter)**: Select only the specific 100k Orders we actually need.
2. **Phase 2 (The Join)**: Then join those 100k rows to the OrderItems and Items tables.

#### Why it's faster

Instead of the DB engine trying to manage billions of potential row combinations, it's only looking at a tiny fraction
> Analogy: It's the difference between searching every single page of every book in a library (Raw Join) versus picking 10 books off the shelf first and then looking for the chapter you need (CTE/Subquery)

This change took the "load" off the DB server's memory and turned an unusable, crashing endpoint into a stable, high—performance API.

## NO LOCK

When we were working on the database for the fulfillment system project, one of the biggest bottleneck was blocking.

Normally, if one part of the app is updating a record (let's say Order #4444), SQL Server puts a "lock" on it. If another user tries to read that same order at the exact same time, they have to wait in line until the first transaction is totally finished. On a busy app, those tiny waits add up fast and slow everything down.

To fix this, we used the NOLOCK hint.

### How it works
Think of NOLOCK as telling the database: "Hey, I'm in a hurry. I don't need to wait for everyone else to finsih their paperwork—just show me what's on the screen right now"
- Transaction A: Is currently updating Order #4444
- Transaction B (With NOLOC): Doesn't wait for A to finish. It just grabs the current value and keeps moving.

### Trade-off

The catch is that NOLOCK doesn't fetch a "perfect snapshot". It reads uncommitted data.

If Trasaction A was halfway through changing a price from $50 to $100, NOLOCK might grab that $100. But If Transaction A crashes and rolls back to $50 a second later, the query would just report a "dirty" value that isn't actually real.

### Why I chose NOLOCK

The real reason I implemented `NOLOCK` wasn't just for a single slow query; it was to handle out background data sync. Every 30 to 60 minutes, our app pulls data from a 3rd-party API to keepour records updated. Depending on the volume, this sync can take anywhere from 30 seconds to 10 minutes

#### 1. Preventing the "Sync Freeze" (Zero Blocking)

When that API sync is running, it is constantly writing and updating hundreds to thousands of rows. In a standard database setup, these "Writes" put **Exclusvie Locks** on the tables.
- The roblem: if a cs member tries to search for the customer's order information while the sync is happening, their request gets stuck begind the migration. They could be waiting for minutes.
- The solution: By using `NOLOCK` on our read queries, I ensured that users could still view their data instantly, even while the database was busy processing a massive 10-minute update in the background.

#### 2. Practical Accuracy (The 99.9% Rule)

When you're showing "Big Picture" stats—like a dashedboard showing _Total Orders Fulfilled Today_—absolute mathematical perfection for a split second is less important than **system availability**.
- **The Trade-off**: Using `NOLOCK` might mean a user sees 1005 orders instead of 1004 becauase of a "dirty read" from the sync
- **The Reality**: I'd rather show a user a number that is 99.9% accurate immediately than make the app 0% usable for 10 minutes while they wait for a lock to clear. 

## Pagination (``OFFSET``)

After optimizing our joins, I still had one more problem: Even if the query is fast, I shouldn't send 100,000 rows of data over the internet to the client. That is where Pagination comes in.

### 1. Saving "Network Bandwidth" (The Pipe is Small)

If each order record is about 2KB, sending 100k orders is 200Mb of data.
- **The problem:** The user's laptop has to download that 200MB, and your API server has to "Package" it all into JSON. This makes the app feel laggy, even if the database is fast.
- **The solution:** With `OFFSET`, we only send 20 or 50 rows rows at a time (e.g. "Page 1"). We only send what the user can actually see on their screen. 

### 2. Reducing "Memory Pressure" on the API

To send 100k rows, your Backend (Node.js, c#, Java, etc.) has to hold all 100k objects in its RAM before sending them to the frontend.
- **The risk:** Even with a smaller group of 15-20 users, if everyone requests 100k orders at the same time, the API server has to allocte a massive amount of RAM to stringify those objects into JSON. This causes CPU spikes and "Memory Pressure". If the server hits its limit, it could crash or start dropping other requests.
- **The benefit:** `OFFSET` keeps the backend "lean.". The API only handles a handful of records at a time, keeping memory usage flat and the server stable, regardless of how much data is actually in the database. In my case, I wanted to lower down the cost of cloud services as best as I could. So I have selected 0.25 vCPU and 0.5 GB memory. I needed to ensure that each request takes up the memory as low as possible. 

### 3. Efficiency Over Overkill

Even if our app only has 15-20 users, and our Azure Container server could technically "survive" a few big requests, it doesn't mean it should be awsting resources.

By using  pagination, we keep the Container's CPU and Memory Usage low. This is vital becauase:
- **Leaving "Headroom:"** Even if the migration logic lives in an Azure Function, both the Function and the Container are talking to the same Database.
- **Database Throughput:** While the Azure function is busy writing thousands of rows from the 3rd-party API, we don't want our Container server to fight for database resouces by requesting 100k rows at the same time.
- **Cost & Scalability:** Keeping the Container "lean" means we don't have to pay for a larger Azure tier just to handle inefficient queires.

## Indexing

A database without indexes is like a library where all the books are piled on the floor in no particular order. If you want to find a book, you have to look at every single one (this is called a **Table Scan**).

By applying indexes, I created a "Table of Contents" for our most important data.

### 1. Indexing Search Filters

In our API endpoint, the most common way users filtered data was by `ShipDate` and `location`
- **The probleme:** Wthout an index, every time a user looked for orders from "US" on "2026-02-21" the database had to check all 440k rows
- ***The solution**: I applied **Non-Clustered Indexes** to the Shipdate and Location columns. Now, th eDB can "jump" straight to the relevant records in milliseconds.

> Non Clustered Indexes: A Non-Clustered index is a separate structure from the actual table data. It contains a sorted list of the values from the columns you chose (like Location), and each value has a pointer (a Row ID) that tells the database exactly where the rest of that row's data is living.

### 2. Speeding up Joins (Foreign Keys)

Our tables have `Orders > OrderItems > Items` relationship involves heavy joins.
- **The Strategy:** I ensured that every **Foreign Key (FK)** used in a `JOIN` clause had an index.
- **The Result:** When joining `Orders` to OrderItem, the DB uses the index to instantly match the `OrderID` across tables instead of searching for matches manually. This is a huge reason why our over 5–min queries started running in under 1 min.

### The "Cost" of indexing (Why I didn't index everything)

I was careful not to index every column. While indexes make Reads faster, they make **Writes** (like our 3rd-party API migration) slighly slower because the database has to update  the index every time a new row is added.

I picked only the "High–traffic" columns–the ones actually being used in `WHERE` and `JOIN` clause–to get the best performance boost without hurting our background migration speed.

## The "Engine Room": Separating Batch Jobs from the Main App

To keep the application fast for users, I made key architectural decision: never let the main app handle heavy, repetitive data tasks. Instead, I moved these "Batch Jobs" into independent Azure Functions.

### 1. The Migrationn Job (The "Loader")

Every hour, we need to sync data from a 3rd–party API.
- **The Decision**: This job runs on its own Azure Function, separate from the main web server.
- **The Benefit**: If the 3rd-party API is slow or th data volume is huge, it doesn't eat up the memory or CPU that our users need. It's like having a separate loading dock for a busy restaurant so the front door stays clear for customers.

### 2. The Maintenaance Job (The "Optimizer")

As the Migration Job adds thousands of new rows, the database's "maps" (Statistics and Indexes) can become disorganized or "stale."
- **The Decision:** I configured a second Azure Function to automatically run `UPDATE STATISTICS` and refresh indexes.
- **The Benefit:** This keeps the database "intelligent." It ensures that my performance optimizations (like Non-Clustered Indexes) stay fast over time. Without this, a query that takes 10 seconds today ight crawl back to 5 minutes next week as the data grows.

> **Key takeaway:** By isolating these tasks, I ensured that "Writing" data and "Maintaining" data never interfered with the user's "Reading" of data.

## 🚀 Breaking the 2 GB Ceiling: Moving to the Standard Tier

As the increase of data by migrating the old data to this system, I hit a hard limit. My database was originally on the Basic Tier, which is limited to a maximum of **2 GB** of storage.

As I applied my **Non-Clustered Indexes** to handle our 440k+ rows, the database size expanded. Since an index is essentially a separate, sorted copy of the data, those "speed shortcuts" took me right past that 2 GB limit.

### The Upgrade: Baisc -> Standard (10 DTUs)

To keep the system running, I migrated the database from the Basic Tier to the **Standard Tier (S0/S1).**
- **Storage Expansion:** The Standard Tier immediately bumped our storage limit from 2 GB to 250 GB, giving us plenty ofroom for future growth and even more indexes.
- **Performance Boost:** Moving to 10 DTUs didn't just give me space; it provided more consistent "Transaction Power."

This extra power was vital because it gave the database more "horsepower" to handle the **Azure Function migration** and the **user's aggregation queries** at the exact same time wihtout slowing down.

> **Technical Insight:** In Azure, sometimes you scale for speed, but somtimes you scale for "Physical Reality." Upgrading to the Standard tier was a necessary step to support the indexing strategy that saved the performance.

# 🎯 The Final Scoreboard: Performance Transformation

By layering architectural changes with deep database optimizations, I transformed an unsusable, crashing system into a stable, high performance API.

## The Progress of Speed:
- **Baseline (The Crash): 5+ Minutes.** Raw triple-joins on 440k+ rows caused memory exhaustion and timeouts.
- **Phase 1 (Indexing): < 1 Minute.** Manually indexing Foreign Keys and search columns (`ShipDate`,`Location`, etc) stopped full table scans.
- **Phase 2 (CTE & Filtering): 10 Seconds.** Using CTEs to filter 100k orders before joining items reduced the "Data explosion."
- **Phase 3 (Final Performance Benchmarks):** With all optimizations (Pagination, CTEs, and indexing) working together:
  - **20k rows:** Processedand fetched in **~1 second.**
  - **100k rows:** Aggregated and fetched in **6-7 seconds.**