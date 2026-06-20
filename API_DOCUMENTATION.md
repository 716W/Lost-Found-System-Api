# LostAndFoundApp — API Documentation

> **Base URL:** `http://localhost:8080/api/v1`  
> **Version:** v1  
> **Framework:** ASP.NET Core 8 | **ORM:** Entity Framework Core | **DB:** SQL Server 2022

---

## 1. Architecture Overview

### 1.1 Clean Architecture Layers

The system is divided into three projects that mirror the Clean Architecture rings:

| Layer | Project | Responsibility |
|---|---|---|
| **Presentation** | `LostAndFound.Api` | HTTP controllers, DTOs, response envelopes, middleware |
| **Domain / Core** | `LostAndFound.Core` | Entities, Enums, Repository interfaces, Domain logic, CQRS commands |
| **Infrastructure** | `LostAndFound.Infrastructure` | EF Core DbContext, repository implementations, JWT provider, Firebase FCM, file service |

**Dependency rule:** `Api` → `Core` ← `Infrastructure`. The Core layer has zero references to ASP.NET or EF packages; it only uses pure .NET.

### 1.2 CQRS

The `VerifyMatchCommand` (admin approve/reject a match) is dispatched through **MediatR**. All other operations use the **Unit of Work / Repository** pattern directly. This hybrid keeps simple CRUD fast while isolating the domain-heavy matching decision behind a proper command handler.

### 1.3 Full Request Data Flow

```
Mobile / Admin Client
        │  HTTP Request
        ▼
  [ExceptionMiddleware]          ← global error handler
        │
  [Authentication / Authorization]
        │
  [Controller]                   ← validates, maps DTO → Domain
        │
  [IUnitOfWork]                  ← orchestrates repositories
        │
  [Repository Implementation]    ← EF Core queries
        │
  [SQL Server (via Docker)]
```

### 1.4 Background Matching Flow

When a new `ItemReport` is created via `POST /api/v1/reports`, a **fire-and-forget background task** is launched:

```
POST /api/v1/reports
        │  Report saved to DB
        │
        └─► Task.Run(async () =>
                IMatchingService.ProcessMatchesForReportAsync(reportId)
                        │
                        ├─ Compares new report against all opposing reports
                        │  (Lost vs Found) using a similarity score
                        │
                        ├─ If score ≥ threshold → Match.Create() persisted
                        │
                        └─ IPushNotificationService.SendPushNotificationAsync()
                                │  FCM payload: { matchId, type }
                                └─ Sent to the lost-item owner's FCM token
```

---

## 2. Authentication & Security

### 2.1 JWT Authentication Flow

```
1. Client sends POST /api/v1/auth/login  { email, password, fcmToken }
2. Server validates credentials via ASP.NET Identity (UserManager)
3. Server issues:
   - JWT Access Token  (expires per JwtOptions.ExpiryHours, default 2h)
   - Opaque Refresh Token (stored server-side, linked to user)
4. Client stores both tokens
5. All subsequent requests must include:
   Authorization: Bearer <accessToken>
6. When access token expires, client calls POST /api/v1/auth/refresh
7. Server validates refresh token, issues a new token pair
8. On logout, POST /api/v1/auth/logout revokes the refresh token server-side
```

The FCM token sent during login is persisted to the `User` record. This token is later used by `IMatchingService` to deliver push notifications when a match is found — it is **separate** from the JWT.

### 2.2 JWT Configuration (`JwtOptions`)

| Property | Description |
|---|---|
| `SecretKey` | HMAC-SHA256 signing secret |
| `Issuer` | Token issuer claim |
| `Audience` | Token audience claim |
| `ExpiryHours` | Access token lifetime (default: 2 hours) |

### 2.3 Role-Based Access Control

The system defines three roles in `AppRoles`:

| Role | Value | Permissions |
|---|---|---|
| `User` | `"User"` | Reports, Claims, Notifications, Feedback, Profile |
| `Admin` | `"Admin"` | All User permissions + Admin dashboards, Reports admin, Claims admin, Matches admin, Handovers, Categories, Locations, Departments, Feedback reply |
| `SuperAdmin` | `"SuperAdmin"` | All Admin permissions + **User Management** (exclusive) |

> **Note:** New registrations via `POST /api/v1/auth/register` are assigned the `User` role automatically. Only a SuperAdmin can elevate a user to `Admin` via `PATCH /api/v1/admin/users/{id}/role`.

### 2.4 Universal Response Envelope

Every API response is wrapped in a consistent envelope:

**Standard Response:**
```json
{
  "succeeded": true,
  "message": "Operation Completed Successfully.",
  "data": { },
  "errors": null
}
```

**Paged Response (extends above):**
```json
{
  "succeeded": true,
  "message": "Data fetched successfully.",
  "data": [ ],
  "errors": null,
  "pageNumber": 1,
  "pageSize": 10,
  "totalPages": 5,
  "totalRecords": 48
}
```

