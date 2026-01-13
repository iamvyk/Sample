# BAMS Prescribed Events Integration  
**Windows Forms | .NET Framework 4.8**

---

## Project Overview

This project implements an integration between **Visual Approvals** and the **Building Activity Management System (BAMS)** to submit **Prescribed Events** for building permits using the official **BAMS REST API**.

The solution is designed to handle **high-volume permit submissions** efficiently while remaining compliant with BAMS API constraints and regulatory timeframes (e.g. Regulation 47).

### Key Capabilities

- Submits **one permit (with multiple prescribed events)** per API request, as required by BAMS
- Supports **high-volume scenarios** (e.g. 600 permits)
- Uses **controlled concurrency** to reduce overall processing time
- Avoids API overload and latency issues caused by excessive parallel requests
- Provides **full audit logging** for compliance and troubleshooting

The integration is implemented in accordance with the  
**“BAMS REST API – Developer Reference Guide v2.0”**.

---

## High-Level Architecture

**Design principles:**

- One permit per request (BAMS constraint)
- Multiple requests in parallel, but throttled
- Single shared OAuth token (cached and refreshed safely)
- Fully asynchronous, non-blocking execution


---

## Main Components

### UI Layer : `PermitManagementForm`

**Responsibilities:**
- Allows users to select permits and prescribed events
- Initiates batch submission
- Displays submission progress and results

**Notes:**
- Does **not** call the BAMS API directly
- Delegates all submission logic to the batch service

---

### Batch Orchestration : `BamsBatchSubmissionService`

**Responsibilities:**
- Coordinates submission of multiple permits
- Applies **controlled concurrency** using `SemaphoreSlim`
- Tracks progress and aggregates results

**Why this exists:**
- Sequential submission of 600 permits can take 30-40 minutes
- Sending too many concurrent requests risks overwhelming BAMS
- A bounded semaphore (e.g. **5 concurrent requests**) provides optimal throughput

**Execution flow:**
1. Accept list of permits
2. Acquire semaphore slot
3. Submit permit asynchronously
4. Release semaphore on completion
5. Continue until all permits are processed

---

### Core Integration (VA.Bams.Submission proj) : `BamsSubmissionService` 

**Responsibilities:**
- Handles OAuth authentication
- Builds and submits Prescribed Events payloads
- Parses BAMS API responses
- Persists results and errors to the database

This is the **core integration layer** that communicates with BAMS.

---

## Authentication Model

Authentication follows the **JWT Bearer OAuth flow** defined by BAMS:

1. Client sends a signed JWT to the Salesforce OAuth endpoint
2. Salesforce validates the JWT
3. Salesforce returns:
   - `access_token`
   - `instance_url`

All subsequent BAMS API calls include:


### Token Management

The integration implements:
- Token caching
- Expiry tracking with a safety buffer
- Semaphore-protected refresh to prevent token stampede

This ensures:
- Only one token refresh occurs under concurrency
- All API calls reuse a valid token

---

## Prescribed Events Submission Flow

### End-to-End Flow (Single Permit)

1. Authenticate (or reuse cached token)
2. Build JSON payload from `BamsPermit`
3. Log submission request to the database (`InProgress`)
4. POST to: {instance_url}/services/apexrest/v1/permit/prescribed-events

5. Receive API response
6. Parse results, warnings, and errors
7. Persist:
   - Submission status
   - Permit-level errors
   - Event-level outcomes
8. Update prescribed event statuses

---

## Concurrency & Performance Design

### Why Concurrency Is Used

- Each API call takes 2-4 seconds
- 600 permits sequentially takes 30-40 minutes

### Why Concurrency Is Limited

When concurrency is too high (e.g. semaphore = 10+):

- Requests are accepted by BAMS
- Responses slow down
- Latency increases due to server-side queuing or throttling

This behaviour is known as **API backpressure**.

### Practical Approach

The recommended strategy is:

> **Send requests in quick succession with controlled parallelism (5 concurrent requests).**

This aligns with BAMS guidance that requests can be sent without waiting for previous responses, while avoiding excessive concurrency that degrades performance.

---

## Error Handling Model

### Error Types

**1. Request-level errors**
- Invalid payload
- Authentication failure
- Returned immediately

**2. Permit-level errors**
- Apply to all prescribed events
- Stored once

**3. Event-level errors**
- Validation errors
- Duplicate warnings
- Stored per event

Error mapping logic is implemented in:
- `MapApiResponseToResult`
- `SavePrescribedEventSubmissionResults`

---

## Database & Auditing

Each submission is fully auditable, including:
- Raw request JSON
- Raw response JSON
- Submission timestamps
- Permit and event-level outcomes

This supports:
- Regulatory compliance
- Operational troubleshooting
- Data reconciliation and replay if required

---

## Configuration

Key settings are defined in `App.config`:
- API endpoints
- JWT assertion
- Timeouts
- `MaxConcurrentSubmissions`
- Logging flags

**Concurrency can be tuned without code changes.**

