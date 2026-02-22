---
title: "Beyond the Spreadsheet: Architecting Azure Infrastructure for 450k+ Order Migrations"
date: 2026-02-22 11:44:00 +0800
categories: [Projects, Fulfillment Server]
tags: [Cloud Architecture, Azure, System Design]
---

> ### 📊 System Specs at a Glance
> * **Dataset Size:** 440,000+ Order Records (Scaling to 1M+)
> * **MVP Arbitrator:** Google Sheets & Drive
> * **Production Stack:** Azure Container Apps + Azure Functions
> * **Sync Frequency:** Hourly (ShipStation API)
> * **Operational Cost:** ~$28/month

## The MVP Phase: Engineering Around Constraints

![MVP Architecture](/assets/2026-02-22-fulfillment-system-architecture/mvp-architecture.png)

When I began this project, I faced a significant hurdle: I had no access to Azure Cloud Services or a dedicated domain to host an MVP. To bridge the gap between the client and the server without traditional networking infrastructure, I designed a custom communication layer using **Google Sheets** as a **"Network Arbitrator."**

In this ecosystem, Google Sheets acted as the message broker, facilitating data exchange and state management between the frontend and the backend.

### The Backend Ecosystem

Before diving into the arbitrator logic, here is how the core data was managed:
- **Database:** A MySQL instance storing 440k+ order records.
- **Fulfillment API:** A central service handling CRUD operations and business logic.
- **Data Sync:** A "ShipStation API Migration" scheduler running every hour as a cron job within the API to fuel the DB with the latest order info.
- **Security:** Authentication handled via the **Google Login API.** The server validated the Google token and issued a custom access token to the user.

### The Communication Workflow: "The Sheet as a Socket"

To simulate a standard HTTP request/response cycle, I utilized a Google Sheet and Google Drive as the transport layer.

#### 1. The Request (Client Side)
When a user saves an order, the client uploads the payload as a JSON file to **Google Drive** and appends a row to the **Google Sheet**:

| Request Route | Type | Status      | Client Attachment ID | Server Attachment ID |
| :------------ | :--- | :---------- | :------------------- | :------------------- |
| `/orders`     | POST | **Pending** | URL to Drive file    | N/A                  |

The client then enters a polling state, watching the row for updates.

#### 2. The Processing (Server Side)
The server runs a listener that polls the sheet every second for rows marked `Pending`. Once detected:
1. The server fetches the JSON payload from the `Client Attachment ID`.
2. It processes the business logic and saves the data to **MySQL**.
3. It generates a response JSON, saves it to **Google Drive**, and updates the sheet.

#### 3. The Completion & Response
The server updates the row to signal the transaction is finished:

| Request Route | Type | Status       | Client Attachment ID | Server Attachment ID     |
| :------------ | :--- | :----------- | :------------------- | :----------------------- |
| `/orders`     | POST | **Complete** | URL to Drive file    | **URL to Response File** |

The client sees the `Complete` status, fetches the response JSON, and renders the result.

### Key Takeaway
While unconventional, this **Sheet-as-a-Service** architecture allowed the project to move forward without a cloud provider. It treated rows as network packets and Google Drive as a data buffer, proving that creativity can overcome almost any technical constraint.

---

## Scaling to Production: Migrating to Azure

![Production Architecture](/assets/2026-02-22-fulfillment-system-architecture/production-architecture.png)

After the MVP was approved, the project outgrew its "Arbitrator" phase. I migrated to a formal Azure infrastructure, moving from polling to a **standard TCP/IP and HTTP-based architecture.**

### The Production Stack

To handle high-volume data and complex synchronization, I utilized a suite of Azure services:

#### 1. Traffic Management (Azure DNS)
While the domain was purchased via GoDaddy, I transitioned management to an **Azure DNS Zone**. This provides a seamless point of entry for API traffic and simplifies SSL/TLS termination within the Azure ecosystem.

#### 2. The Core API (Azure Container Apps)
The "Fulfillment API" was containerized and deployed via **Azure Container Apps**. This acts as the backbone of the system, handling:
- **Custom Authentication:** Validating identities and managing JWT tokens.
- **RESTful Endpoints:** Processing all CRUD operations for orders and robots.
- **Auto-scaling:** Ensuring the server stays responsive regardless of traffic spikes.

#### 3. Decoupled Logic (Azure Functions)
To keep the main API performant, I moved background tasks to **Azure Functions**. These serverless workers handle three critical schedulers:

* **ShipStation Migration:** Pulls the latest order data hourly into Azure SQL.
* **DB Maintenance:** Periodically refreshes indexes and compiles statistics to keep queries fast across nearly half a million records.
* **Orphan Robot Migration:** > **The "Orphan" Logic:** Occasionally, robot data arrives before the associated order exists in the DB. To prevent data loss, these are stored in a temporary `OrphanTable`. This scheduler checks for newly arrived orders hourly and migrates "orphaned" robots to the primary tables once their parent records are found.

### Why This Matters

By moving to this architecture, we replaced high-latency polling with immediate HTTP requests and offloaded heavy processing to background functions. We moved from a fragile "workaround" to a **self-healing system** where data integrity and performance are handled automatically.


### Cloud Cost Breakdown (Monthly)

By utilizing serverless components and efficient database tiers, the production environment remains highly cost-effective, totaling approximately **$25.00 – $28.00 per month**.

| Service                        | Configuration / Tier             | Monthly Cost         |
| :----------------------------- | :------------------------------- | :------------------- |
| **Azure Container App (Prod)** | 0.25 vCPU, 0.5 GB (Min Scale: 1) | $4.00 – $7.00        |
| **Azure Container App (Test)** | 0.25 vCPU, 0.5 GB (Min Scale: 0) | $0.00 (Free)         |
| **Azure SQL DB (Prod)**        | Standard S0 (10 DTUs)            | $14.90               |
| **Azure SQL DB (Test)**        | Basic (5 DTUs)                   | $4.90                |
| **Azure DNS Zone**             | 1 Hosted Zone                    | $0.50                |
| **Azure Functions**            | Consumption Plan (3 Schedulers)  | $0.00 – $1.00        |
| **Total Overhead**             |                                  | **~$25.30 – $28.30** |

> **Cost Optimization Tip:** Setting the Test environment's minimum scale to 0 ensures that compute costs are only incurred during active development, effectively keeping that branch of the infrastructure free of charge when idle.