**Error Response:**
```json
{
  "succeeded": false,
  "message": "Operation Failed.",
  "data": null,
  "errors": ["Descriptive error message here."]
}
```
## 3. Core Modules & Domain Logic

### 3.1 Enum Reference Tables

#### `enReportType`
| Value | Int | Meaning |
|---|---|---|
| `Lost` | 1 | User lost an item |
| `Found` | 2 | User found an item |

#### `enStatusType` (Report lifecycle)
| Value | Int | Meaning |
|---|---|---|
| `Open` | 1 | Newly submitted, awaiting review |
| `UnderReview` | 2 | Admin is reviewing |
| `Matched` | 3 | A match has been proposed |
| `Approved` | 4 | Match confirmed |
| `Rejected` | 5 | Match rejected |
| `Returned` | 6 | Item physically returned to owner (Handover created) |
| `Canceled` | 7 | User or admin canceled the report |
| `Closed` | 8 | Case closed |

#### `enApprovalStatus` (Claim lifecycle)
| Value | Int | Meaning |
|---|---|---|
| `Pending` | 1 | Claim submitted, awaiting admin decision |
| `Approved` | 2 | Admin approved the claim — eligible for Handover |
| `Completed` | 3 | Handover physically completed |
| `Rejected` | 4 | Admin rejected the claim |
| `Cancelled` | 5 | User cancelled the claim |
| `Closed` | 6 | Case closed |

#### `enMatchStatus`
| Value | Int | Meaning |
|---|---|---|
| `Pending` | 1 | Auto-generated, awaiting user or admin review |
| `UnderReview` | 2 | Being evaluated |
| `Confirmed` | 3 | Accepted (by user) or Approved (by admin) |
| `Rejected` | 4 | Rejected with a reason |
| `Completed` | 5 | Fully resolved |

#### `enConditionType`
| Value | Int | Meaning |
|---|---|---|
| `New` | 1 | Brand new |
| `Good` | 2 | Good condition |
| `Used` | 3 | Visibly used |
| `Damaged` | 4 | Partially damaged |
| `Broken` | 5 | Broken / non-functional |

#### `enIdType` (Handover ID verification)
| Value | Int | Meaning |
|---|---|---|
| `NationalId` | 1 | National ID card |
| `Passport` | 2 | Passport |

#### `enLocationType`
| Value | Int | Meaning |
|---|---|---|
| (defined in enum) | — | University building zones |

---

### 3.2 Matching Algorithm

The `IMatchingService.ProcessMatchesForReportAsync(reportId)` is invoked as a background `Task.Run` immediately after a new report is saved. The algorithm:

1. **Determines polarity** — If the new report is `Lost`, it searches all `Found` reports; if `Found`, it searches all `Lost` reports.
2. **Computes a similarity score** — Each candidate is scored across multiple dimensions (item name text similarity, category match, location proximity, condition type, color, date range). Scores are weighted and summed to produce a `double` between 0 and 1.
3. **Applies a threshold** — Only candidates whose score meets or exceeds the configured threshold produce a `Match` record.
4. **Persists the Match** — `Match.Create(lostId, foundId, matchScore, matchedBy: null)` is called. The match enters `enMatchStatus.Pending`.
5. **Sends FCM notification** — The lost-item owner's FCM device token is retrieved from their `User` record. A push notification is sent via `IPushNotificationService` with the `matchId` in the data payload so the mobile app can navigate directly to the match detail screen.

#### Match Domain Methods (Rich Domain Model)

| Method | Caller | Guard | Effect |
|---|---|---|---|
| `Match.Accept(userId)` | Lost-item owner | Status must be `Pending` | Sets status → `Confirmed` |
| `Match.Reject(userId, reason)` | Lost-item owner | Status must be `Pending`; reason non-empty | Sets status → `Rejected` |
| `Match.Approve(adminId)` | Admin/SuperAdmin | Status must be `Pending` | Sets status → `Confirmed` |

---

### 3.3 Handover Business Rules

The `HandoverRepository.CreateHandoverAsync` enforces strict domain invariants before persisting:

1. `LocationId`, `ReceiverUserId`, and `ClaimId` must all be valid (> 0).
2. `IdNumber` must not be blank.
3. The referenced `Claim` must exist.
4. The claim's `ApprovalStatus` must be **`Approved` (2)** — not Pending or any other status.
5. The `ReceiverUserId` on the handover must match the `claim.UserId` (the person who filed the claim).
6. **No duplicate** — Only one handover per claim is allowed.
7. On success:  
   - `Claim.ApprovalStatus` → `Completed`  
   - `Claim.Report.StatusType` → `Returned`

---

### 3.4 Push Notification Flow (FCM)

```
IMatchingService finds a match
        │
        ├─ Loads lost-item owner's User.FcmToken from DB
        │
        └─ IPushNotificationService.SendPushNotificationAsync(
               deviceToken: user.FcmToken,
               title:       "Item Match Found!",
               body:        "We found a potential match for your lost item.",
               data: {
                   "matchId": "42",
                   "type":    "match"
               }
           )
```

The **`data` payload** (not the notification payload) is used so the mobile app receives it even when in the background and can silently navigate to the correct screen without requiring user interaction with the OS notification banner.

**FCM Token Lifecycle:**
- Token is sent by the mobile client at `POST /api/v1/auth/login` in the `fcmToken` field.
- Stored on the `User` record in the database.
- Updated on every subsequent login (tokens can rotate).
- No token = no push notification (match is still saved, no crash).
## 4. API Endpoints Reference

---

### 4.1 Authentication — `/api/v1/auth`

---

#### `POST /api/v1/auth/login`
**Description:** Authenticates a user and returns a JWT access token + refresh token. Accepts an optional FCM device token to enable push notifications.  
**Auth:** None (public)

**Request Body:**
```json
{
  "email":    "user@example.com",
  "password": "P@ssw0rd!",
  "fcmToken": "fJk3mX9..."
}
```

| Field | Type | Required |
|---|---|---|
| `email` | string | ✅ |
| `password` | string | ✅ |
| `fcmToken` | string | ❌ |

**200 OK:**
```json
{
  "succeeded": true,
  "message": "Login successful.",
  "data": {
    "token": "eyJhbGci...",
    "refreshToken": "d4f8c1a2..."
  }
}
```
**401 Unauthorized:**
```json
{
  "succeeded": false,
  "message": "Operation Failed.",
  "errors": ["Invalid email or password."]
}
```

---

#### `POST /api/v1/auth/register`
**Description:** Registers a new user account. The new user is automatically assigned the `User` role.  
**Auth:** None (public)

**Request Body:**
```json
{
  "name":     "Ali Hassan",
  "email":    "ali@example.com",
  "password": "P@ssw0rd!"
}
```

**201 Created:**
```json
{ "succeeded": true, "message": "Account created successfully.", "data": null }
```
**400 Bad Request:**
```json
{ "succeeded": false, "errors": ["Passwords must have at least one non alphanumeric character."] }
```

---

#### `POST /api/v1/auth/refresh`
**Description:** Exchanges a valid refresh token for a new JWT + refresh token pair.  
**Auth:** None (public)

**Request Body:**
```json
{ "token": "eyJhbGci...", "refreshToken": "d4f8c1a2..." }
```

**200 OK:** Same shape as Login 200.  
**401 Unauthorized:** `"Invalid or expired refresh token."`

---

#### `PUT /api/v1/auth/change-password`
**Description:** Changes the authenticated user's password.  
**Auth:** `Bearer <token>` (any authenticated user)

**Request Body:**
```json
{ "currentPassword": "OldPass1!", "newPassword": "NewPass2@" }
```

**200 OK:** `{ "succeeded": true, "message": "Password changed successfully." }`  
**400 Bad Request:** `{ "errors": ["Incorrect password."] }`

---

#### `POST /api/v1/auth/logout`
**Description:** Revokes the current refresh token and logs the user out.  
**Auth:** `Bearer <token>`

**200 OK:** `{ "succeeded": true, "message": "Logged out successfully." }`

---

### 4.2 Profile — `/api/v1/profile`

---

#### `GET /api/v1/profile/me`
**Description:** Returns the authenticated user's profile information.  
**Auth:** `Bearer <token>`

**200 OK:**
```json
{
  "succeeded": true,
  "data": {
    "name":      "Ali Hassan",
    "email":     "ali@example.com",
    "avatarUrl": "/uploads/ProfileImages/ali.jpg"
  }
}
```

---

#### `PUT /api/v1/profile/me`
**Description:** Updates the authenticated user's name, email, and/or profile picture. Uses `multipart/form-data`.  
**Auth:** `Bearer <token>`  
**Content-Type:** `multipart/form-data`

| Field | Type | Required |
|---|---|---|
| `name` | string | ✅ |
| `email` | string | ✅ |
| `profileImage` | file | ❌ |

**200 OK:** Returns updated `ProfileResponseDto`.  
**400 Bad Request:** Validation errors from Identity.  
**404 Not Found:** `"User not found."`

---

### 4.3 Reports — `/api/v1/reports`

---

#### `GET /api/v1/reports` *(Public)*
**Description:** Returns a paginated, filterable list of all active reports. Available to anonymous users.  
**Auth:** None

**Query Parameters:**

| Param | Type | Description |
|---|---|---|
| `search` | string | Keyword search on item name |
| `categoryId` | int | Filter by category |
| `locationId` | int | Filter by location |
| `reportType` | int | `1` = Lost, `2` = Found |
| `pageNumber` | int | Default: 1 |
| `pageSize` | int | Default: 10 |

**200 OK (Paged):**
```json
{
  "succeeded": true,
  "data": [
    {
      "id": 12,
      "itemName": "Black Wallet",
      "imagePath": "/uploads/Reports/wallet.jpg",
      "reportType": "Lost",
      "status": "Open",
      "locationName": "Main Library",
      "reporterName": "Ali Hassan",
      "dateReported": "2026-06-18T10:00:00Z"
    }
  ],
  "pageNumber": 1,
  "pageSize": 10,
  "totalPages": 3,
  "totalRecords": 28
}
```

---

#### `GET /api/v1/reports/{id}`
**Description:** Returns full details of a single report.  
**Auth:** `Bearer <token>`

**200 OK:**
```json
{
  "succeeded": true,
  "data": {
    "id": 12,
    "itemName": "Black Wallet",
    "images": [
      { "id": 1, "path": "/uploads/Reports/wallet.jpg" }
    ],
    "reportType": "Lost",
    "locationName": "Main Library",
    "dateReported": "2026-06-18T10:00:00Z",
    "description": "Black leather wallet with student ID inside."
  }
}
```
**404 Not Found:** `"Report not found."`

---

#### `GET /api/v1/reports/me`
**Description:** Returns all reports submitted by the authenticated user.  
**Auth:** `Bearer <token>`

**200 OK:**
```json
{
  "succeeded": true,
  "data": [
    {
      "id": 12,
      "itemName": "Black Wallet",
      "imagePath": "/uploads/Reports/wallet.jpg",
      "dateReported": "2026-06-18T10:00:00Z",
      "reportType": 1,
      "statusType": 1
    }
  ]
}
```

---

#### `POST /api/v1/reports`
**Description:** Creates a new Lost or Found item report. Triggers the background matching algorithm on success.  
**Auth:** `Bearer <token>`  
**Content-Type:** `multipart/form-data`

| Field | Type | Required | Notes |
|---|---|---|---|
| `itemName` | string | ✅ | |
| `reportType` | int | ✅ | 1=Lost, 2=Found |
| `conditionType` | int | ✅ | 1–5 |
| `locationId` | int | ✅ | Must exist |
| `categoryId` | int | ✅ | Must exist |
| `dateReported` | datetime | ✅ | |
| `color` | string | ❌ | |
| `description` | string | ❌ | |
| `images` | file[] | ❌ | Max 5 files |

**201 Created:**
```json
{ "succeeded": true, "message": "Report created successfully.", "data": { "id": 13 } }
```
**400 Bad Request:**
```json
{ "succeeded": false, "errors": ["Max 5 images allowed"] }
```

---

#### `PUT /api/v1/reports/{id}`
**Description:** Updates an existing report. Only the report owner or an Admin/SuperAdmin may update. Uses `multipart/form-data`. All fields are optional — only provided fields are updated.  
**Auth:** `Bearer <token>` (owner) or Admin/SuperAdmin

| Field | Type | Notes |
|---|---|---|
| `itemName` | string | |
| `reportType` | int | |
| `conditionType` | int | |
| `locationId` | int | |
| `categoryId` | int | |
| `dateReported` | datetime | |
| `color` | string | |
| `description` | string | |
| `deletedImageIds` | int[] | IDs of images to remove |
| `newImages` | file[] | New images to add |

**200 OK:** `"Report updated with images successfully."`  
**403 Forbidden:** `"You do not have permission to edit this report."`  
**404 Not Found:** `"Report not found."`

---

#### `PUT /api/v1/reports/{id}/cancel`
**Description:** Cancels the user's own report.  
**Auth:** `Bearer <token>` (owner only)

**200 OK:** `"Report canceled successfully."`  
**400 Bad Request:** `"Failed to cancel report."`

---

#### `GET /api/v1/admin/reports` *(Admin)*
**Description:** Admin view of all reports with extended filtering including status and date range.  
**Auth:** Admin | SuperAdmin

**Additional Query Params:** `statusType` (int), `fromDate` (datetime), `toDate` (datetime)

**200 OK (Paged):**
```json
{
  "data": [
    {
      "id": 12, "itemName": "Black Wallet", "color": "Black",
      "conditionType": 2, "statusType": 1, "reportType": 1,
      "dateReported": "2026-06-18T10:00:00Z",
      "categoryName": "Accessories", "locationName": "Main Library",
      "userName": "Ali Hassan"
    }
  ]
}
```

---

#### `GET /api/v1/admin/reports/{id}` *(Admin)*
**Description:** Admin detail view of a single report.  
**Auth:** Admin | SuperAdmin

---

#### `DELETE /api/v1/admin/reports/{id}` *(Admin)*
**Description:** Permanently deletes a report along with all its physical image files. Wrapped in a DB transaction.  
**Auth:** Admin | SuperAdmin

**200 OK:** `"Report and images deleted successfully."`  
**500 Internal Server Error:** `"Failed to delete report."` (transaction rolled back)

---

#### `PUT /api/v1/admin/reports/{id}/status` *(Admin)*
**Description:** Changes the status of any report.  
**Auth:** Admin | SuperAdmin

**Request Body:**
```json
{ "statusType": 3 }
```
**200 OK:** `"Report status changed successfully."`

---

#### `PUT /api/v1/admin/reports/{id}/type` *(Admin)*
**Description:** Corrects the type of a report (Lost ↔ Found).  
**Auth:** Admin | SuperAdmin

**Request Body:**
```json
{ "reportType": 2 }
```
**200 OK:** `"Report type updated successfully."`
---

### 4.4 Matches — `/api/v1/matches`

---

#### `GET /api/v1/admin/matches` *(Admin)*
**Description:** Paginated list of all matches across all statuses.  
**Auth:** Admin | SuperAdmin

**Query Params:** `pageNumber` (default 1), `pageSize` (default 10)

**200 OK (Paged):**
```json
{
  "data": [
    {
      "id": 5,
      "matchScore": 0.87,
      "status": 1,
      "matchDate": "2026-06-19T08:00:00Z",
      "createdAt": "2026-06-19T08:00:00Z",
      "updatedAt": "2026-06-19T08:00:00Z",
      "rejectionReason": null,
      "reviewedBy": null,
      "reviewedAt": null,
      "lostId": 10,
      "foundId": 12
    }
  ]
}
```

---

#### `GET /api/v1/admin/matches/pending` *(Admin)*
**Description:** Paginated list of matches with `Status = Pending (1)` only.  
**Auth:** Admin | SuperAdmin

**200 OK:** Same shape as above, filtered to pending items.

---

#### `GET /api/v1/admin/matches/{matchId}` *(Admin)*
**Description:** Returns a single match by ID. No ownership check — admin may view any match.  
**Auth:** Admin | SuperAdmin

**404 Not Found:** `"Match not found."`

---

#### `PUT /api/v1/admin/matches/{matchId}/verify` *(Admin)*
**Description:** Approves or rejects a pending match. `RejectionReason` is **required** when `isApproved` is `false`.  
**Auth:** Admin | SuperAdmin

**Request Body:**
```json
{
  "isApproved": false,
  "rejectionReason": "Item color does not match the lost report."
}
```

**200 OK:**
```json
{ "succeeded": true, "message": "Match rejected successfully.", "data": true }
```
**400 Bad Request:** `"Rejection reason is required when rejecting a match."` or `"Failed to verify the match. It may not exist or is not in a Pending state."`

---

#### `GET /api/v1/matches/{matchId}` *(User — IDOR Protected)*
**Description:** Returns a match by ID. **Only the owner of the associated lost item may call this endpoint.** The mobile app uses this after receiving an FCM push notification containing the `matchId`.  
**Auth:** `Bearer <token>`

**403 Forbidden:** `"Access denied."` (if the caller does not own the lost item)  
**404 Not Found:** `"Match not found."`

---

#### `POST /api/v1/matches/{matchId}/accept` *(User — IDOR Protected)*
**Description:** The lost-item owner accepts a pending match, confirming the found item is theirs.  
**Auth:** `Bearer <token>` (lost-item owner only)

**200 OK:** `{ "succeeded": true, "message": "Match accepted successfully.", "data": true }`  
**400 Bad Request:** `"Match is not in a Pending state and cannot be accepted."`  
**403 Forbidden:** `"Access denied."`

---

#### `POST /api/v1/matches/{matchId}/reject` *(User — IDOR Protected)*
**Description:** The lost-item owner rejects a pending match with a required reason.  
**Auth:** `Bearer <token>` (lost-item owner only)

**Request Body:**
```json
{ "reason": "This is not my wallet." }
```

**200 OK:** `{ "succeeded": true, "message": "Match rejected successfully.", "data": true }`  
**400 Bad Request:** `"A rejection reason is required."`

---

### 4.5 Claims — `/api/v1/claims`

---

#### `GET /api/v1/admin/claims` *(Admin)*
**Description:** Paginated, filterable list of all claims. Each claim includes its associated match score.  
**Auth:** Admin | SuperAdmin

**Query Params:**

| Param | Type | Description |
|---|---|---|
| `search` | string | Filter by item name or claimant |
| `approvalStatus` | int | 1–6 (see `enApprovalStatus`) |
| `fromDate` | datetime | Start of claim date range |
| `toDate` | datetime | End of claim date range |
| `pageNumber` | int | Default: 1 |
| `pageSize` | int | Default: 10 |

**200 OK (Paged):**
```json
{
  "data": [
    {
      "id": 7,
      "claimCode": "CLM-007",
      "itemName": "Black Wallet",
      "claimantName": "Sara Ahmed",
      "claimDate": "2026-06-19T09:00:00Z",
      "matchScore": 0.91,
      "approvalStatus": 1
    }
  ]
}
```

---

#### `GET /api/v1/admin/claims/{id}` *(Admin)*
**Description:** Full detail view of a claim for admin review, including item images and claimant contact.  
**Auth:** Admin | SuperAdmin

**200 OK:**
```json
{
  "data": {
    "id": 7,
    "claimCode": "CLM-007",
    "claimDate": "2026-06-19T09:00:00Z",
    "approvalStatus": 1,
    "remarks": "",
    "itemName": "Black Wallet",
    "description": "Found near library entrance.",
    "locationName": "Main Library",
    "dateReported": "2026-06-18T10:00:00Z",
    "claimantName": "Sara Ahmed",
    "claimantEmail": "sara@example.com",
    "matchScore": 0.91,
    "itemImages": ["/uploads/Reports/wallet.jpg"]
  }
}
```

---

#### `PUT /api/v1/admin/claims/{id}/approve` *(Admin)*
**Description:** Approves a claim, making it eligible for a physical handover.  
**Auth:** Admin | SuperAdmin

**200 OK:** `"Claim approved successfully."`  
**400 Bad Request:** `"Failed to approve claim."`

---

#### `PUT /api/v1/admin/claims/{id}/reject` *(Admin)*
**Description:** Rejects a claim with mandatory remarks.  
**Auth:** Admin | SuperAdmin

**Request Body:**
```json
{ "remarks": "The claim does not match the found item description." }
```
**200 OK:** `"Claim rejected successfully."`  
**400 Bad Request:** `"Remarks are required."`

---

#### `POST /api/v1/claims`
**Description:** Submits a new claim on a found-item report.  
**Auth:** `Bearer <token>`

**Request Body:**
```json
{ "reportId": 12 }
```
**200 OK:** `"Claim submitted successfully."`  
**400 Bad Request:** `"Failed to create claim."` (e.g., duplicate claim, report not found)

---

#### `GET /api/v1/claims/me`
**Description:** Returns all claims submitted by the authenticated user.  
**Auth:** `Bearer <token>`

**200 OK:**
```json
{
  "data": [
    {
      "id": 7,
      "claimCode": "CLM-007",
      "reportId": 12,
      "itemName": "Black Wallet",
      "claimDate": "2026-06-19T09:00:00Z",
      "approvalStatus": 1,
      "imagePath": "/uploads/Reports/wallet.jpg"
    }
  ]
}
```

---

#### `PUT /api/v1/claims/{id}/cancel`
**Description:** Cancels a pending claim made by the authenticated user.  
**Auth:** `Bearer <token>` (claimant only)

**200 OK:** `"Claim canceled successfully."`

---

### 4.6 Handovers — `/api/v1/admin/handovers`

All handover endpoints require **Admin or SuperAdmin** role.

---

#### `POST /api/v1/admin/handovers`
**Description:** Records the physical handover of a found item to its owner. The linked claim must be in `Approved` status. On success, claim status → `Completed` and report status → `Returned`. This action is audit-logged.  
**Auth:** Admin | SuperAdmin  
**Content-Type:** `multipart/form-data`

| Field | Type | Required | Notes |
|---|---|---|---|
| `claimId` | int | ✅ | Must be an approved claim |
| `idType` | int | ✅ | 1=NationalId, 2=Passport |
| `idNumber` | string | ✅ | Receiver's ID number |
| `handoverDate` | datetime | ❌ | Defaults to UTC now |
| `locationId` | int | ❌ | Defaults to report's location |
| `receiverUserId` | int | ❌ | Defaults to claim's user |
| `notes` | string | ❌ | |
| `idPhoto` | file | ❌ | Photo of receiver's ID |
| `signatureImage` | file | ❌ | Digital signature image |

**200 OK:** `"Handover created successfully."`  
**400 Bad Request:** `"Failed to create handover."` (e.g., claim not approved, duplicate handover)

---

#### `GET /api/v1/admin/handovers/{id}`
**Description:** Returns handover details by handover ID.  
**Auth:** Admin | SuperAdmin

**200 OK:**
```json
{
  "data": {
    "id": 3,
    "idType": 1,
    "idNumber": "123456789",
    "imagePath": "/uploads/handovers/ids/id.jpg",
    "signaturePath": "/uploads/handovers/signatures/sig.png",
    "handoverDate": "2026-06-20T12:00:00Z",
    "notes": "Handed over at the admin office.",
    "locationName": "Admin Office",
    "receiverName": "Sara Ahmed",
    "handedByName": "Admin User",
    "claimId": 7
  }
}
```

---

#### `GET /api/v1/admin/handovers/claim/{claimId}`
**Description:** Returns the handover record associated with a specific claim ID.  
**Auth:** Admin | SuperAdmin

**404 Not Found:** `"Handover not found."`

---

### 4.7 Notifications — `/api/v1/notifications`

---

#### `GET /api/v1/notifications/me`
**Description:** Returns all in-app notifications for the authenticated user. Notifications are created automatically by the system when a match is found.  
**Auth:** `Bearer <token>`

**200 OK:**
```json
{
  "data": [
    {
      "id": 1,
      "title": "Match Found!",
      "message": "A potential match has been found for your lost item.",
      "isRead": false,
      "createdAt": "2026-06-19T08:05:00Z"
    }
  ]
}
```

---

#### `PUT /api/v1/notifications/{id}/read`
**Description:** Marks a notification as read. Only the owning user can perform this action.  
**Auth:** `Bearer <token>`

**200 OK:** `{ "succeeded": true, "message": "Notification marked as read successfully.", "data": true }`  
**404 Not Found:** `"Notification not found or you don't have permission to mark it as read."`

---

### 4.8 Feedback — `/api/v1/feedbacks`

---

#### `POST /api/v1/feedbacks`
**Description:** Submits new feedback from the authenticated user.  
**Auth:** `Bearer <token>`

**Request Body:**
```json
{
  "subject": "Great service!",
  "message": "I found my wallet within 2 days.",
  "rating": 5
}
```
**201 Created:** Returns the created `FeedbackResponseDto`.

---

#### `GET /api/v1/feedbacks/me`
**Description:** Returns all feedback submitted by the authenticated user, including any admin replies.  
**Auth:** `Bearer <token>`

**200 OK:**
```json
{
  "data": [
    {
      "id": 3,
      "subject": "Great service!",
      "message": "I found my wallet within 2 days.",
      "rating": 5,
      "isReplied": true,
      "adminReply": "Thank you for your kind words!",
      "userId": 4,
      "userName": "Ali Hassan",
      "userEmail": "ali@example.com",
      "createdAt": "2026-06-18T10:00:00Z"
    }
  ]
}
```

---

#### `GET /api/v1/admin/feedbacks` *(Admin)*
**Description:** Returns all system feedback. Optionally filter to show only unanswered entries.  
**Auth:** Admin | SuperAdmin

**Query Params:** `pendingOnly` (bool, default: false)

---

#### `POST /api/v1/admin/feedbacks/{id}/reply` *(Admin)*
**Description:** Adds an admin reply to a feedback entry and marks it as replied.  
**Auth:** Admin | SuperAdmin

**Request Body:**
```json
{ "adminReply": "Thank you for your kind words!" }
```
**200 OK:** `{ "message": "Reply added successfully." }`  
**404 Not Found:** `"Feedback not found."`

---

### 4.9 Reference Data (Categories, Locations, Departments, Universities)

All reference data follows the same CRUD pattern. Public `GET` endpoints require authentication; mutating endpoints (POST/PUT/DELETE) require **Admin or SuperAdmin**. Mutations are audit-logged.

#### Categories

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/v1/categories` | Any authenticated | List all categories |
| GET | `/api/v1/categories/{id}` | Any authenticated | Get category by ID |
| POST | `/api/v1/admin/categories` | Admin/SuperAdmin | Create category |
| PUT | `/api/v1/admin/categories/{id}` | Admin/SuperAdmin | Update category |
| DELETE | `/api/v1/admin/categories/{id}` | Admin/SuperAdmin | Delete (blocked if has reports) |

**Category Body:**
```json
{ "name": "Electronics" }
```
**Category Response:**
```json
{ "id": 1, "name": "Electronics" }
```

#### Locations

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/v1/locations` | Any authenticated | List all locations |
| GET | `/api/v1/locations/{id}` | Any authenticated | Get location by ID |
| POST | `/api/v1/admin/locations` | Admin/SuperAdmin | Create location |
| PUT | `/api/v1/admin/locations/{id}` | Admin/SuperAdmin | Update location |
| DELETE | `/api/v1/admin/locations/{id}` | Admin/SuperAdmin | Delete (blocked if has reports or handovers) |

**Location Body:**
```json
{ "name": "Main Library", "locationType": 1, "departmentId": 2 }
```
**Location Response:**
```json
{ "id": 3, "name": "Main Library", "locationType": 1, "departmentId": 2, "departmentName": "Science" }
```

#### Departments & Universities

Same pattern as Categories: public GET, admin-only POST/PUT/DELETE, audit-logged on mutations. Linked to Locations via foreign key.

---

### 4.10 Admin — User Management

**All endpoints require `SuperAdmin` role exclusively.**

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/v1/admin/users` | Paginated user list with reports/claims counts |
| GET | `/api/v1/admin/users/{id}` | Single user detail |
| POST | `/api/v1/admin/users` | Create a user account |
| PUT | `/api/v1/admin/users/{id}` | Update user name |
| DELETE | `/api/v1/admin/users/{id}` | Permanently delete user |
| PATCH | `/api/v1/admin/users/{id}/block` | Toggle active/blocked status |
| PATCH | `/api/v1/admin/users/{id}/role` | Change role (`User` or `Admin` only) |

**User Response:**
```json
{
  "id": 4,
  "name": "Ali Hassan",
  "email": "ali@example.com",
  "isActive": true,
  "created": "2026-06-01T00:00:00Z",
  "roles": ["User"],
  "reportsCount": 3,
  "claimsCount": 1
}
```

**Create User Body:**
```json
{ "name": "New Admin", "email": "admin@example.com", "password": "P@ssw0rd!" }
```

**Change Role Body:**
```json
{ "role": "Admin" }
```
> ⚠️ Role must be exactly `"User"` or `"Admin"`. Setting `"SuperAdmin"` is rejected.

---

### 4.11 Admin — Dashboard & Audit Logs

---

#### `GET /api/v1/admin/dashboard/stats`
**Description:** Returns aggregate statistics for the admin dashboard.  
**Auth:** Admin | SuperAdmin

**200 OK:**
```json
{
  "succeeded": true,
  "data": {
    "totalReports": 128,
    "totalLostReports": 74,
    "totalFoundReports": 54,
    "totalMatches": 32,
    "totalClaims": 21,
    "pendingClaims": 5,
    "totalUsers": 89,
    "recentReports": [ ]
  }
}
```

---

#### `GET /api/v1/admin/audit-logs`
**Description:** Paginated list of all admin actions recorded by the `[AuditLog]` attribute filter.  
**Auth:** Admin | SuperAdmin

**Query Params:** `searchTerm` (string), `page` (int), `pageSize` (int)

**200 OK (Paged):**
```json
{
  "data": [
    {
      "id": 1,
      "timestamp": "2026-06-20T10:00:00Z",
      "adminName": "Super Admin",
      "action": "Created New Handover",
      "target": "Handover",
      "ipAddress": "192.168.1.1"
    }
  ]
}
```

---

## 5. External Integrations

### 5.1 Firebase Cloud Messaging (FCM)

**Integration Point:** `LostAndFound.Infrastructure` — implements `IPushNotificationService` using the Firebase Admin SDK (`FirebaseAdmin` NuGet package). A test endpoint is available at `GET /test-firebase` to verify initialization.

#### 5.1.1 FCM Initialization

The Firebase app is initialized from a service account credentials file configured in `appsettings.json`. The `GET /test-firebase` route returns `200 OK` with `"Firebase initialized successfully!"` if the SDK is loaded, or a 500 Problem response if not.

#### 5.1.2 Push Notification Payload Contract

When a match is found by the background algorithm, the following FCM message is dispatched:

```
FCM Message Structure:
├── notification:
│   ├── title: "Item Match Found!"
│   └── body:  "We found a potential match for your lost item."
└── data:                          ← key-value string dictionary
    ├── "matchId": "42"            ← ID of the created Match record
    └── "type":    "match"         ← discriminator for the mobile app router
```

> The notification is sent via the **data-only payload** approach. The `data` dictionary keys are always `string → string`.

#### 5.1.3 Mobile App Handling Contract

| Key | Type | Usage |
|---|---|---|
| `matchId` | string (parse to int) | Call `GET /api/v1/matches/{matchId}` to fetch match details |
| `type` | string | Route discriminator; `"match"` → navigate to Match Detail screen |

**Expected mobile flow:**
1. App receives FCM data payload while in background/foreground.
2. App reads `data["type"]` — if `"match"`, navigate to Match Detail.
3. App reads `data["matchId"]`, calls `GET /api/v1/matches/{matchId}` with the stored JWT.
4. App displays match information and presents Accept / Reject buttons.
5. User action calls `POST /api/v1/matches/{matchId}/accept` or `/reject`.

#### 5.1.4 FCM Token Management

| Event | Action |
|---|---|
| User logs in with `fcmToken` | Token saved/updated on `User` record |
| User logs in without `fcmToken` | Existing token preserved; no push notification will be sent |
| Token becomes stale | Recommended: re-send `fcmToken` on every app launch |

---

## 6. Deployment

### 6.1 Docker Compose Services

```yaml
services:
  db:        # SQL Server 2022 — port 2020:1433
  migration: # Runs EF Core migrations before API starts
  api:       # ASP.NET Core app — port 8080:8080
```

**Start order:** `db` (health-checked) → `migration` (completes successfully) → `api`

**Volumes:**
- `sql_data` — SQL Server data persistence
- `uploaded_files` — Maps to `/app/wwwroot` for user-uploaded images (report photos, profile pictures, handover ID images, signatures)

### 6.2 Static File URLs

Uploaded files are served as static files via `app.UseStaticFiles()`. A file stored at path `/uploads/Reports/wallet.jpg` is accessible at:

```
http://your-host:8080/uploads/Reports/wallet.jpg
```

File categories and their storage paths:

| Category | Storage Path |
|---|---|
| Report images | `wwwroot/uploads/Reports/` |
| Profile pictures | `wwwroot/uploads/ProfileImages/` |
| Handover ID photos | `wwwroot/uploads/handovers/ids/` |
| Handover signatures | `wwwroot/uploads/handovers/signatures/` |

### 6.3 CORS Configuration

The API allows cross-origin requests from:
- `http://localhost:8081` (local React/Expo dev)
- `http://192.168.1.107:8081` (LAN mobile dev)

Credentials are allowed (required for cookie-based auth if used in future).
